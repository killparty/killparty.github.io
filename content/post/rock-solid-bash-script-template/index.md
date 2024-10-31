---
title: Bash Boilerplate
description: Rock-solid bash script template for easy scripting
date: 2024-10-23 17:02:00+0000
toc: false
---

Writing reliable Bash scripts can be tricky, but having a solid starting point helps. Below is a Bash boilerplate that I often use to make scripts safer and easier to manage.

It includes strict error handling (`set -euo pipefail`) to catch mistakes early, log rotation to keep log files from getting too big, and a lockfile to prevent multiple script instances from running at the same time. There’s also a section for checking dependencies and handling cleanups.

### Customization:

- Modify the `main` function to include the actual logic of your script.
- Add or remove dependencies as needed in the `check_dependencies` function.
- Adjust paths for the `LOCKFILE` and `LOGFILE` as per your requirements.
- Remove the root check in case it is not required.

This template ensures your script adheres to good security and reliability standards, minimizing the risk of common pitfalls in bash scripting.

```sh=
#!/bin/bash

# Strict mode - makes the script safer and more reliable
set -euo pipefail
IFS=$'\n\t'

# Variables
LOCKFILE="/tmp/$(basename "$0").lock"
LOGFILE="/var/log/$(basename "$0").log"
PID=$$
SCRIPT_NAME="$(basename "$0")"
START_TIME=$(date '+%Y-%m-%d %H:%M:%S')
MAX_LOG_SIZE=1048576  # Max log size in bytes (1MB = 1048576 bytes)
LOG_BACKUP_COUNT=5     # Number of rotated logs to keep

# Logging function
log() {
    local level="$1"
    local message="$2"
    echo "$START_TIME [$level] : $message" | tee -a "$LOGFILE"
    rotate_log_if_needed
}

# Log rotation function
rotate_log_if_needed() {
    if [[ -f "$LOGFILE" ]]; then
        local log_size
        log_size=$(stat -c%s "$LOGFILE")

        if (( log_size > MAX_LOG_SIZE )); then
            echo "$START_TIME [INFO] : Log file exceeded $MAX_LOG_SIZE bytes, rotating logs." >> "$LOGFILE"
            
            for (( i=LOG_BACKUP_COUNT-1; i>0; i-- )); do
                if [[ -f "$LOGFILE.$i" ]]; then
                    mv "$LOGFILE.$i" "$LOGFILE.$((i+1))"
                fi
            done

            mv "$LOGFILE" "$LOGFILE.1"
            : > "$LOGFILE"  # Truncate the log file
        fi
    fi
}

# Clean up function
cleanup() {
    if [[ -f "$LOCKFILE" ]]; then
        rm -f "$LOCKFILE"
        log "INFO" "Lock file removed"
    fi
}

# Lockfile mechanism to avoid multiple instances
create_lockfile() {
    if [[ -f "$LOCKFILE" ]]; then
        log "ERROR" "Lockfile exists: $LOCKFILE. Another instance is running."
        exit 1
    fi
    echo "$PID" > "$LOCKFILE"
    log "INFO" "Created lockfile: $LOCKFILE"
}

# Trap signals to ensure cleanup
trap 'cleanup; exit' INT TERM EXIT

# Root check (if needed)
if [[ "$EUID" -ne 0 ]]; then
   log "ERROR" "This script must be run as root"
   exit 1
fi

# Secure PATH (avoids executing malicious binaries in an altered PATH)
export PATH='/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'

# Function to check required commands (dependencies)
check_dependencies() {
    local dependencies=("command1" "command2") # Replace with actual commands
    for cmd in "${dependencies[@]}"; do
        if ! command -v "$cmd" &> /dev/null; then
            log "ERROR" "Dependency '$cmd' is not installed or not in PATH."
            exit 1
        fi
    done
}

# Function to handle errors
error_handler() {
    local exit_code=$?
    local line_no=$1
    log "ERROR" "Error on line $line_no: Exit code $exit_code"
    exit "$exit_code"
}
trap 'error_handler $LINENO' ERR

# Main script logic
main() {
    log "INFO" "Starting script: $SCRIPT_NAME"

    # Add your script logic here
    # ...

    log "INFO" "Script completed successfully"
}

# Run the main logic
create_lockfile
check_dependencies
main
cleanup
```

### Explanation and Best Practices:

1. **Strict Mode**:
    
    - `set -euo pipefail`:
        - `-e`: Exit immediately if a command exits with a non-zero status.
        - `-u`: Treat unset variables as an error.
        - `-o pipefail`: Ensures that if any part of a pipe fails, the entire pipeline fails.
    - `IFS=$'\n\t'`: Ensures that word splitting is done only on newlines and tabs, preventing security issues with spaces.
2. **Lockfile Mechanism**:
    
    - The script uses a lockfile (`/tmp/${SCRIPT_NAME}.lock`) to ensure only one instance runs at a time.
    - If the script is already running (lockfile exists), it exits to avoid multiple instances.
3. **Logging**:
    
    - `log()` function logs messages with a timestamp, log level (`INFO`, `ERROR`, etc.), and appends them to a log file (`/var/log/${SCRIPT_NAME}.log`).
    - Log entries are echoed to both the terminal and log file.
4. **Signal Trapping and Cleanup**:
    
    - `trap 'cleanup; exit' INT TERM EXIT`: This ensures the lockfile is removed if the script exits unexpectedly (e.g., via `Ctrl+C` or system signals).
    - Cleanup function removes the lockfile, avoiding stale lockfiles.
5. **Root Check**:
    
    - If the script requires root privileges, this check prevents it from running as a non-root user.
6. **Secure PATH**:
    
    - The script explicitly sets the `PATH` to a known secure set of directories, avoiding potential execution of malicious commands from user-controlled locations.
7. **Dependency Checks**:
    
    - The `check_dependencies` function ensures that all required commands (`command1`, `command2`, etc.) are available before the script proceeds. You can replace these with actual dependencies.
8. **Error Handling**:
    
    - The `error_handler` function is triggered on any error (via the `ERR` trap), providing useful information about where the error occurred and what the exit code was.

Feel free to use this template as a foundation for your own scripts and tweak it to fit your needs!
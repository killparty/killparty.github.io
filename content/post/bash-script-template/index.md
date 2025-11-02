---
title: Bash Boilerplate
description: Bash script template for easy scripting
date: 2024-10-23 17:02:00+0000
toc: false
---

```sh=
#!/bin/bash
#
# Bash Script Boilerplate
# ========================
# A production-ready bash script template with:
#   - Error handling and strict mode
#   - Logging (optional, disabled by default)
#   - Lockfile mechanism (prevents multiple instances)
#   - Automatic cleanup on exit
#   - Log rotation
#
# Quick Start:
#   1. Copy this file to your script name
#   2. Add your logic in the main() function
#   3. Uncomment and customize features as needed:
#      - Enable logging: change ENABLE_LOGGING to "true" below
#      - Add dependencies: uncomment and modify check_dependencies()
#      - Require root: uncomment the root check section
#
# Usage examples:
#   ./script.sh                          # Run with logging disabled
#   ENABLE_LOGGING=true ./script.sh      # Enable logging via env var
#   LOG_DIR=/var/log ./script.sh         # Enable logging to specific directory
#

# ============================================================================
# Configuration
# ============================================================================

# Secure PATH
export PATH='/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'

# Strict mode (exits on error, undefined variables, pipe failures)
set -euo pipefail
IFS=$'\n\t'

# Script metadata
SCRIPT_NAME="$(basename "$0")"
PID=$$

# Lockfile location (prevents multiple instances)
LOCKFILE="/tmp/${SCRIPT_NAME}.lock"

# ============================================================================
# Logging Configuration
# ============================================================================

# Enable logging: set to "true" to enable file logging, or keep "false" for console-only
# You can also enable via environment: ENABLE_LOGGING=true ./script.sh
ENABLE_LOGGING="${ENABLE_LOGGING:-false}"

# Log directory configuration
# - If LOG_DIR env var is set, logging is automatically enabled
# - Default: ~/.local/log (falls back to /tmp if HOME is not available)
if [[ -n "${LOG_DIR:-}" ]]; then
    ENABLE_LOGGING=true
elif [[ -n "${HOME:-}" ]] && [[ -d "${HOME:-}" ]]; then
    LOG_DIR="${HOME}/.local/log"
else
    LOG_DIR="/tmp"
fi

LOGFILE="${LOG_DIR}/${SCRIPT_NAME}.log"
MAX_LOG_SIZE=1048576    # Max log size: 1MB (1048576 bytes)
LOG_BACKUP_COUNT=4      # Number of rotated log files to keep

# ============================================================================
# Functions
# ============================================================================

# Ensure log directory exists
ensure_log_dir() {
    if [[ ! -d "$LOG_DIR" ]]; then
        mkdir -p "$LOG_DIR" 2>/dev/null || {
            echo "Error: Cannot create log directory: $LOG_DIR" >&2
            return 1
        }
    fi
}

# Logging function
# Usage: log "INFO" "Your message here"
#        log "ERROR" "Something went wrong"
#        log "WARN" "Warning message"
log() {
    local level="$1"
    local message="$2"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S' 2>/dev/null || echo "NO-DATE")
    
    # If logging is disabled, output to console only
    if [[ "${ENABLE_LOGGING}" != "true" ]]; then
        if [[ "$level" == "ERROR" ]]; then
            echo "$timestamp [$level] : $message" >&2
        else
            echo "$timestamp [$level] : $message"
        fi
        return 0
    fi
    
    # Logging is enabled, write to file (and console)
    if ensure_log_dir 2>/dev/null; then
        echo "$timestamp [$level] : $message" | tee -a "$LOGFILE" 2>/dev/null || {
            echo "$timestamp [$level] : $message" >&2
        }
        rotate_log_if_needed
    else
        # Fallback to stderr if log file is unavailable
        echo "$timestamp [$level] : $message" >&2
    fi
}

# Log rotation function (runs automatically when log file exceeds MAX_LOG_SIZE)
rotate_log_if_needed() {
    if [[ -f "$LOGFILE" ]]; then
        local log_size
        local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        log_size=$(wc -c < "$LOGFILE")

        if (( log_size > MAX_LOG_SIZE )); then
            echo "$timestamp [INFO] : Log file exceeded $MAX_LOG_SIZE bytes, rotating logs." >> "$LOGFILE"
            
            for (( i=LOG_BACKUP_COUNT-1; i>0; i-- )); do
                if [[ -f "$LOGFILE.$i" ]]; then
                    mv "$LOGFILE.$i" "$LOGFILE.$((i+1))"
                fi
            done

            mv "$LOGFILE" "$LOGFILE.1"
            : > "$LOGFILE"
        fi
    fi
}

# Error handler (called automatically on script errors)
error_handler() {
    local exit_code=$?
    local line_no=$1
    log "ERROR" "Script failed at line $line_no with exit code $exit_code." 2>/dev/null || true
    exit "$exit_code"
}

# Lockfile mechanism to prevent multiple instances
create_lockfile() {
    if [[ -f "$LOCKFILE" ]]; then
        local old_pid
        old_pid=$(cat "$LOCKFILE" 2>/dev/null || echo "")
        if [[ -n "$old_pid" ]] && kill -0 "$old_pid" 2>/dev/null; then
            log "ERROR" "Another instance is already running (PID: $old_pid). Lockfile: $LOCKFILE"
            exit 1
        else
            # Stale lockfile detected, removing it
            rm -f "$LOCKFILE"
            log "INFO" "Removed stale lockfile at $LOCKFILE."
        fi
    fi
    echo "$PID" > "$LOCKFILE"
    log "INFO" "Created lockfile at $LOCKFILE."
}

# Clean up function (called automatically on exit)
cleanup() {
    if [[ -z "${CLEANUP_DONE:-}" ]]; then
        export CLEANUP_DONE=1
        if [[ -f "$LOCKFILE" ]]; then
            rm -f "$LOCKFILE"
            log "INFO" "Lockfile removed."
        fi
    fi
}

# Check required commands (dependencies)
# Uncomment and modify as needed:
check_dependencies() {
    # Example: Check for required commands
    # local dependencies=("curl" "jq" "git")
    # for cmd in "${dependencies[@]}"; do
    #     if ! command -v "$cmd" &> /dev/null; then
    #         log "ERROR" "Required command '$cmd' not found in PATH."
    #         exit 1
    #     fi
    # done
    :
}

# Main script logic - ADD YOUR CODE HERE
main() {
    log "INFO" "Starting $SCRIPT_NAME"

    # ========================================
    # TODO: Add your script logic here
    # ========================================
    #
    # Examples:
    #   log "INFO" "Processing files..."
    #   if some_condition; then
    #       log "ERROR" "Something went wrong"
    #       exit 1
    #   fi
    #   log "INFO" "Done processing"
    #

    log "INFO" "Completed successfully."
}

# ============================================================================
# Optional: Root Check
# ============================================================================
# Uncomment the block below if your script requires root privileges

# if [[ "$EUID" -ne 0 ]]; then
#     log "ERROR" "This script must be run as root."
#     exit 1
# fi

# ============================================================================
# Signal handlers and execution
# ============================================================================

# Set up error handler and cleanup on exit
trap 'error_handler $LINENO' ERR
trap 'cleanup; exit' INT TERM EXIT

# Initialize and run
if [[ "${ENABLE_LOGGING}" == "true" ]]; then
    ensure_log_dir
fi
create_lockfile
check_dependencies
main
cleanup
```

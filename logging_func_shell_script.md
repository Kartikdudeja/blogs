# Logging Like a Pro: A Simple Yet Powerful Logger Function for Your Shell Scripts

> *“Great logs don't just tell you what went wrong — they help you understand why it happened.”*

Whether you're automating a deployment or writing a bash script to clean up temp files, there's one thing your future self will always thank you for: **clear, timestamped, consistent logging**.

![Linux Logging](./images/bash-logging.png)

## Why Even Bother With Logging?

Shell scripts often get the job done fast — but when they go wrong, debugging them can feel like trying to read hieroglyphics in the dark. Logging provides a flashlight. It tells you what ran, when it ran, and how it ran — especially when you're not watching.

This post walks you through a compact, reusable logger function for your shell scripts that:
- Categorizes messages by severity (INFO, DEBUG, ERROR, etc.)
- Automatically timestamps log entries
- Writes to a dedicated log file

## The Logger Function: A Quick Glance

Here’s the complete snippet of the logger function:

```bash
#!/bin/bash

logger(){

    # variables
    LOG_FILE_PATH="./<log-file-path>"
    LOG_FILE_NAME="script.log"

    # Usage: logger <SEVERITY> <LOG_MESSAGE>
    # Example: logger "INFO" "some message"
    # Severity Levels: DEBUG, INFO, WARNING, ERROR, EXCEPTION, CRITICAL

    if [ $# -eq 2 ]
    then
        SEVERITY="$1"
        LOG_MESSAGE="$2"
    elif [ $# -eq 1 ]
    then
        SEVERITY="INFO"
        LOG_MESSAGE="$1"
    else
        SEVERITY="EXCEPTION"
        LOG_MESSAGE="Wrong Syntax used for logger function. Usage: logger <SEVERITY> <LOG_MESSAGE>"
    fi
    
    echo -e "$(date '+%Y-%m-%d %T') $SEVERITY: $LOG_MESSAGE" >> $LOG_FILE_PATH/$LOG_FILE_NAME
}
```

---

## Example in Action

Here's how you might use this logger function inside your shell script:

```bash
#!/bin/bash

source ./logger.sh  # assuming you saved the function in logger.sh

logger "INFO" "Starting Execution Started..."

# Your Script Goes Here!
# Invoke `logger` function within your script to get detailed logs

logger "INFO" "Script Completed."

```

> **Pro Tip**: You can also wrap this into your CI/CD pipelines or cron jobs to get insights into when and why things run (or crash).

## Why It’s Cool

- **Readability:** Each log entry is clean, consistent, and timestamped.
- **Minimalism:** The function is only ~15 lines but incredibly flexible.

## Visualizing the Log File
```bash
tail -f script.log
```

```
2025-04-26 14:00:01 INFO: Starting Execution Started...
2025-04-26 14:00:05 INFO: Script Completed.
```

Simple, clear, and human-readable — exactly what you want when you're troubleshooting a production script that ran two weeks ago.

---

## Final Thoughts

Scripting isn't just about writing commands. It's about writing them smartly. By building in a logger function like this, you add **observability** to your scripts — and in the world of DevOps and automation, that's gold.

So the next time you’re writing a bash script, don’t just echo status messages to the screen. Log them like a pro.

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```

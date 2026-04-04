---
name: at
description: Schedule one-time future command execution using the Unix at command. Schedule jobs at specific times, list pending jobs, view job contents, and remove scheduled jobs. Use for deferred task execution, follow-up actions, and delayed operations.
compatibility: Requires 'at' package installed and 'atd' daemon running. Standard on most Unix systems.
---

# Unix at Command - One-Time Job Scheduling

Schedule commands to run once at a future time. Unlike cron (recurring), `at` is for one-shot execution. Jobs inherit the current environment (working directory, env vars) from when scheduled.

## Scheduling Jobs

### Relative Time

```bash
echo "command-to-run" | at now + 30 minutes
echo "command-to-run" | at now + 2 hours
echo "command-to-run" | at now + 1 day
echo "command-to-run" | at now + 1 week
```

### Specific Time Today

```bash
echo "command-to-run" | at 14:30
echo "command-to-run" | at 2:30 PM
echo "command-to-run" | at noon
echo "command-to-run" | at midnight
echo "command-to-run" | at teatime    # 4:00 PM
```

### Specific Date and Time

```bash
echo "command-to-run" | at 9:00 AM 2026-03-25
echo "command-to-run" | at 14:00 March 25
echo "command-to-run" | at noon tomorrow
echo "command-to-run" | at 10:00 AM next week
```

### From a Script File

```bash
at -f /path/to/script.sh now + 1 hour
at -f /path/to/script.sh 3:00 PM
```

### Multi-line Jobs

```bash
at now + 1 hour <<'EOF'
cd /working/directory
command-one
command-two
EOF
```

## Listing Scheduled Jobs

### List All Pending Jobs

```bash
atq
```

Output format:
```
3   Tue Mar 25 09:00:00 2026 a root
5   Wed Mar 26 14:30:00 2026 a root
```

Fields: job ID, scheduled time, queue letter, user.

### View Contents of a Specific Job

```bash
at -c <job-id>
```

Example:
```bash
at -c 3
```

This prints the full job script including captured environment and the command at the bottom.

## Removing Scheduled Jobs

### Remove by Job ID

```bash
atrm <job-id>
```

### Remove Multiple Jobs

```bash
atrm 3 5 7
```

### Remove All Jobs (Current User)

```bash
atq | awk '{print $1}' | xargs atrm
```

## Job Queues

Jobs can be placed in named queues (a-z). Higher letters run at lower priority (higher nice value):

```bash
echo "low-priority-task" | at -q z now + 10 minutes
echo "high-priority-task" | at -q a now + 10 minutes
```

Default queue is `a`.

## User, Shell, and Permissions

### Job Ownership

Jobs run as the user who scheduled them. If user `agent` schedules a job, it executes as `agent` with that user's UID/GID.

```bash
# See who owns each job (last field)
atq
# 3   Tue Mar 25 09:00:00 2026 a agent
```

### Shell Selection

At scheduling time, `at` captures the `$SHELL` environment variable. If unset, it falls back to the user's login shell from `/etc/passwd`. The job runs through that shell.

```bash
# Check what shell at will use
echo $SHELL
```

### Environment Capture

When you schedule a job, `at` captures:
- Current working directory
- All environment variables
- The shell

View exactly what was captured:

```bash
at -c <job-id>
```

This dumps the full script including all `export` statements and the command at the bottom.

### Access Control

Two files control who can use `at`:

| File | Behavior |
|------|----------|
| `/etc/at.allow` | If exists, only listed users can use `at` |
| `/etc/at.deny` | If allow doesn't exist, listed users are blocked |

If neither file exists, behavior varies by distribution (often only root).

In containers, you may need to ensure the user is permitted:

```bash
# Allow a specific user
echo "agent" >> /etc/at.allow

# Or remove all restrictions
rm -f /etc/at.deny
```

### Docker Considerations

The `atd` daemon runs as root, but jobs execute as the scheduling user:

```bash
# In entrypoint (runs as root)
atd                        # start daemon as root
exec gosu agent "$@"       # drop to agent user

# Later, agent schedules jobs - they run as agent
echo "command" | at now + 30 minutes
```

## Practical Patterns

### Schedule with Output Logging

By default `at` tries to mail output. In containers, redirect explicitly:

```bash
echo "command >> /var/log/scheduled.log 2>&1" | at now + 1 hour
```

### Self-Rescheduling Job

A job that schedules its next run dynamically:

```bash
#!/bin/bash
# do-work.sh
perform-task

# reschedule self based on result
if [ "$NEEDS_FOLLOWUP" = "true" ]; then
    echo "$0" | at now + 30 minutes
fi
```

### Conditional Scheduling

Schedule follow-up only if needed:

```bash
# In your script
if [ "$status" = "pending" ]; then
    echo "check-status.sh" | at now + 15 minutes
fi
```

### Check if Job Exists Before Scheduling

```bash
# Avoid duplicate scheduling
if ! atq | grep -q "pattern"; then
    echo "command" | at 9:00 AM
fi
```

## Time Expression Reference

| Expression | Meaning |
|------------|---------|
| `now` | Current time |
| `now + N minutes` | N minutes from now |
| `now + N hours` | N hours from now |
| `now + N days` | N days from now |
| `now + N weeks` | N weeks from now |
| `noon` | 12:00 PM |
| `midnight` | 12:00 AM |
| `teatime` | 4:00 PM |
| `tomorrow` | Same time tomorrow |
| `next week` | Same time next week |
| `HH:MM` | Specific time (24h) |
| `HH:MM AM/PM` | Specific time (12h) |
| `HH:MM YYYY-MM-DD` | Specific date and time |
| `HH:MM Month Day` | Specific date and time |

## Important Notes

- **atd must be running** - without the daemon, jobs queue but never execute
- **Jobs persist on disk** - stored in `/var/spool/at/`, survive daemon restarts
- **Environment captured** - jobs run with the env vars and working directory from when scheduled
- **No recurring jobs** - use cron/supercronic for that; `at` is strictly one-shot
- **Output handling** - redirect stdout/stderr explicitly in containerized environments

## Verifying atd is Running

```bash
pgrep atd
# or
ps aux | grep atd
```

Start if needed:
```bash
atd
```

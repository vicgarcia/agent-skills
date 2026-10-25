# atd

Schedule one-time, deferred command execution with the Unix `at` command. Unlike cron (recurring), `at` runs a job once at a future time.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

## What it covers

- **Scheduling** — relative times, specific times/dates, from a script file, and multi-line jobs.
- **Managing jobs** — listing pending jobs (`atq`), viewing a job's contents, and removing jobs.
- **Job queues, user/shell/permissions**, and the environment a job inherits.
- **Practical patterns**, a time-expression reference, and how to verify the `atd` daemon is running.

## Requirements

The `at` package installed and the `atd` daemon running. Standard on most Unix systems; not native to Windows.

```bash
sudo apt install at && sudo systemctl enable --now atd    # Debian/Ubuntu
```

## Canonical documentation

- [`at(1)` man page](https://man7.org/linux/man-pages/man1/at.1p.html)

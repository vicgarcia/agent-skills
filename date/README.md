# date

Date and time operations via the `date` system command. LLMs have unreliable internal date knowledge, so this skill provides ground-truth current time and calendar navigation.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

## What it covers

- **The `date` command** — current time, custom output formats, parsing.
- **Date arithmetic** — adding/subtracting days, weeks, months; differences between dates.
- **Timezones** — converting and displaying across zones.
- **Calendar navigation** — month boundaries, weekday lookups, week ranges.
- **Common calculations**, a quick reference, and agent rules.

## Requirements

POSIX `date` (Unix/macOS/Linux). GNU `date` is assumed for arithmetic (`-d`); macOS/BSD `date` uses `-v` syntax.

## Canonical documentation

- [GNU coreutils `date`](https://www.gnu.org/software/coreutils/manual/html_node/date-invocation.html)

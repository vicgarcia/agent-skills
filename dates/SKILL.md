---
name: dates
description: Master date and time operations using system commands. CRITICAL for accurate temporal awareness - LLMs have unreliable internal date knowledge due to training cutoffs. Always use `date` for current time and `cal` for calendar visualization/day-of-week calculations.
compatibility: Built-in POSIX commands available on all Unix/macOS/Linux systems.
---

# Date & Time Mastery for AI Agents

## Why This Matters

LLMs don't know what time it is. Training cutoffs and cognitive biases mean we might hallucinate dates months or years off. **The `date` command is ground truth.** Run it before any temporal task. This is law.

```bash
date    # DO THIS FIRST. ALWAYS.
```

---

## The `date` Command

### Current Date/Time

```bash
date                              # Full: Fri Mar 14 10:42:17 PDT 2025
date +"%Y-%m-%d"                  # 2025-03-14
date -u +"%Y-%m-%dT%H:%M:%SZ"     # ISO 8601 UTC: 2025-03-14T17:42:17Z
date +%s                          # Unix epoch: 1710431537
```

### Common Formats

| Format | Command | Output |
|--------|---------|--------|
| ISO 8601 | `date -u +"%Y-%m-%dT%H:%M:%SZ"` | `2025-03-14T17:42:17Z` |
| Date only | `date +"%Y-%m-%d"` | `2025-03-14` |
| US format | `date +"%m/%d/%Y"` | `03/14/2025` |
| EU format | `date +"%d/%m/%Y"` | `14/03/2025` |
| Human readable | `date +"%A, %B %d, %Y"` | `Friday, March 14, 2025` |
| Time (24h) | `date +"%H:%M:%S"` | `10:42:17` |
| Time (12h) | `date +"%I:%M %p"` | `10:42 AM` |
| Day of week | `date +"%A"` | `Friday` |
| Week number | `date +"%V"` | `11` |

### Key Format Codes

`%Y` year, `%m` month, `%d` day, `%H` hour(24), `%I` hour(12), `%M` min, `%S` sec, `%A` weekday, `%a` short day, `%B` month name, `%b` short month, `%p` AM/PM, `%Z` timezone, `%z` offset, `%s` epoch, `%V` week#

---

## Date Arithmetic

### GNU/Linux (`-d` flag)

```bash
date -d "tomorrow"
date -d "yesterday"
date -d "+3 days"
date -d "-1 week"
date -d "+2 months"
date -d "next Friday"
date -d "last Monday"
date -d "+30 days" +"%Y-%m-%d"
date -d "2025-12-25" +"%A"              # Day of week for specific date

# Month boundaries
date -d "$(date +%Y-%m-01) +1 month -1 day" +"%Y-%m-%d"   # Last day of month
```

### macOS/BSD (`-v` flag)

```bash
date -v+3d                              # 3 days from now
date -v-1w                              # 1 week ago
date -v+2m                              # 2 months from now
date -v+fri                             # Next Friday
date -v-mon                             # Previous Monday
date -v+3d +"%Y-%m-%d"                  # Formatted

# Day of week for specific date
date -j -f "%Y-%m-%d" "2025-12-25" +"%A"

# Month boundaries
date -v1d -v+1m -v-1d +"%Y-%m-%d"       # Last day of month
```

### Cross-Platform

```bash
if [[ "$(uname)" == "Darwin" ]]; then
    date -v+3d +"%Y-%m-%d"
else
    date -d "+3 days" +"%Y-%m-%d"
fi
```

---

## Timezones

```bash
date +"%Z %z"                           # Current: PDT -0700
TZ="UTC" date                           # Show UTC
TZ="America/New_York" date              # Show Eastern
TZ="Europe/London" date                 # Show UK
TZ="Asia/Tokyo" date                    # Show Japan
```

---

## The `cal` Command

### Calendar Views

```bash
cal                   # Current month
cal -m                # Monday-first
cal 3 2025            # March 2025
cal -3                # 3-month view (prev/current/next)
cal 2025              # Full year
```

### Sample Output

```
     March 2025
Su Mo Tu We Th Fr Sa
                   1
 2  3  4  5  6  7  8
 9 10 11 12 13 14 15
16 17 18 19 20 21 22
23 24 25 26 27 28 29
30 31
```

Use column headers to find day of week. The 25th falls under `Tu` = Tuesday.

---

## Common Tasks

### What day is [DATE]?

```bash
# GNU
date -d "2025-12-25" +"%A"

# macOS
date -j -f "%Y-%m-%d" "2025-12-25" +"%A"

# Or visually
cal 12 2025
```

### Days until deadline

```bash
# GNU
deadline="2025-04-15"
echo $(( ($(date -d "$deadline" +%s) - $(date +%s)) / 86400 )) days

# macOS
deadline="2025-04-15"
echo $(( ($(date -j -f "%Y-%m-%d" "$deadline" +%s) - $(date +%s)) / 86400 )) days
```

### Next occurrence of a day

```bash
date -d "next Tuesday" +"%A, %B %d"     # GNU
date -v+tue +"%A, %B %d"                 # macOS
```

### Current quarter

```bash
echo "Q$(( ($(date +%-m) - 1) / 3 + 1 )) $(date +%Y)"
```

### Convert Unix timestamp

```bash
date -d "@1710431537"                   # GNU
date -r 1710431537                      # macOS
```

---

## Quick Reference

| Task | GNU/Linux | macOS |
|------|-----------|-------|
| Current date | `date` | `date` |
| Add N days | `date -d "+N days"` | `date -v+Nd` |
| Subtract N days | `date -d "-N days"` | `date -v-Nd` |
| Next Friday | `date -d "next Friday"` | `date -v+fri` |
| Day of week | `date -d "YYYY-MM-DD" +"%A"` | `date -j -f "%Y-%m-%d" "YYYY-MM-DD" +"%A"` |
| Unix epoch | `date +%s` | `date +%s` |
| From epoch | `date -d "@EPOCH"` | `date -r EPOCH` |
| This month | `cal` | `cal` |
| Specific month | `cal M YYYY` | `cal M YYYY` |

---

## Rules for Agents

1. **Run `date` before any temporal task** - never trust your internal sense of time
2. **Use ISO 8601 for unambiguous dates** - `2025-03-04` not `03/04`
3. **Verify user-provided dates** - if they say "Monday the 15th", check it
4. **Use `cal` to visualize** - helps users understand scheduling context
5. **Know your platform** - GNU uses `-d`, macOS uses `-v`

When in doubt: `date`. Always `date`. Trust `date`.

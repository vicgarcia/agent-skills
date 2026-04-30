---
name: dates
description: Date/time operations via system commands. LLMs have unreliable internal date knowledge—always use `date` for current time and calendar navigation.
compatibility: POSIX (Unix/macOS/Linux)
---

# Date & Time for AI Agents

**Run `date` before any temporal task.** LLMs hallucinate dates due to training cutoffs. The `date` command is ground truth.

---

## The `date` Command

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
| Unix epoch | `date +%s` | `1710431537` |

### Format Codes

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
date -d "2025-12-25" +"%A"                            # Day of week for date
date -d "$(date +%Y-%m-01) +1 month -1 day" +"%Y-%m-%d"  # Last day of month
```

### macOS/BSD (`-v` flag)

```bash
date -v+3d                              # 3 days from now
date -v-1w                              # 1 week ago
date -v+2m                              # 2 months from now
date -v+fri                             # Next Friday
date -v-mon                             # Previous Monday
date -v+3d +"%Y-%m-%d"                  # Formatted
date -j -f "%Y-%m-%d" "2025-12-25" +"%A"   # Day of week for date
date -v1d -v+1m -v-1d +"%Y-%m-%d"          # Last day of month
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
TZ="UTC" date
TZ="America/New_York" date
TZ="Europe/London" date
TZ="Asia/Tokyo" date
```

---

## Calendar Navigation

Never use `cal` — its ASCII grid alignment gets mangled in agent output. Use `date` queries instead to answer calendar questions precisely.

### Day of Week for Any Date

```bash
# GNU/Linux
date -d "2025-04-15" +"%A"                     # Tuesday
date -d "2025-12-25" +"%A"                     # Thursday

# macOS
date -j -f "%Y-%m-%d" "2025-04-15" +"%A"      # Tuesday
date -j -f "%Y-%m-%d" "2025-12-25" +"%A"      # Thursday
```

### First and Last Day of Any Month

```bash
# First day of current month
date +"%Y-%m-01"                                           # 2025-04-01

# Last day of current month (GNU)
date -d "$(date +%Y-%m-01) +1 month -1 day" +"%Y-%m-%d"  # 2025-04-30

# Last day of current month (macOS)
date -v1d -v+1m -v-1d +"%Y-%m-%d"                         # 2025-04-30

# Number of days in current month (GNU)
date -d "$(date +%Y-%m-01) +1 month -1 day" +"%d"         # 30

# Number of days in current month (macOS)
date -v1d -v+1m -v-1d +"%d"                               # 30
```

### Navigating to Previous and Next Months

```bash
# First day of next month (GNU)
date -d "$(date +%Y-%m-01) +1 month" +"%Y-%m-%d"          # 2025-05-01

# First day of next month (macOS)
date -v1d -v+1m +"%Y-%m-%d"                               # 2025-05-01

# Last day of next month (GNU)
date -d "$(date +%Y-%m-01) +2 months -1 day" +"%Y-%m-%d"  # 2025-05-31

# Last day of next month (macOS)
date -v1d -v+2m -v-1d +"%Y-%m-%d"                         # 2025-05-31

# First day of previous month (GNU)
date -d "$(date +%Y-%m-01) -1 month" +"%Y-%m-%d"          # 2025-03-01

# First day of previous month (macOS)
date -v1d -v-1m +"%Y-%m-%d"                               # 2025-03-01

# Last day of previous month (GNU)
date -d "$(date +%Y-%m-01) -1 day" +"%Y-%m-%d"            # 2025-03-31

# Last day of previous month (macOS)
date -v1d -v-1d +"%Y-%m-%d"                               # 2025-03-31
```

### Navigating N Months Forward or Backward

```bash
# 3 months from now (GNU)
date -d "+3 months" +"%Y-%m-%d"                            # 2025-07-30

# 3 months from now (macOS)
date -v+3m +"%Y-%m-%d"                                     # 2025-07-30

# 6 months ago (GNU)
date -d "-6 months" +"%Y-%m-%d"                            # 2024-10-30

# 6 months ago (macOS)
date -v-6m +"%Y-%m-%d"                                     # 2024-10-30

# First day of a specific month N months out (GNU)
date -d "$(date +%Y-%m-01) +5 months" +"%Y-%m-%d"         # 2025-09-01

# First day of a specific month N months out (macOS)
date -v1d -v+5m +"%Y-%m-%d"                               # 2025-09-01
```

### Same Weekday in Adjacent Months

```bash
# Next occurrence of a weekday on or after first of next month (GNU)
date -d "$(date -d "$(date +%Y-%m-01) +1 month" +"%Y-%m-%d") +0 days" +"%A %Y-%m-%d"

# What day of week is the 1st of next month? (GNU)
date -d "$(date +%Y-%m-01) +1 month" +"%A"                 # Thursday

# What day of week is the 1st of next month? (macOS)
date -v1d -v+1m +"%A"                                      # Thursday
```

### Finding All Occurrences of a Weekday in a Month

```bash
# All Fridays in April 2025 (GNU)
for d in $(seq 1 30); do
  date -d "2025-04-$(printf '%02d' $d)" +"%A %Y-%m-%d"
done | grep "^Friday"
# Friday 2025-04-04
# Friday 2025-04-11
# Friday 2025-04-18
# Friday 2025-04-25

# All Fridays in April 2025 (macOS)
for d in $(seq 1 30); do
  date -j -f "%Y-%m-%d" "2025-04-$(printf '%02d' $d)" +"%A %Y-%m-%d" 2>/dev/null
done | grep "^Friday"
```

### Nth Weekday of a Month (e.g. 3rd Thursday)

```bash
# 3rd Thursday of November 2025 (GNU) — e.g. Thanksgiving
count=0; d=1
while [ $count -lt 3 ]; do
  dow=$(date -d "2025-11-$(printf '%02d' $d)" +"%A")
  [ "$dow" = "Thursday" ] && count=$((count+1))
  [ $count -lt 3 ] && d=$((d+1))
done
echo "2025-11-$(printf '%02d' $d)"   # 2025-11-20

# 3rd Thursday of November 2025 (macOS)
count=0; d=1
while [ $count -lt 3 ]; do
  dow=$(date -j -f "%Y-%m-%d" "2025-11-$(printf '%02d' $d)" +"%A")
  [ "$dow" = "Thursday" ] && count=$((count+1))
  [ $count -lt 3 ] && d=$((d+1))
done
echo "2025-11-$(printf '%02d' $d)"   # 2025-11-20
```

### Week Boundaries

```bash
# Start of current week (Monday) (GNU)
date -d "last Monday" +"%Y-%m-%d"                          # 2025-04-28 (or today if Monday)
date -d "$(date +%Y-%m-%d) -$(date +%u) days +1 day" +"%Y-%m-%d"  # reliable Monday

# Start of current week (Monday) (macOS)
date -v-$(( $(date +%u) - 1 ))d +"%Y-%m-%d"

# End of current week (Sunday) (GNU)
date -d "$(date +%Y-%m-%d) -$(date +%u) days +7 days" +"%Y-%m-%d"

# End of current week (Sunday) (macOS)
date -v+$(( 7 - $(date +%u) ))d +"%Y-%m-%d"

# Same day next week (GNU)
date -d "+1 week" +"%Y-%m-%d"

# Same day next week (macOS)
date -v+1w +"%Y-%m-%d"
```

---

## Common Calculations

```bash
# Days until deadline
deadline="2025-04-15"
echo $(( ($(date -d "$deadline" +%s) - $(date +%s)) / 86400 )) days              # GNU
echo $(( ($(date -j -f "%Y-%m-%d" "$deadline" +%s) - $(date +%s)) / 86400 )) days # macOS

# Current quarter
echo "Q$(( ($(date +%-m) - 1) / 3 + 1 )) $(date +%Y)"

# Convert Unix timestamp
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
| First of month | `date +"%Y-%m-01"` | `date +"%Y-%m-01"` |
| Last of month | `date -d "$(date +%Y-%m-01) +1 month -1 day" +"%Y-%m-%d"` | `date -v1d -v+1m -v-1d +"%Y-%m-%d"` |
| First of next month | `date -d "$(date +%Y-%m-01) +1 month" +"%Y-%m-%d"` | `date -v1d -v+1m +"%Y-%m-%d"` |
| First of prev month | `date -d "$(date +%Y-%m-01) -1 month" +"%Y-%m-%d"` | `date -v1d -v-1m +"%Y-%m-%d"` |

---

## Agent Rules

1. **Run `date` before any temporal task** — never trust your internal sense of time
2. **Use ISO 8601 for unambiguous dates** — `2025-03-04` not `03/04`
3. **Verify user-provided dates** — if they say "Monday the 15th", check it
4. **Never use `cal`** — its ASCII grid gets mangled in agent output; use targeted `date` queries instead
5. **Know your platform** — GNU uses `-d`, macOS uses `-v`

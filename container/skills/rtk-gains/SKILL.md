---
name: rtk-gains
description: Show RTK token savings for this container session. Use when the user asks about token savings, RTK stats, or runs /rtk-gains.
---

# /rtk-gains — RTK Token Savings

Run `rtk gain` and report the results.

```bash
rtk gain
```

Present the output as-is. If RTK is not found, not in PATH, or reports a GLIBC/compatibility error, say so briefly — RTK ARM64 Linux builds require glibc 2.38+ which may not be available in this container image.

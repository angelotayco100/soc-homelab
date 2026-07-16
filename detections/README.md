# Detections

This folder will hold one write-up per detection rule, in the format:

```
# Detection: [Name]

**MITRE ATT&CK:** [Technique ID]

**SPL Query:**
[query]

**What it detects:** [plain-English explanation]

**Simulated attack:** [exact command run on the Kali VM]

**Result:** [screenshot + what happened]

**False positive considerations:** [tuning notes]
```

**Status:** No detections have been built yet. This is the next phase of the
project — adversary simulation (Nmap scans, brute-force attempts) from the
Kali VM, followed by SPL correlation searches and alerting to detect them.
See the main [README](../README.md#whats-next) for current progress.

# Cybersecurity Lab Write-ups

A growing collection of cybersecurity lab notes and walkthroughs from platforms such as Hack The Box, TryHackMe, CyberDefenders, and other hands-on training environments.

The goal of this repository is to document repeatable methodology, explain the reasoning behind each step, and keep a practical reference for future labs.

> [!IMPORTANT]
> These write-ups are for educational use in authorized lab environments only. Some entries may contain spoilers. Active machines, flags, credentials, and other restricted platform content should not be published.

## Write-ups

### DFIR

| Lab | Platform | Difficulty | Focus | Summary |
| --- | --- | --- | --- | --- |
| [BFT](dfir/bft.md) | Hack The Box | Very Easy | MFT forensics | A downloaded Google Drive archive drops `invoice.bat`; the script body is recovered straight from its resident `$MFT` record, exposing the C2 address. |
| [MangoBleed](dfir/mangobleed.md) | Hack The Box | Very Easy | Linux triage | MongoDB 8.0.16 exposed to CVE-2025-14847; connection floods followed by SSH reuse of `mongoadmin`, LinPEAS enumeration, and suspected database exfiltration. |
| [RomCom](dfir/romcom.md) | Hack The Box | Very Easy | Windows triage | A malicious RAR exploits WinRAR path traversal (CVE-2025-8088) to write a backdoor and a Startup shortcut outside the extraction directory. |
| [JetBrains](dfir/jetbrains.md) | CyberDefenders | Easy | Network forensics | TeamCity 2023.11.3 auth bypass (CVE-2024-27198) creates an admin account, uploads a plugin web shell, tampers with stored credentials, and attempts a container escape. |

## Repository structure

Write-ups are grouped by discipline, one Markdown file per lab:

```
dfir/
  template.md   # section structure every write-up follows
  bft.md
  jetbrains.md
  mangobleed.md
  romcom.md
```

## Format

Each write-up follows [`dfir/template.md`](dfir/template.md):

- **Lab Info** - platform, difficulty, and focus area
- **Scenario** - the platform's briefing, verbatim
- **Investigation Actions Taken** - numbered steps showing the command run, the output returned, and what it means
- **Timeline** - facts only, in UTC
- **Indicators of Compromise** - grouped and defanged, with MITRE ATT&CK mappings
- **Summary of Incident** - initial access, actions taken, impact, detection, and what the evidence does not establish
- **Remediation** - specific and mapped to the identified root cause

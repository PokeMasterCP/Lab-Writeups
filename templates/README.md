# Write-up templates

Copy the template that matches the lab's primary workflow into the corresponding folder, using a descriptive filename such as `lab_name.md`. For mixed labs, choose the dominant workflow or your preferred format.

| Template | Use for | Destination |
| --- | --- | --- |
| [Blue Team](blue_team.md) | Alert triage, SOC investigations, threat hunting, log analysis, and detection-focused labs | `blue_team/` |
| [DFIR](dfir.md) | Reconstructing activity from disk, memory, network, email, or other forensic artifacts | `dfir/` |
| [Offensive Security](offensive_security.md) | Enumeration, exploitation, access, and privilege escalation | `offensive_security/` |

These templates follow [the repository editing instructions](../AGENTS.md) and the structures used in the existing write-ups. They provide writing prompts, not lab answers. Replace bracketed prompts with your own notes and remove all template comments before publication.

## Writing a draft

- Use the four-column Lab Info table. Standardize platform names, including `Hack The Box` and `CyberDefenders`.
- Preserve the platform's scenario verbatim except for defanging network indicators. List only tools you actually used.
- Keep challenge questions and answers in their original order. Write in first person when describing your actions, including the artifact or command, result, and your interpretation when documented.
- Include only commands and output you collected. Preserve redacted credentials exactly. Record flag retrieval or submission only when completed; omit flag values unless intentionally included for publication.
- Separate observed evidence from inference. Retain uncertainty, failed attempts, and limits on what the artifacts establish.
- Use only your documented events in the timeline, in chronological order. Convert to UTC only when the source timezone is known; resolve an unknown timezone before publication. Use `Timestamp unavailable` when an event has no timestamp, and never invent precision.
- List only indicators and ATT&CK mappings you identified. Use populated bold category labels such as `Network`, `Files`, `Accounts`, `Endpoints`, `Behavior`, and `MITRE ATT&CK`. Format entries as a backticked value followed by ` - description`.
- Write assessments, summaries, attack chains, and recommendations from your own work. Do not fill gaps with assumed results.

## Optional sections and gaps

For **Blue Team**, keep Timeline only when chronology matters and timestamps were supplied, Indicators of Compromise only when reportable indicators were identified, and Remediation only when you supplied recommendations. The Incident Assessment should contain your own disposition, scope, confidence, impact, and escalation decision to the extent documented.

For **DFIR**, the template includes the full expected section order. Record missing evidence or unfinished sections in your working notes for follow-up. If your source material cannot support a timeline, incident summary, or remediation section, flag that gap rather than inventing content or leaving an empty section in a completed write-up.

For **Offensive Security**, remove User Access when it is not distinct from the initial foothold, and remove Privilege Escalation, Post-Exploitation, or Cleanup when not applicable or documented. Keep Defensive Considerations only when you want that section and have supplied the content. Preserve challenge-question order and numbering across phase sections. Do not add incident-response sections by default.

## Defanging

Apply these rules everywhere, including prose, commands, filters, quoted output, tables, and code fences:

| Indicator | Publication format |
| --- | --- |
| IPv4 | `203[.]0[.]113[.]10` |
| IPv4 with port | `203[.]0[.]113[.]10:443` |
| Domain | `example[.]com` |
| Email address | `user@example[.]com` |
| HTTPS URL | `hxxps://example[.]com/path` |
| HTTP URL | `hxxp://example[.]com/path` |
| IPv6 | Replace every colon with `[:]` |

Keep ports, paths, query strings, and fragments intact. Do not defang file paths, hashes, CVE identifiers, ATT&CK identifiers, or software versions. Only trusted documentation citations, such as NVD, vendor advisories, and MITRE ATT&CK, may remain clickable.

## Before publishing

- Check the section order and original challenge-answer order.
- Verify that every claim, command, finding, and recommendation comes from your work and retains its original certainty.
- Check defanging, timestamp provenance, credential redaction, and Markdown formatting.
- Remove unused sections, bracketed prompts, template comments, and repetition. Track unresolved gaps separately.
- Add the completed write-up to the appropriate table and directory listing in the [repository README](../README.md).

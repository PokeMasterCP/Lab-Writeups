# Repository Editing Instructions

## Purpose

This repository is a public portfolio of cybersecurity lab write-ups. The analysis, investigative reasoning, commands, findings, and conclusions must remain the repository owner's own work so readers can use the write-ups to evaluate those skills.

Act as an editor and reviewer, not as the author or analyst. Improve how the owner's work is communicated without replacing it with AI-generated analysis.

## Editing Boundary

You may:

- Correct spelling, grammar, punctuation, and awkward phrasing.
- Clarify sentences while preserving their original technical meaning.
- Reduce repetition, filler, and unnecessary verbosity.
- Improve Markdown structure, code fences, tables, lists, and formatting.
- Standardize headings, platform names, timestamp formatting, and IOC presentation.
- Defang network indicators according to the rules below.
- Point out unsupported, ambiguous, or potentially inaccurate claims in your response to the owner.

You must not:

- Perform the lab or write the analysis on the owner's behalf.
- Add investigation steps, commands, tool output, findings, timestamps, IOCs, ATT&CK mappings, conclusions, incident impact, or remediation that the owner did not provide.
- Turn a brief answer into a new technical explanation based on your own knowledge.
- Claim that the owner ran a tool, observed an artifact, or reached a conclusion unless the draft says so.
- Fill an empty section or answer a missing challenge question. Report the gap to the owner instead.
- Convert an inference into a confirmed fact or strengthen the certainty of the owner's wording.
- Introduce threat attribution or other claims that are not supported in the draft.
- Use an official write-up or outside source to silently complete missing analysis.

If a technical statement may be wrong, preserve the owner's intended meaning where possible and flag the concern separately. Do not silently replace it with new analysis. If the intended meaning is unclear and a rewrite could change the finding, ask the owner rather than guessing.

If the owner explicitly requests fact-checking or research, provide the results separately as review feedback. Do not insert new analysis into the write-up without the owner's approval.

## Standard Write-up Structure

Keep completed write-ups in this section order:

1. `## Lab Info`
2. `## Scenario`
3. `## Investigation Actions Taken`
4. `## Timeline`
5. `## Indicators of Compromise`
6. `## Summary of Incident`
7. `## Remediation`

Reorder existing material when necessary, but do not generate content for a missing or incomplete section.

### Lab Info

Use a four-column table: Lab, Platform, Difficulty, and Focus. Standardize platform names, such as `Hack The Box` and `CyberDefenders`.

### Scenario

Preserve the platform's scenario wording. Only change it when necessary to defang a network indicator. Keep a short `Tools used` list when the owner has identified the tools used.

### Investigation Actions Taken

Keep the owner's numbered investigation steps in the challenge's original order. Each existing step should communicate, as concisely as the source material allows:

1. What artifact, filter, command, or technique the owner used.
2. What the owner found.
3. What the owner concluded from that result.

Preserve first-person language when it describes the owner's actions. Include only the relevant portions of commands and output already supplied by the owner. Do not turn routine steps into tutorials or add an explanation the owner did not write.

### Timeline

Format existing events as one factual bullet each:

- `YYYY-MM-DD HH:MM:SS UTC: Event.`
- `Timestamp unavailable: Event.`

Keep events chronological. Do not derive new events or timestamps from the investigation notes. If an existing timeline entry overstates what the evidence proves, flag it for the owner.

### Indicators of Compromise

Group existing indicators under populated bold subheadings such as `Network`, `Files`, `Accounts`, `Endpoints`, `Behavior`, and `MITRE ATT&CK`. Format each entry as:

- `value` - concise description

Do not add an indicator or ATT&CK mapping that the owner has not identified. Empty categories and placeholder bullets should not appear in a completed write-up; report them to the owner instead of filling them.

### Summary of Incident

Edit the owner's summary for clarity and concision. Preserve the stated sequence, confirmed impact, uncertainty, and evidentiary limitations. Do not write a new summary from the investigation steps when the owner has not supplied one.

### Remediation

Edit the owner's recommendations for clarity and specificity without adding new recommendations. Do not create containment, eradication, patching, detection, or credential-rotation guidance on the owner's behalf.

## Defanging Rules

Defang every IP address, domain, and lab-derived URL everywhere it appears, including investigation steps, quoted output, timelines, IOC lists, summaries, and remediation.

- IPv4: `203[.]0[.]113[.]10`
- IPv4 with port: `203[.]0[.]113[.]10:443`
- Domain: `example[.]com`
- HTTPS URL: `hxxps://example[.]com/path`
- HTTP URL: `hxxp://example[.]com/path`
- IPv6: replace each colon with `[:]`

Keep ports, paths, query strings, and fragments intact. Do not defang file paths, hashes, CVE identifiers, MITRE ATT&CK identifiers, or software version numbers. Trusted citations to documentation such as NVD, vendor advisories, and MITRE ATT&CK may remain clickable; this is the only URL exception.

Apply defanging consistently. An indicator that is defanged in the IOC section must also be defanged in prose, code blocks, filters, commands, and quoted artifact output.

## Style

- Preserve the owner's technical voice and level of detail.
- Prefer direct sentences and short paragraphs.
- Keep the revision at or below the draft's original length when practical. There is no minimum word count.
- Remove repetition before removing supporting evidence.
- Use backticks for commands, filenames, paths, hashes, accounts, IOCs, CVEs, and ATT&CK IDs.
- Use consistent capitalization and UTC timestamps.
- Expand an abbreviation on first use when the owner has provided enough context to do so accurately.
- Avoid filler, dramatic language, unsupported attribution, and repeated conclusions.
- Preserve redacted credentials exactly as provided. Never reconstruct or expose secrets.
- Do not add template comments, placeholder text, or empty sections to a completed write-up.

## Review Checklist

Before finishing, verify that:

1. The owner's challenge answers remain in their original order and retain their original meaning.
2. No investigation, evidence, conclusion, or recommendation was authored on the owner's behalf.
3. Every IP, domain, and lab-derived URL is defanged everywhere it appears.
4. Existing timestamps use UTC or `Timestamp unavailable` and no timestamp was invented.
5. Claims retain the same level of certainty as the source draft.
6. Redacted credentials remain redacted.
7. The Markdown renders cleanly and contains no accidental placeholders.
8. The revision is concise and does not repeat the same explanation unnecessarily.

After editing, briefly summarize the editorial changes. Separately list missing evidence, incomplete sections, possible technical inaccuracies, or claims that need the owner's review. Do not reproduce the entire write-up in the response.

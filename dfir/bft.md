## Lab Info

| Lab | Platform | Difficulty | Focus |
| --- | --- | --- | --- |
| BFT | HackTheBox | Very Easy | DFIR |

## Scenario

In this Sherlock, you will become acquainted with MFT (Master File Table) forensics. You will be introduced to well-known tools and methodologies for analyzing MFT artifacts to identify malicious activity. During our analysis, you will utilize the MFTECmd tool to parse the provided MFT file, TimeLine Explorer to open and analyze the results from the parsed MFT, and a Hex editor to recover file contents from the MFT.

Tools used:

- MFTECmd
- Timeline Explorer
- HxD Hex Editor

## Investigation Actions Taken

1. With the provided KAPE triage package, I started by parsing the Windows MFT so I could review it in Timeline Explorer:

```powershell
& 'C:\Tools\Zimmerman Tools\net9\MFTECmd.exe' -f ".\C\`$MFT" --csv 'C:\Users\Christian\Desktop\BFT\' --csvf MFT_ANALYSIS.csv
```

I opened the parsed `MFT_ANALYSIS.csv` file with Timeline Explorer to begin investigating. The first question asked about a ZIP file that was received on Feb 13th. I filtered the **Extension** column for `.zip`, which yielded a single result: **Stage-20240213T093324Z-001.zip**, created at 2024-02-13 16:34:40.

2. With the filename known, I filtered on it and found the matching `Zone.Identifier` alternate data stream. Because the ADS is resident in the MFT record, MFTECmd surfaces its contents in the **Zone Id Contents** column, which gave the full download URL and confirmed the file came from `storage.googleapis.com` rather than a local source.

3. Testing the theory that the archive was expanded in the same Downloads directory, I searched **Parent Path** for `.\Users\simon.stark\Downloads\S` and found multiple files created under `.\Users\simon.stark\Downloads\Stage-20240213T093324Z-001\Stage`. An immediate standout is `invoice.bat`, a Windows batch file.

4. The **Created0x10** timestamp for `invoice.bat` is 2024-02-13 16:38:39, roughly four minutes after the archive itself landed on disk.

5. The MFT entry number for `invoice.bat` is 23436. MFT records are a fixed 1024 bytes, so the record's offset into `$MFT` is 23436 * 1024 = 23,998,464 bytes, or `0x16E3000`.

6. Opening `$MFT` in HxD and jumping to that offset, I selected the 1024-byte block for the record. `invoice.bat` is small enough that its `$DATA` attribute is resident, so the full script body is stored inside the record. The decoded text revealed the C2 address `43[.]204[.]110[.]203:6666`.

## Timeline

- 2024-02-13 16:34:40 UTC: `Stage-20240213T093324Z-001.zip` was written to `C:\Users\simon.stark\Downloads`. Its `Zone.Identifier` shows it was downloaded from `storage.googleapis.com` as a Google Drive bulk export. How the user was pointed at that link is not established by the MFT alone.
- 2024-02-13 16:38:39 UTC: The archive was extracted into `Downloads\Stage-20240213T093324Z-001\Stage`, dropping the stager `invoice.bat`.

## Indicators of Compromise

**Files**

- `Stage-20240213T093324Z-001.zip` - malicious archive delivered to the user
- `invoice.bat` - batch stager extracted from the archive

**Network**

- `43[.]204[.]110[.]203:6666` - C2 endpoint hardcoded in `invoice.bat`
- hxxps[://]storage[.]googleapis[.]com/drive-bulk-export-anonymous/20240213T093324[.]039Z/4133399871716478688/a40aecd0-1cf3-4f88-b55a-e188d5c1c04f/1/c277a8b4-afa9-4d34-b8ca-e1eb5e5f983c?authuser - archive download URL

## Summary of Incident

On February 13, 2024, the user `simon.stark` downloaded `Stage-20240213T093324Z-001.zip` to their Downloads folder at 16:34:40 UTC. The `Zone.Identifier` stream on the archive records the source as a `storage.googleapis.com` bulk-export link, indicating the file was pulled from Google Drive over the browser rather than copied from local media. Roughly four minutes later, at 16:38:39 UTC, the archive was extracted in place, writing `invoice.bat` into `Downloads\Stage-20240213T093324Z-001\Stage`.

`invoice.bat` is small enough to be resident in its MFT record, so the script body was recoverable directly from `$MFT` at offset `0x16E3000` without the file itself. Its contents show the script reaching out to `43.204.110.203` on TCP 6666, making it a stager for an attacker-controlled C2 channel.

The MFT is a record of file metadata, not of process activity, so it establishes that the stager was written to disk but cannot confirm that it executed or that the C2 connection was ever made. Confirming execution requires Prefetch, Amcache, `$UsnJrnl`, EDR telemetry, or Sysmon/4688 process-creation logs, none of which are in scope for this exercise. Impact therefore remains undetermined.

## Remediation

The host should be isolated pending confirmation of whether `invoice.bat` executed. That question is the priority for the next collection: pull Prefetch, Amcache, `$UsnJrnl`, and the Windows Security and Sysmon event logs, and check them for `invoice.bat` and for `cmd.exe` spawning network activity around 16:38 UTC on 2024-02-13.

Block `43.204.110.203` at the perimeter and search proxy and firewall logs for any connection to that address on port 6666, both from this host and estate-wide, to scope whether other machines were staged the same way. The `storage.googleapis.com` base domain cannot be blocked outright without breaking legitimate Google Workspace use; the practical control is to alert on downloads from the `drive-bulk-export-anonymous` path, which is anonymous-share traffic and rare in normal business use.

The delivery vector is still unknown. The mail gateway should be searched for messages to `simon.stark` containing that Drive link so any other recipients can be identified and the message purged. Longer term, the root cause here is a user executing an archive from an anonymous Drive share, so the controls that address it are blocking or prompting on script file types (`.bat`, `.cmd`, `.js`, `.vbs`) launched from user Downloads directories via AppLocker or WDAC, and alerting when a batch or script file is written into a Downloads subdirectory by an archive utility.

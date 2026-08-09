## Lab Info

| Lab | Platform | Difficulty | Focus |
| --- | --- | --- | --- |
| RomCom | HackTheBox | Very Easy | DFIR |

## Scenario

Susan works at the Research Lab in Forela International Hospital. A Microsoft Defender alert was received from her computer, and she also mentioned that while extracting a document from the received file, she received tons of errors, but the document opened just fine. According to the latest threat intel feeds, WinRAR is being exploited in the wild to gain initial access into networks, and WinRAR is one of the software programs the staff uses. You are a threat intelligence analyst with some background in DFIR. You have been provided a lightweight triage image to kick off the investigation while the SOC team sweeps the environment to find other attack indicators.

## Investigation Actions Taken

1. Searching Google for WinRAR exploits associated with the RomCom threat group reveals it was [CVE-2025-8088](https://nvd.nist.gov/vuln/detail/CVE-2025-8088)

2. Reviewing the NVD entry for this CVE reveals it is a path traversal vulnerability. A crafted archive uses alternate data streams to write files outside the directory the user selected, including into the user's Startup folder. Code execution follows from where the files land rather than from the extraction itself, which also explains the extraction errors Susan reported: the archive contains entries that WinRAR cannot write normally, so the visible document extracts fine while the traversal entries throw errors.

3. The task provided us with a `.vhdx` file, so the first step I took was to get the hash with:
```powershell
Get-FileHash -Path .\2025-09-02T083211_pathology_department_incidentalert.vhdx -Algorithm SHA256

Algorithm  Hash
---------  ----
SHA256     cccd85ef47fd372a1ebf0f759d212dda378a8b0832bd98d4263eb6a9c7d99ee1
```
This can be used in the future to confirm the integrity of the image. The next step was to right click on the image and mount it. Once mounted, I checked the CopyLog to see what was collected:
```powershell
PS F:\> Import-Csv 'F:\2025-09-02T08_32_11_5202830_CopyLog.csv' | Format-Table -AutoSize

CopiedTimestamp             SourceFile               DestinationFile                       FileSize  SourceFileSha1                           DeferredCopy CreatedOnUtc                ModifiedOnUtc               LastAccessedOnUt
                                                                                                                                                                                                                   c
---------------             ----------               ---------------                       --------  --------------                           ------------ ------------                -------------               ----------------
2025-09-02 08:32:16.9881736 C:\$Extend\$UsnJrnl:$J   C:\Windows\Temp\triage\C\$Extend\$J   60146904  3FE93030A4F1BEDE41F7EB82499E43D89C24EF17 True         2025-09-01 20:10:49.1924250 2025-09-01 20:10:49.1924250 2025-09-01 20...
2025-09-02 08:32:17.0195635 C:\$Extend\$UsnJrnl:$Max C:\Windows\Temp\triage\C\$Extend\$Max 32        0DE103B7CEF8FEA91D8974825528695F89B5763B True         2025-09-01 20:10:49.1924250 2025-09-01 20:10:49.1924250 2025-09-01 20...
2025-09-02 08:32:22.4740385 C:\$MFT                  C:\Windows\Temp\triage\C\$MFT         143130624 75649E0EB13F5E96E23284E3EEA265B14ACE0FF7 True         2025-09-01 21:08:54.0003510 2025-09-01 21:08:54.0003510 2025-09-01 21...
```
With the MFT and USN journal available, I used Zimmerman's MFTECmd.exe to parse them:
```powershell
.\MFTECmd.exe -f 'F:\C\$MFT' --csv C:\Users\Christian\Desktop\RomCom\out --csvf mft.csv
.\MFTECmd.exe -f 'F:\C\$Extend\$J' -m 'F:\C\$MFT' --csv C:\Users\Christian\Desktop\RomCom\out --csvf usn.csv
```

With the data parsed, I can now investigate with Timeline Explorer. I started with the MFT and searched for `Susan\Documents` as the parent path and found the malicious archive file was **Pathology-Department-Research-Records.rar**

4. With the filename now found, I simply checked the **Created** column to confirm the file was created on disk at 2025-09-02 08:13:50

5. I checked the USN for **Pathology-Department-Research-Records.rar** and noted an **ObjectIDChange** action on the archive at 2025-09-02 08:14:04, indicating the point the archive was accessed and extraction began.

6. Looking at the MFT I noticed a .pdf file titled **Genotyping_Results_B57_Positive.pdf** alongside the archive. I cross checked with USN and confirmed it was created shortly after the previous **ObjectIDChange** timestamp at 2025-09-02 08:14:18

7. I filtered the USN to view all events around the time the decoy .pdf file was created and noticed `C:\Users\Susan\AppData\Local\ApbxHelper.exe` was created at the same time.

8. While still reviewing the USN events from question 7, I noticed `C:\Users\Susan\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Display Settings.lnk` was created at the same time as the .pdf and .exe. WinRAR has no legitimate reason to write into the user's Startup folder, and that directory is outside the extraction path Susan chose. This is the path traversal in CVE-2025-8088 being used to plant an autorun shortcut, which will launch the payload at every logon.

9. Cross referencing with MITRE ATT&CK, dropping a shortcut that runs automatically at logon to launch a payload maps to **T1547.009 – Shortcut Modification**, a sub-technique of T1547 (Boot or Logon Autostart Execution) under the Persistence tactic.

10. Searching through USN, I see the decoy file **Genotyping_Results_B57_Positive.pdf** has an **ObjectIDChange** event at 2025-09-02 08:15:05

## Timeline

- 2025-09-02 08:13:50 UTC: The malicious archive `Pathology-Department-Research-Records.rar` was created in Susan's Documents folder.
- 2025-09-02 08:14:04 UTC: The archive was accessed and extraction began.
- 2025-09-02 08:14:18 UTC: Extraction wrote three files:
    - `Genotyping_Results_B57_Positive.pdf` - decoy document, so the user does not suspect anything
    - `C:\Users\Susan\AppData\Local\ApbxHelper.exe` - malicious backdoor
    - `C:\Users\Susan\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Display Settings.lnk` - autorun trigger for the backdoor
- 2025-09-02 08:15:05 UTC: Susan opened the decoy PDF.

## Indicators of Compromise

**Files**

- `Pathology-Department-Research-Records.rar` - malicious archive, delivered to Susan
- `Genotyping_Results_B57_Positive.pdf` - decoy document
- `C:\Users\Susan\AppData\Local\ApbxHelper.exe` - backdoor
- `C:\Users\Susan\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Display Settings.lnk` - persistence

**MITRE ATT&CK**

- T1547.009 - Boot or Logon Autostart Execution: Shortcut Modification

## Summary of Incident

On September 2, 2025, Microsoft Defender alerted on Susan's workstation in the Forela International Hospital research lab. Susan reported that extracting a document from an archive she had received produced a large number of errors, although the document itself opened normally. Those errors are the signature of CVE-2025-8088 exploitation: the archive carries alternate data stream entries whose paths traverse outside the extraction directory, so the decoy document extracts cleanly while the traversal entries fail visibly.

Triage of the MFT and USN journal shows the archive `Pathology-Department-Research-Records.rar` was written to Susan's Documents folder at 08:13:50 UTC and extracted at 08:14:04 UTC. Extraction produced three files within the same second at 08:14:18 UTC. One was the decoy PDF Susan opened at 08:15:05 UTC. The other two were written outside the extraction directory: a backdoor at `%LOCALAPPDATA%\ApbxHelper.exe` and a shortcut named `Display Settings.lnk` in the user's Startup folder, which establishes persistence by launching the backdoor at each logon. This activity pattern and the ApbxHelper filename are consistent with reporting on the RomCom threat group, which was the first actor observed exploiting this vulnerability in the wild.

## Remediation

Susan's workstation should be isolated from the network and a full disk image plus memory capture taken before anything is removed, since the current triage set cannot answer whether the backdoor ran. Once evidence is preserved, the host should be reimaged rather than cleaned, because the backdoor's capabilities are unknown. If reimaging has to wait, `Display Settings.lnk` should be deleted from the Startup folder to break persistence and `ApbxHelper.exe` removed from `%LOCALAPPDATA%`. Susan's domain credentials and any secrets used from that host should be rotated, and her mailbox reviewed to identify the delivery message so it can be pulled from any other recipients.

WinRAR must be updated to 7.13 or later on every host where staff use it, since 7.12 and below are vulnerable and WinRAR has no automatic update mechanism. Because the traversal writes to a predictable location, the SOC should hunt the estate for `ApbxHelper.exe`, for the archive filename and hash, and for any other unexpected executables or shortcuts written into user Startup folders. Detection should be added for file creation events in `\Start Menu\Programs\Startup\` where the creating process is an archive utility, which catches this technique regardless of the payload name. Finally, the incident should be assessed against the hospital's breach notification obligations, since the compromised user has access to pathology research records.

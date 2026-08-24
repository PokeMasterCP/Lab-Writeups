## Lab Info

| Lab | Platform | Difficulty | Focus |
| --- | --- | --- | --- |
| MangoBleed | Hack The Box | Very Easy | Linux forensics |

## Scenario

You were contacted early this morning to handle a high‑priority incident involving a suspected compromised server. The host, mongodbsync, is a secondary MongoDB server. According to the administrator, it's maintained once a month, and they recently became aware of a vulnerability referred to as MongoBleed. As a precaution, the administrator has provided you with root-level access to facilitate your investigation.

You have already collected a triage acquisition from the server using UAC. Perform a rapid triage analysis of the collected artifacts to determine whether the system has been compromised, identify any attacker activity (initial access, persistence, privilege escalation, lateral movement, or data access/exfiltration), and summarize your findings with an initial incident assessment and recommended next steps.

Tools used:

- `rg`
- `grep`
- `jq`

## Investigation Actions Taken

1. The scenario mentions the MongoBleed vulnerability, so I first investigated it and confirmed that the host was vulnerable. The CVE in question is [`CVE-2025-14847`](https://nvd.nist.gov/vuln/detail/CVE-2025-14847).

2. Per NVD, this CVE affects MongoDB Server v8 versions before `8.0.17`. While reviewing the provided files, I found one containing the output of `dpkg -l`, which lists installed packages. Running `grep mongo live_response/packages/dpkg_-l.txt` confirmed that the host was running MongoDB `8.0.16`, an affected version.

3. The included files were a UAC triage fileset containing all files on disk. Searching online for MongoDB's default log path identified `/var/log/mongodb/mongod.log`, which I confirmed by reviewing `/etc/mongod.conf`. I used `rg -o '\b(?:\d{1,3}\.){3}\d{1,3}\b' mongod.log | sort | uniq` to search for IPv4 addresses and found three matches, two of which were local addresses. Viewing a complete log message confirmed that the remaining address represented a connection: `grep "65[.]0[.]76[.]43" mongod.log | head -n1 | jq`

    ```json
    {
      "t": {
        "$date": "2025-12-29T05:25:52.743+00:00"
      },
      "s": "I",
      "c": "NETWORK",
      "id": 22943,
      "ctx": "listener",
      "msg": "Connection accepted",
      "attr": {
        "remote": "65[.]0[.]76[.]43:35340",
        "isLoadBalanced": false,
        "uuid": {
          "uuid": {
            "$uuid": "099e057e-11c1-46ed-b129-a158578d2014"
          }
        },
        "connectionId": 1,
        "connectionCount": 1
      }
    }
    ```

4. Reviewing `/var/log/mongodb/mongod.log`, I found that the first "Connection accepted" event from the threat actor's IP occurred at `2025-12-29T05:25:52.743+00:00`.

5. I used `grep 65[.]0[.]76[.]43 mongod.log | wc -l` to count the events from the threat actor, including both "Connection accepted" and "Connection ended" events.

6. I checked `/var/log/auth.log`, which contains information about authentication attempts, using `grep "mongoadmin" auth.log | grep "Accepted"`.

7. Knowing how the attacker gained access and that the affected account was `mongoadmin`, I pivoted to that user's home directory and checked `.bash_history` for executed commands.

8. In the same `.bash_history` file, I found that the attacker moved to `/var/lib/mongodb`, installed `zip`, compressed the directory's contents, and launched a Python web server, suggesting data exfiltration.

## Timeline

- 2025-12-29 05:25:52 UTC: The attacker started malicious connections against the MongoDB server.
- 2025-12-29 05:40:03 UTC: The attacker gained remote access to the victim over SSH as `mongoadmin`.
- Timestamp unavailable: The attacker downloaded the `linpeas.sh` tool for host enumeration and privilege escalation.
- Timestamp unavailable: Bash history recorded the attacker archiving `/var/lib/mongodb` and launching a Python web server, suggesting data exfiltration.

## Indicators of Compromise

**Network**

- `65[.]0[.]76[.]43` - attacker source IP

**Accounts**

- `mongoadmin` - account the attacker authenticated as over SSH

**Files**

- `linpeas.sh` - privilege-escalation enumeration script downloaded to the host
- `/var/lib/mongodb` - MongoDB data directory archived and staged for exfiltration

## Summary of Incident

On December 29, 2025, at 05:25:52 UTC, MongoDB logs recorded a high volume of short-lived connections from `65[.]0[.]76[.]43`. The server was running MongoDB `8.0.16`, a version affected by `CVE-2025-14847`, an unauthenticated memory-disclosure vulnerability. The same source later authenticated to the host over SSH as `mongoadmin`. This sequence is consistent with credentials being exposed through MongoBleed and subsequently reused.

The `mongoadmin` Bash history records execution of LinPEAS through a `curl | sh` pipeline, indicating privilege-escalation enumeration. It also records access to `/var/lib/mongodb` followed by the launch of a Python HTTP server, suggesting data exfiltration.

## Remediation

The MongoDB server should be isolated while the investigation is completed, and relevant logs and forensic evidence should be preserved. Patching MongoDB to the latest version would protect against `CVE-2025-14847`. Because the Bash history suggests data exfiltration, any credentials and secrets stored on the server should also be rotated. Network access to MongoDB should be restricted to authorized systems, and the threat actor's IP address should be blocked and searched for across other available logs and systems. The database contents need to be reviewed to determine the impact and any public-disclosure responsibilities.

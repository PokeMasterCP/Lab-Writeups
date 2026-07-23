# Lab Info

| Lab | Platform | Difficulty | Focus |
| --- | --- | --- | --- |
| MangoBleed | HackTheBox | Very Easy | DFIR |

## Scenario

You were contacted early this morning to handle a high‑priority incident involving a suspected compromised server. The host, mongodbsync, is a secondary MongoDB server. According to the administrator, it's maintained once a month, and they recently became aware of a vulnerability referred to as MongoBleed. As a precaution, the administrator has provided you with root-level access to facilitate your investigation.

You have already collected a triage acquisition from the server using UAC. Perform a rapid triage analysis of the collected artifacts to determine whether the system has been compromised, identify any attacker activity (initial access, persistence, privilege escalation, lateral movement, or data access/exfiltration), and summarize your findings with an initial incident assessment and recommended next steps.

## Investigation Actions Taken

1. The scenario mentions the MongoBleed vulnerability, so the first step was to investigate it and confirm that the host was vulnerable. The CVE in question is CVE-2025-14847. Here is the [NIST NVD page](https://nvd.nist.gov/vuln/detail/CVE-2025-14847) for it.

2. Per NVD, this CVE affects all MongoDB Server v8 versions prior to 8.0.17. While reviewing the provided files, I found one containing the output of `dpkg -l`, which lists all installed packages. Using `grep mongo live_response/packages/dpkg_-l.txt` confirmed that the MongoDB version on this host was 8.0.16 and vulnerable.

3. The included files were a UAC triage fileset containing all files on disk. Searching online for MongoDB's default logging path showed that it was `/var/log/mongodb/mongod.log`, which I confirmed by reviewing `/etc/mongod.conf`. I used `rg -o '\b(?:\d{1,3}\.){3}\d{1,3}\b' mongod.log | sort | uniq` to search for all IPv4 addresses and found only three matches, two of which were local addresses. Viewing an entire log message confirmed that the remaining address represented a connection: `grep "65.0.76.43" mongod.log | head -n1 | jq`

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
    "remote": "65.0.76.43:35340",
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

4. Reviewing the same `/var/log/mongodb/mongod.log` file, I found that the first "Connection accepted" event from the threat actor's IP occurred at 2025-12-29T05:25:52.743+00:00.

5. I used `grep 65.0.76.43 mongod.log | wc -l` to count the total number of events from the threat actor, including both "Connection accepted" and "Connection ended" events.

6. I checked `/var/log/auth.log`, which contains information about authentication attempts, using `grep "mongoadmin" auth.log | grep "Accepted"`.

7. Knowing how the attacker gained access and that the affected user was **mongoadmin**, I pivoted to that user's home directory and checked their `.bash_history` for commands that were run.

8. Checking the same `.bash_history` file, I found that the attacker moved to the `/var/lib/mongodb` directory. They then installed `zip` and compressed everything in that directory before exfiltrating it with a Python web server.

## Timeline

- 2025-12-29T05:25:52 UTC: The attacker started malicious connections against the MongoDB server.
- 2025-12-29T05:40:03 UTC: The attacker gained remote access to the victim.
- Timestamp unavailable: The attacker downloaded the `linpeas.sh` tool for host enumeration and privilege escalation.
- Timestamp unavailable: The attacker exfiltrated the contents of `/var/lib/mongodb` via a Python web server.

## Indicators of Compromise

**IP**

65[.]0[.]76[.]43

## Summary of Incident

On December 29, 2025, at 05:25:52 UTC, MongoDB logs recorded a high volume of short-lived connections from 65[.]0[.]76[.]43. The server was running MongoDB 8.0.16, a version affected by CVE-2025-14847, an unauthenticated memory-disclosure vulnerability. The same source later authenticated to the host over SSH as `mongoadmin`. This sequence is consistent with credentials being exposed through MongoBleed and subsequently reused.

The `mongoadmin` Bash history records execution of LinPEAS through a `curl | sh` pipeline, indicating privilege-escalation enumeration. It also records access to `/var/lib/mongodb` followed by the launch of a Python HTTP server, suggesting data exfiltration.

## Remediation

The MongoDB server should be isolated while the investigation is completed, and relevant logs and forensic evidence should be preserved. Patching the MongoDB server to the latest version would protect against CVE-2025-14847. Due to data exfiltration, any credentials and secrets stored on the server should also be rotated. Network access to MongoDB should be restricted to authorized systems, and the threat actor's IP address should be blocked and searched for across other available logs and systems. The contents of the database need to be reviewed to determine the impact and any public disclosure responsibilities.

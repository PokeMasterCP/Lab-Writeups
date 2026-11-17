## Lab Info

| Lab | Platform | Difficulty | Focus |
| --- | --- | --- | --- |
| SSHole | Hack The Box | Very Easy | DFIR |

## Scenario

Suspicious traffic was observed from a misconfigured jumpbox to a suspicious domain. You have been provided with evidence pulled from the jumpbox server. Determine how the attackers gained access to the server and what activity they performed afterwards.

Tools used:

- `cat`
- `grep`
- `ls`
- `cd`

## Investigation Actions Taken

1. **What Linux distribution name and version was the targeted server running?**

    The files provided were a dump of OS files. To determine the OS, I checked `/etc/os-release`, which provided `Alpine Linux 3.24.3` as the result.

2. **What is the CVE ID of the vulnerability abused by the attacker to gain access?**

    I Googled the OS with CVE and quickly found articles on [`CVE-2026-123158`](https://nvd.nist.gov/vuln/detail/CVE-2026-123158). This is an OpenSSH password bypass vulnerability that affects this version.

3. **When did the attacker exploit the vulnerability?**

    Since the CVE allows a remote user to bypass authentication on SSH, I checked `/var/log/messages` and found a successful login for the service account `adminsvc` at `Nov 6 11:27:08`.

4. **Which user account was abused by the attacker?**

    The log `/var/log/messages` shows authentication attempts, which is how I confirmed the user `adminsvc` had a successful authentication.

5. **What string value causes the authentication bypass?**

    I found [this Alpine Linux advisory](https://alpinelinux.org/posts/2026-11-05-openssh-password-authentication-bypass/), which shows the string as `*`.

6. **What command did the attacker use to scan the network reachable via the jumpbox, allowing them to target internal developer systems?**

    I checked the `.bash_history` file, which contains the commands run by that user, and found a loop used to scan the network:

    ```bash
    for ip in {1..254}; do nc -zvw 1 192[.]168[.]18[.]$ip 1-65535 2>&1 | grep -E "succeeded|open"; done
    ```

7. **What was the exfiltration URI where the attacker exfiltrated a sensitive file?**

    Continuing with the `.bash_history` file, I can see the exfil URI that was used: `hxxps://devv[.]coding[.]eu/process`.

## Timeline

- Nov  6 11:27:08 `adminsvc` successful login

## Indicators of Compromise

**Network**

- `hxxps://devv[.]coding[.]eu/process` - exfil URI found in `.bash_history`
- `192[.]168[.]72[.]1` - source of the `adminsvc` remote login

**Behavior**

- Password logins for accounts with no configured passwords

## Summary of Incident

The attacker exploited `CVE-2026-123158` by sending a `*` to a service account that had no password.

## Remediation

- Update OpenSSH to at least 10.6p2
- Disable password logins or use PAM

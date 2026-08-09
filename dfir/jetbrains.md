## Lab Info

| Lab | Platform | Difficulty | Focus |
| --- | --- | --- | --- |
| JetBrains Lab | CyberDefenders | Easy | Network Forensics |

## Scenario

During a recent security incident, an attacker successfully exploited a vulnerability in our web server, allowing them to upload webshells and gain full control over the system. The attacker utilized the compromised web server as a launch point for further malicious activities, including data manipulation.

As part of the investigation, you are provided with a packet capture (PCAP) of the network traffic during the attack to piece together the attack timeline and identify the methods used by the attacker. The goal is to determine the initial entry point, the attacker's tools and techniques, and the compromise's extent.

## Investigation Actions Taken

1. Loading the provided `.pcap`, I see it contains 33,279 packets. The scenario mentions the threat actor uploaded a webshell, so I searched for `http.request.method == "POST" && http.content_type contains "multipart"`, which returned only two packets, both from the same public IP of 23[.]158[.]56[.]196 against the server 172[.]31[.]25[.]119. The uploaded file was called `NSt8bHTg.zip`.

2. With the threat actor and compromised web server IP info known, I filtered for just the HTTP responses from that server to the threat actor with `http.response && ip.src == 172.31.25.119 && ip.dst == 23.158.56.196`. Within them was an XML response which provided build info regarding the server, including the version 2023.11.3.

3. The build info above confirms the web server is running JetBrains TeamCity. Searching for vulnerabilities on this build version yields [CVE-2024-27198](https://nvd.nist.gov/vuln/detail/CVE-2024-27198), an authentication bypass affecting TeamCity On-Premises 2023.11.3 and earlier.

4. Searching for `ip.src == 23.158.56.196 && ip.dst == 172.31.25.119`, I saw the threat actor made a POST to `/hax?jsp=/app/rest/users;.jsp` which contained the credentials for a new user: `c91oyemw:CL5v********`. This request is the authentication bypass in action. `/app/rest/users` normally requires an authenticated administrator, but the trailing `;.jsp` is the whole trick: TeamCity's servlet mapping treats the request as a harmless `.jsp` file and skips the authentication filter, while the `?jsp=` parameter still routes it to the privileged REST endpoint. The attacker created an admin account with no credentials at all.

5. I followed the HTTP stream where the threat actor uploaded the malicious `.zip` and was able to trace their actions through their requests. The first command from the threat actor was `ls` at 2024-06-30 08:03. Shortly thereafter, they ran `whoami`, and the server returned `root`, confirming the web shell was executing with full privileges rather than as a service account.

6. Their requests had the HTML form item `cmd` with the value being their shell commands, making it easy to view them with the filter `urlencoded-form.key == "cmd"` and adding the value as a column. While the threat actor was searching the host, they found a text file with credentials in `/tmp/Creds.txt`, which they modified to be `a1l4m:youa********`.

7. The threat actor's activity of modifying data at rest maps to [T1565.001 Data Manipulation: Stored Data Manipulation](https://attack.mitre.org/techniques/T1565/001/).

8. TeamCity was running inside a container, so root in the web shell is root in the container, not on the underlying host. The last commands within the `.pcap` show the attacker attempting to cross that boundary with `docker run --rm -it -v /:/host ubuntu chroot /host`. This starts a new container with the host's entire root filesystem bind-mounted at `/host`, then chroots into it. Because containers are a namespace boundary rather than a security boundary, and the container's root user could reach the Docker socket, this is a well-known escape: the mounted filesystem is the real host's, so anything written under `/host` is written to the host itself.

## Timeline

- 2024-06-30 08:02:49 UTC: Threat actor probed the server and discovered the TeamCity version.
- 2024-06-30 08:02:49 UTC: A new user, `c91oyemw`, was created via the CVE-2024-27198 authentication bypass.
- 2024-06-30 08:03:06 UTC: `NSt8bHTg.zip` was uploaded to 172[.]31[.]25[.]119 from 23[.]158[.]56[.]196 through `/admin/pluginUpload.html` as a TeamCity plugin.
- 2024-06-30 08:03:08 UTC: Threat actor triggered the plugin and obtained a web shell.
- 2024-06-30 08:03:57 UTC: First command from the web shell, `ls`, followed by `whoami` returning `root`.
- 2024-06-30 08:13:56 UTC: Threat actor modified the credentials stored in `/tmp/Creds.txt`.
- 2024-06-30 08:14 UTC: Threat actor attempted a container escape via `docker run --rm -it -v /:/host ubuntu chroot /host`. The capture ends here.

## Indicators of Compromise

**Network**

- `23[.]158[.]56[.]196` - attacker source IP

**Files**

- `NSt8bHTg.zip` - malicious TeamCity plugin uploaded to deliver the web shell
- `/tmp/Creds.txt` - credential file modified by the attacker

**Accounts**

- `c91oyemw` - TeamCity user created through the authentication bypass

**Endpoints**

- `/hax?jsp=/app/rest/users;.jsp` - CVE-2024-27198 exploit request
- `/admin/pluginUpload.html` - plugin upload abused for web shell delivery

**MITRE ATT&CK**

- T1565.001 - Data Manipulation: Stored Data Manipulation

## Summary of Incident

On June 30, 2024, an attacker at 23[.]158[.]56[.]196 compromised a public-facing JetBrains TeamCity server at 172[.]31[.]25[.]119 by exploiting CVE-2024-27198. The server disclosed its own version, 2023.11.3, in an XML build-info response, and that version falls within the affected range. The vulnerability lets anyone with HTTP access to the server reach authenticated REST endpoints without credentials by appending `;.jsp` to the request path, which causes the authentication filter to be skipped while the request still resolves to the privileged endpoint. The capture shows it used exactly that way: a single unauthenticated POST to `/hax?jsp=/app/rest/users;.jsp` created the administrative account `c91oyemw` less than a second after the version probe.

Seventeen seconds later the attacker used that account to upload `NSt8bHTg.zip` through `/admin/pluginUpload.html`. TeamCity executes uploaded plugins as trusted server-side code, so triggering the plugin returned a web shell running in the TeamCity process context. Commands were passed in a `cmd` field in URL-encoded form data rather than over an encrypted channel, which leaves the entire post-exploitation session readable in the capture. The first command, `ls`, ran at 08:03:57, and `whoami` returned `root` immediately after.

The confirmed impact is data manipulation. The attacker enumerated the filesystem, located `/tmp/Creds.txt`, and rewrote the credentials it contained at 08:13:56, which maps to T1565.001. That root shell was inside the TeamCity container rather than on the host, and the final requests in the capture are an attempt to cross that boundary: `docker run --rm -it -v /:/host ubuntu chroot /host` bind-mounts the host's root filesystem into a new container and chroots into it, which converts container root into host root. The capture ends at that point, so whether the escape succeeded, and whether the attacker reached the underlying host, is not established by this evidence. The host must be treated as compromised until disk-level artifacts say otherwise.

The pcap is the only artifact provided and it does not record how the intrusion was noticed. Nothing in the traffic indicates a preventive control interrupted the attacker at any stage; the sequence from first probe to credential tampering completed in about eleven minutes without interference.

## Remediation

The server should be taken offline and rebuilt from a known-good image rather than cleaned, since compromise is confirmed and the container escape attempt means the underlying host cannot be assumed intact either. Preserve a disk image of the host and the running container state before rebuilding, because the pcap alone cannot answer whether the escape succeeded. TeamCity should not be running as root inside its container, and the container should not have access to the Docker socket; both of those are what made the escape attempt viable and both should be corrected in the rebuilt deployment.

TeamCity should be upgraded to 2023.11.4 or later, which is the release that fixes CVE-2024-27198, and every other TeamCity instance in the organization should be inventoried and patched to the same level. The `c91oyemw` account should be deleted, and every account on the server audited for others created the same way, since the bypass leaves no authentication trail. Any credentials stored on or used by that host must be rotated wherever else they appear in the organization, including the ones in `/tmp/Creds.txt` and any build secrets or deployment keys held by TeamCity, which is the more serious exposure given that a CI server typically holds credentials to production.

Block 23[.]158[.]56[.]196 at the perimeter and search firewall and proxy logs for that address across the estate to determine whether the attacker touched other systems. Because the exploit is a single unauthenticated request, the highest-value detection is a rule on any inbound request whose path contains `;.jsp` or that pairs a `?jsp=` parameter with a semicolon in the path. That signature is specific to this bypass and has no legitimate equivalent. Secondary detections should cover requests to `/app/rest/users` carrying no authentication header, and any upload to `/admin/pluginUpload.html` that does not correspond to a change ticket.

The root cause is an unpatched, internet-exposed CI server. TeamCity should not be reachable from the public internet; it should sit behind a VPN or an identity-aware proxy, with a WAF in front if external access is genuinely required. Plugin upload should be restricted to a named set of administrators, since that feature is arbitrary code execution by design and was the mechanism that turned an authentication bypass into a shell.

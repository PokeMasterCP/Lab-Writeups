# Nexus

## Lab Info

| Lab | Platform | Difficulty | Focus |
| --- | --- | --- | --- |
| Nexus | Hack The Box | Easy | Web enumeration, credential reuse, authenticated file upload, and Git path traversal |

## Scenario

Nexus is an easy-difficulty Linux machine that features an exposed Gitea repository leaking credentials and a job posting that reveals valid usernames.

Tools used:

- Nmap
- Gobuster
- FFUF
- Git
- Gitea API
- Hydra
- Netcat
- LinPEAS

## Enumeration

1. **How many open TCP ports are listening on Nexus? — `2`**

   I ran an Nmap service scan against the original target address:

   ```text
   nmap -sC -sV --open -oA /tmp/nexus_basic_tcp 10[.]129[.]98[.]233

   PORT   STATE SERVICE VERSION
   22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
   80/tcp open  http    nginx 1.24.0 (Ubuntu)
   ```

   The scan identified two open TCP ports: SSH on `22/tcp` and HTTP on `80/tcp`. The web service redirected to `hxxp://nexus[.]htb/`.

2. **What is the hiring manager's full email address? — `j.matthew@nexus[.]htb`**

   I reviewed the careers content and job modal on the port 80 homepage. It identified the hiring manager as `j.matthew@nexus[.]htb`. I treated the related username formats `j.matthew`, `jmatthew`, and `matthew` as potential options to consider.

3. **What additional subdomain hosts the Git service? — `git`**

   I enumerated virtual hosts with Gobuster and filtered the default response. Two additional hosts responded:

   ```text
   billing.nexus[.]htb  Status: 302  -> hxxp://billing.nexus[.]htb/admin/login
   git.nexus[.]htb      Status: 200
   ```

   The page title identified `git.nexus[.]htb` as Gitea. The `billing` host redirected to an administrative login page.

4. **What `DB_PASSWORD` was exposed in the repository? — `N27xh!!2ucY04`**

   On `git.nexus[.]htb`, I inspected the `admin/krayin-docker-setup` repository and its history. Commit `1615c465b74e5d7ad3162873382dd8b3869ca892` contained a `.env` file with these database settings:

   ```text
   DB_CONNECTION=mysql
   DB_HOST=krayin-mysql
   DB_PORT=3306
   DB_DATABASE=krayin
   DB_USERNAME=krayin
   DB_PASSWORD=N27xh!!2ucY04
   ```

   The password had been removed from the current `main` revision but remained recoverable from Git history. One SSH login attempt as `j.matthew` failed, but the credential successfully authenticated `j.matthew@nexus[.]htb` to the billing administration panel. This confirmed password reuse between the historical database configuration and the Krayin administrator account.

5. **What version of Krayin CRM is running on the billing subdomain? — `2.2.0`**

   Enumeration of `billing.nexus[.]htb` identified the application as Krayin CRM version `2.2.0`.

## Initial Access

6. **What CVE allows unrestricted PHP upload and remote code execution? — `CVE-2026-38526`**

   I validated `CVE-2026-38526` through the authenticated `/admin/tinymce/upload` endpoint. A harmless PHP marker uploaded with a declared `image/jpeg` content type was stored at:

   ```text
   /storage/tinymce/1fb7b36f6489dce1c9edcca2bf4cb856.php
   ```

   Requesting the uploaded file returned HTTP 200 and the fixed string `NEXUS_RCE_CONFIRMED`, confirming server-side PHP execution without initially running an operating-system command.

   I then uploaded and triggered a PHP callback payload. The target established a TCP connection to `10[.]10[.]17[.]61:4444`, providing a shell as `www-data`.

   Reference: [NVD — CVE-2026-38526](https://nvd.nist.gov/vuln/detail/CVE-2026-38526)

## User Access

7. **What password for `jones` was discovered during post-exploitation? — `y27xb3ha!!74GbR`**

   From the `www-data` shell, I read `/var/www/krayin/.env`. The live configuration contained a different database password from the historical Git credential:

   ```text
   DB_HOST=127[.]0[.]0[.]1
   DB_PORT=3306
   DB_DATABASE=krayin
   DB_USERNAME=krayin
   DB_PASSWORD=y27xb3ha!!74GbR
   ```

   The password authenticated successfully as `jones`. The same credentials returned HTTP 200 from the Gitea `/api/v1/user` endpoint for account ID `2`, whose email was `j.matthew@nexus[.]htb`. The account could create and push repositories. `jones` had no permitted `sudo` commands, and the recorded group membership revealed no notable privileges.

8. **Submit the user flag.**

   After authenticating as `jones`, I obtained the flag from the user's home directory. Its value is intentionally not recorded in this write-up.

## Privilege Escalation

9. **What systemd timer triggers the template synchronization service? — `gitea-template-sync.timer`**

   Service enumeration identified these units and execution details:

   ```text
   Timer:       gitea-template-sync.timer
   Service:     gitea-template-sync.service
   ExecStart:   /usr/bin/python3 /etc/gitea/template-sync.py
   Service user: root
   Frequency:   approximately once per minute
   ```

10. **Submit the root flag.**

    I ran LinPEAS which initially drew attention to the timer's relative unit reference, but that reference was not the vulnerability. Reviewing `/etc/gitea/template-sync.py` showed that the root-owned service enumerated Gitea template repositories, trusted paths returned by `git ls-tree -r HEAD`, joined them to the repository staging path with `os.path.join()`, and wrote each blob without confirming that the resulting destination remained below `/home/git/template-staging`.

    I created the public template repository `jones/sync-pwn` and pushed crafted commit `6a7ae6df318b89f3af57cfecb88814e4f0049ee1`. Its raw Git tree contained this path:

    ```text
    ../../../../../etc/sudoers.d/jones
    ```

    The blob contained:

    ```text
    jones ALL=(ALL:ALL) NOPASSWD: ALL
    ```

    Five parent components escaped the extraction root at `/home/git/template-staging/jones/sync-pwn`. On the next timer execution, the service wrote `/etc/sudoers.d/jones` as `root`. I then ran `sudo -s`, obtained a root shell, and retrieved the root flag. The flag value is intentionally not recorded.

## Attack Chain

1. Enumerated `nexus[.]htb` and discovered `billing.nexus[.]htb` and `git.nexus[.]htb`.
2. Recovered `N27xh!!2ucY04` from historical Gitea commit `1615c465b74e5d7ad3162873382dd8b3869ca892`.
3. Authenticated to Krayin CRM as `j.matthew@nexus[.]htb`.
4. Exploited `CVE-2026-38526` to upload PHP and obtain a `www-data` shell.
5. Read `/var/www/krayin/.env` and recovered `y27xb3ha!!74GbR`.
6. Reused that password for the local and Gitea user `jones`.
7. Created a Gitea template repository containing a crafted raw Git tree with parent traversal.
8. Allowed the root-owned template synchronization service to write a `sudoers` entry outside its staging directory.
9. Ran `sudo -s` as `jones` and obtained a root shell.

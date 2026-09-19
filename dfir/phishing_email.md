## Lab Info

| Lab | Platform | Difficulty | Focus |
| --- | --- | --- | --- |
| Phishing Email | Hack The Box | Very Easy | Phishing investigation |

## Scenario

Your email address has been leaked and you receive an email from Paypal in German. Try to analyze the suspicious email.

Tools used:

- `Notepad.exe`

## Investigation Actions Taken

1. I opened the suspicious email in `Notepad.exe`. The `Return-Path` header was set to `bounce@rjttznyzjjzydnillquh[.]designclub[.]uk[.]com`. This appears to have been used to pass Sender Policy Framework (SPF).

2. I reviewed the German email body, which roughly translates to:

    ```text
    You are AU PayPal Rewards customer no. 12819202501, and we have been waiting for your confirmation since August 9, 2022.
    This delivery is for you. To activate the delivery...
    ```

    Although the email impersonates PayPal, the confirmation button points to a suspicious link on `storage[.]googleapis[.]com`.

3. The `Received-SPF` header lists the sender's IP as `134[.]195[.]196[.]43`.

## Timeline

- 2022-08-15 14:35:02 UTC: The email was sent to the recipient.

## Indicators of Compromise

**Network**

- `134[.]195[.]196[.]43` - sender IP listed in the `Received-SPF` header
- `storage[.]googleapis[.]com` - domain hosting the suspicious link

## Summary of Incident

On August 15, 2022, at 14:35 UTC, a phishing email impersonating PayPal was sent to the user. The email passed SPF by using a bounce address and directed the user to a link on `storage[.]googleapis[.]com`. The link likely contained a stager or opened a phishing page to attempt credential theft.

## Remediation

Check firewall logs for connections to either indicator of compromise (IOC). Search the email server for other recipients and quarantine matching emails. Contact affected users to confirm whether they opened the link, and provide awareness training to those who did. If further investigation is required, open the link in a sandbox or URL scanner such as `urlscan[.]io` to confirm whether it is a phishing site or malicious code was executed.

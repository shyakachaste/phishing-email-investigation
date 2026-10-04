# Snapped Phish-ing Line: Phishing Campaign Investigation

Simulated phishing investigation completed in a TryHackMe lab environment. All emails, domains and credentials are lab data.

## Overview

In this simulated scenario, several employees at a fictional company (SwiftSpend Financial) received the same suspicious email on one day, and some entered their login details and were locked out of their accounts. My task was to analyze the emails, work out how the attacker operated, and judge how far the attack had spread.

The investigation went further than a normal email check, because the attacker had left files exposed on their own website. I followed the phishing link to the fake login page, recovered the phishing kit used to build it, and found a log file of captured credentials.

## Tools used

- **Thunderbird:** reading the reported emails
- **Firefox (analysis VM):** opening the phishing link and browsing the attacker's site safely
- **VirusTotal:** checking the phishing kit archive
- **sha256sum:** hashing the archive before opening it
- **Basic Linux commands:** extracting and reading the kit's files

## Investigation

### 1. Reviewing the reported emails

The first email went to an employee with the subject "Quote for Services Rendered". It came from a "Group Marketing Online" sender (`Accounts.Payable@groupmarketingonline[.]icu`) with a PDF attached. The same sender address was used across the whole campaign.

![Quote for Services Rendered email](Email.png)
*Figure 1: Phishing email sent from the attacker's address*

### 2. The HTML attachment

A second email in the batch carried an HTML attachment instead of a PDF. Opening the attachment revealed a redirect link to `kennaroads[.]buzz`. Using an HTML file lets the attacker send victims straight to the phishing site without a visible link in the email body.

### 3. Following the redirect

I opened the link inside the VM to see where it led.

![kennaroads.buzz homepage](kennaroads.png)
*Figure 2: The site's home page looks like an ordinary WordPress blog, most likely a legitimate site that was compromised and reused to host the phishing page*

The full redirect led to a fake Microsoft sign-in page, pre-filled with the victim's email address to look convincing.

![Fake Microsoft login page](microsoft.png)
*Figure 3: Fake Microsoft login page asking for the victim's password*

### 4. Exposed directory

Because the page lived on a real website, I checked whether the attacker had left directory browsing enabled. The `/data/` path showed an open directory listing.

![Index of /data directory](kennaroads_data.png)
*Figure 4: The open `/data` directory contained `Update365.zip`, the phishing kit*

### 5. Hashing the phishing kit

I downloaded the archive into the VM and hashed it before opening it.

![sha256sum of the zip file](terminal_1.png)
*Figure 5: SHA256 hash of the archive*

```
ba3c15267393419eb08c7b2652b8b6b39b406ef300ae8a18fee4d16b19ac9686
```

### 6. VirusTotal analysis

Searching the hash on VirusTotal showed 32 of 66 vendors flagging the file as malicious. Besides phishing, it carries a **trojan** category and labels such as `phishmailer`, `phishingms` and `hacktool`. That tells me it is a packaged phishing kit that security vendors treat as malware, not just a static fake page.

![VirusTotal detection for the zip](virustotal_hash.png)
*Figure 6: VirusTotal detections for the archive*

![VirusTotal bundle details](virustotal_folder.png)
*Figure 7: The archive contains 49 files*

### 7. Exposed credential log

The open `/data` directory also contained a log file with captured credentials in plain text, each with an IP address and timestamp.

![Log file with captured credentials](log.png)
*Figure 8: Log of captured credentials (lab data)*

One employee account appeared more than once, meaning that user submitted credentials on the fake form multiple times.

### 8. Analyzing the kit: where credentials go

I extracted the archive in the VM and looked through its structure for the script that handles submitted credentials.

![Extracted kit folder structure](terminal_2.png)
*Figure 9: Extracted kit, with the phishing pages and scripts in the `Validation` folder*

The `submit.php` script builds a message containing the victim's email, password, IP address, browser and country, then emails it to a hard-coded address (`m3npat@yandex[.]com`). This means credentials are exfiltrated even if the server log is later deleted.

![submit.php contents](terminal_3.png)
*Figure 10: `submit.php` collecting the details and sending them by email*

## Indicators of compromise (IOCs)

Domains and emails are defanged with `[.]` so they cannot be clicked by accident.

| Indicator | Type | Context |
|---|---|---|
| `Accounts.Payable@groupmarketingonline[.]icu` | Sender email | Used for the whole campaign |
| `kennaroads[.]buzz` | Domain | Compromised site hosting the redirect, fake login and kit |
| `kennaroads[.]buzz/data/` | URL path | Exposed directory with the kit and credential log |
| `Update365.zip` | File name | Phishing kit archive (49 files) |
| `ba3c15267393419eb08c7b2652b8b6b39b406ef300ae8a18fee4d16b19ac9686` | SHA256 | Phishing kit, 32/66 vendors flag it |
| `m3npat@yandex[.]com` | Email | Collection address configured in `submit.php` |

## Impact

At least one employee account submitted credentials more than once, and all accounts listed in the exposed log should be treated as compromised.

## Conclusion

This was a larger and better organized campaign than a single phishing email. The attacker used one sender address, delivered victims to a fake Microsoft login through PDF and HTML attachments, and hosted the page on what looks like a compromised legitimate website. Because the attacker left a directory open, I could recover the phishing kit, read the captured credentials, and identify the address collecting them.

## Recommendations

- Force a password reset for every account in the exposed log, and check those accounts for suspicious activity such as new mailbox rules, forwarding, or sign-ins from unusual locations.
- Block the sender address and the `kennaroads[.]buzz` domain at the email gateway and web proxy.
- Report the compromised site to its host so the kit and log file can be removed.
- Require multi-factor authentication on all accounts so a stolen password alone is not enough.
- Keep training staff to check senders and links before entering credentials, and to report suspicious emails quickly.

## What I learned

- How a phishing kit is put together, and how scripts like `submit.php` collect and forward stolen credentials.
- That attackers sometimes host phishing pages on compromised legitimate websites instead of new domains.
- How exposed directories and logs can give extra evidence during an investigation.
- How to check an archive with VirusTotal and read its threat categories and file count.
- How to write up findings as indicators of compromise with clear recommendations.

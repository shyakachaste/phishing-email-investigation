# Greenholt Phish: Phishing Email Investigation

Simulated phishing email investigation completed in a TryHackMe lab environment. All emails, domains and addresses are lab data.

## Overview

In this simulated scenario, a sales employee at a fictional company (Greenholt PLC) flagged an email that appeared to come from a known customer. It had a generic greeting, mentioned a money transfer the employee was not expecting, and carried an attachment. The employee said the tone did not match how this customer normally writes, so the message was sent to the security team for review.

My task was to work out who really sent the email, whether the sender could be trusted, and whether the attachment was safe. I concluded it is a phishing email with a malicious attachment.

## Tools used

- **Thunderbird:** reading the email and viewing its source (headers)
- **Cisco Talos Intelligence:** looking up the sender IP address and its owner
- **dmarcian SPF Surveyor:** checking the SPF record of the sender domain
- **dmarcian Domain Checker:** checking the DMARC record of the sender domain
- **sha256sum (Linux):** hashing the attachment
- **VirusTotal:** checking the hash and finding the real file type

## Investigation

### 1. Reading the email

I started with what a normal user would see. The subject contains a transfer reference number, and the body says funds were sent "via SWIFT" that morning. It gives payment details of about 149,650 USD, says a receipt is attached, and is signed by "Mr. James Jackson" from Accounts Payable at SEC Marine Services PTE LTD.

![Top part of the email](email_1.png)
*Figure 1: Top part of the email*

![Bottom part of the email](email_2.png)
*Figure 2: Bottom part of the email with the payment details and signature*

Things that looked wrong:

- The greeting uses my email address instead of a name, and the same address is repeated in the subject line.
- "As instructed" suggests I asked for this payment, but nothing was requested. It pressures the reader to open the receipt.
- The grammar is poor for an accounts department ("funds has been transferred").
- The signature names SEC Marine Services, but the sending domain is different.
- The attachment is named `SWT_#09674321____PDF__.CAB`. It is meant to look like a PDF, but the extension is .CAB.

### 2. Sender details from the headers

Next I opened the email source and read the headers.

| Item | Value |
|---|---|
| Display name | Mr. James Jackson |
| Sender address | `info@mutawamarine[.]com` |
| Reply-To address | `info.mutawamarine@mail[.]com` |
| Originating IP | `192[.]119[.]71[.]157` |

![Email source with headers](email_sourcecode.png)
*Figure 3: Email source showing the Received headers, Reply-To, and the originating IP (highlighted)*

The most important finding is that the Reply-To differs from the sender. The email appears to come from one domain, but a reply would go to a free mail.com address. This is a common phishing trick, because the attacker still receives the reply even if the sender address is fake or gets blocked.

I also noticed the receiving server's spam filter marked the email as "not spam" with a score of -0.5, where 5.0 would mark it as spam. The filter missed it, which is why the employee's report mattered.

### 3. Who owns the sending IP?

I searched the originating IP on [Cisco Talos Intelligence](https://talosintelligence.com/).

![Cisco Talos lookup](cisco_talos.png)
*Figure 4: Cisco Talos lookup for the originating IP*

Talos shows the IP is in Dallas, United States, owned by **HostPapa**, a web hosting company. Its reputation is "Neutral", it has no email volume history, and it is not on the common block lists (SpamCop, CBL, PBL).

This means the IP is not known to be bad yet, but that does not prove the email is safe. A legitimate company would normally send mail from its own servers or a large mail provider, not from a small hosting server with no email history.

### 4. SPF record check

SPF is a DNS record listing which servers may send email for a domain. I checked the sender domain with the [dmarcian SPF Surveyor](https://dmarcian.com/spf-survey/).

![SPF record](spf_record.png)
*Figure 5: SPF record of the sender domain*

```
v=spf1 include:spf.protection.outlook.com -all
```

This record allows only Microsoft 365 servers to send mail for the domain, and `-all` means every other server should be rejected. The email came from a HostPapa IP, not Microsoft, so it does not match the domain's own SPF policy. That suggests the sender was not the real domain owner.

*Note: the headers I looked at did not show the SPF result from the receiving server, so this conclusion comes from comparing the SPF record with the sending IP.*

### 5. DMARC record check

DMARC tells receiving servers what to do with emails that fail SPF or DKIM. I checked it with the [dmarcian Domain Checker](https://dmarcian.com/domain-checker/).

![DMARC record](dmarc.png)
*Figure 6: DMARC record of the sender domain*

```
v=DMARC1; p=quarantine; fo=1
```

The domain has a valid DMARC record with a quarantine policy, meaning failing emails should go to spam. dmarcian recommends `p=reject` for the strongest protection. Here the email still reached the inbox, so the protection did not stop it.

### 6. The attachment

The attachment is `SWT_#09674321____PDF__.CAB`, about 400 KB. I saved it in the lab machine and ran `sha256sum` on it without opening it.

![SHA256 hash of the attachment](hashing.png)
*Figure 7: SHA256 hash of the attachment*

```
2e91c533615a9bb8929ac4bb76707b2444597ce063d84a4b33525e25074fff3f
```

### 7. Checking the hash on VirusTotal

I searched the hash on [VirusTotal](https://www.virustotal.com/).

![VirusTotal detection result](virustotal_1.png)
*Figure 8: VirusTotal detection result*

![VirusTotal file details](virustotal_2.png)
*Figure 9: VirusTotal file details*

48 of 64 security vendors flagged the file as malicious, and it is tagged "rar", "spreader" and "attachment". The name says PDF and the extension says CAB, but the real file type is a **RAR archive**. The disguise is meant to make it look like a normal receipt and slip past simple filters.

## Indicators of compromise (IOCs)

Domains, emails and IPs are defanged with `[.]` so they cannot be clicked by accident.

| Indicator | Type | Context |
|---|---|---|
| `info@mutawamarine[.]com` | Sender email | Display name "Mr. James Jackson" |
| `info.mutawamarine@mail[.]com` | Reply-To email | Different from the sender, replies go to the attacker |
| `192[.]119[.]71[.]157` | IP address | Originating IP, owned by HostPapa, not allowed by the domain's SPF |
| `SWT_#09674321____PDF__.CAB` | File name | Disguised as a PDF, real type is RAR |
| `2e91c533615a9bb8929ac4bb76707b2444597ce063d84a4b33525e25074fff3f` | SHA256 | Attachment, 400.26 KB, flagged by 48/64 vendors |
| `09674321` | Subject reference | Transfer reference number used as lure |

## Conclusion

This is a phishing email, for these reasons:

- It uses a fake payment notice to push the reader to open an attachment.
- The Reply-To address points to a different domain than the sender.
- The sending IP is a web hosting server that does not match the domain's SPF record, which only allows Microsoft servers.
- The signature company name does not match the sending domain.
- The attachment pretends to be a PDF, uses a .CAB extension, and is really a RAR archive that 48 of 64 vendors detect as malicious.

## Recommendations

- Block the sender address, the Reply-To address and the IP address, and delete other copies of this email from mailboxes.
- Search email and endpoint logs for the attachment hash to see who else received or opened it.
- If anyone opened the attachment, isolate that computer and scan it.
- Block or scan archive attachments (RAR, CAB, ZIP) from outside senders, and check the real file type instead of trusting the extension.
- Keep encouraging staff to report emails like this, and remind them to confirm payment requests by calling the sender on a known number.

## What I learned

- How to read email headers and find the real origin of a message.
- How SPF and DMARC work, and how to compare them with the sending server.
- How to use Cisco Talos, dmarcian and VirusTotal to research an IP, a domain and a file.
- That file names and extensions cannot be trusted, so the hash and real file type must be checked.

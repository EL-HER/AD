<div align="center">

# 🛡️ AD-Security-Lab

### Active Directory & Network Defense — Personal Study Journal

*A structured, session-by-session deep dive into Windows authentication,*
*Kerberos internals, and real-world Active Directory attack chains.*

---

[![Status](https://img.shields.io/badge/Status-In%20Progress-blue?style=flat-square)]()
[![Sessions](https://img.shields.io/badge/Sessions%20Completed-1-green?style=flat-square)]()
[![Source](https://img.shields.io/badge/Source-SANS%20SEC560-red?style=flat-square)]()
[![Focus](https://img.shields.io/badge/Focus-Offense%20%26%20Defense-orange?style=flat-square)]()

</div>

---

## 📖 About This Repository

This repo documents my journey learning Active Directory security from the ground up — covering architecture, authentication protocols, and the attack techniques that target them.

Every session includes:
- **Core concepts** explained in plain language
- **Step-by-step mechanisms** (how things actually work under the hood)
- **Defensive strategies** mapped directly to each attack
- **Key Event IDs** to monitor in a real SOC environment

> **Study sources:** SANS SEC560 Enterprise Penetration Testing material

---

## 📚 Table of Contents

- [Session 1 — AD Fundamentals & Authentication](#-session-1--ad-fundamentals--authentication)
  - [Part A: Active Directory Architecture](#part-a-active-directory-architecture)
  - [Part B: Windows Hashing — LM, NT, NTLM](#part-b-windows-hashing--lm-nt-ntlm)
  - [Part C: NTLMv2 Challenge/Response Protocol](#part-c-ntlmv2-challengeresponse-protocol)
  - [Part D: Kerberos Authentication Deep Dive](#part-d-kerberos-authentication-deep-dive)
  - [Part E: AD Attacks — Kerberoast, DCSync, DCShadow](#part-e-ad-attacks--kerberoast-dcsync-dcshadow)
- [Attack & Defense Summary Table](#-attack--defense-summary-table)
- [Key Event IDs Cheatsheet](#-key-event-ids-cheatsheet)
- [Future Sessions](#-future-sessions)
- [Resources](#-resources)

---

## 🔐 Session 1 — AD Fundamentals & Authentication

---

### Part A: Active Directory Architecture

Active Directory is Microsoft's centralised identity and access management solution. Every Windows enterprise network runs on it.

**The core idea in one line:**
> AD lets administrators control every user, device, and permission in the network from a single server — the **Domain Controller (DC)**.

```
                        [ Main Server ]
                              │
                    ┌─────────┴─────────┐
               [ bank.local ]      [ other domains ]
                    │
          ┌─────────┴──────────┐
    [ egypt.bank.local ]   [ saudi.bank.local ]
          │
   ┌──────┴──────┐
[ cairo.egypt ]  [ alex.egypt ]
```

**Key components:**

| Component | What It Is | Why It Matters |
|-----------|-----------|----------------|
| **Domain Controller (DC)** | The central server. Must have a static IP. | All auth flows through it. Compromise = game over. |
| **NTDS.dit** | AD's credential database. Lives only on the DC. | Contains all password hashes in the domain. |
| **Domain** | Logical boundary (e.g. `bank.local`) | All devices inside share the same auth rules. |
| **OU (Organisational Unit)** | Logical container (e.g. IT, HR, PenTest) | Used to apply Group Policy granularly per department. |
| **Group Policy** | Rules pushed from DC to all joined devices | Enforce settings, disable features, control access. |
| **Trust Relationship** | Auth link between domains | Child inherits trust from parent automatically. |

**What AD manages:**
- `Users` — accounts + passwords
- `Computers` — any device that joins the domain
- `Groups` — collections of users sharing permissions
- `OUs` — containers for applying Group Policy at scale

> 🛡️ **Defensive note:** The DC is the single highest-value target in any Windows environment. Protecting it, monitoring its logs, and controlling who can replicate from it are Tier-0 security controls.

---

### Part B: Windows Hashing — LM, NT, NTLM

Windows never stores plaintext passwords. It stores **hashes** — one-way mathematical transformations.

#### How hashing works

```
Password ──► [ Hash Algorithm ] ──► Hash Value
  "Pass123"        (MD4)           "a3f8b2..."

• One-way: you cannot reverse a hash back to a password
• Used for: integrity checks, password verification
• If hashes match → authenticated
```

#### The three hash types

**① LM Hash** *(legacy — Windows XP / Server 2003 era)*

```
Step 1:  "password"  ──►  "PASSWORD"          (force uppercase)
Step 2:  "PASSWORD"  ──►  "PASSWORDOOOOOOO"   (pad to 14 chars)
Step 3:  Split into two 7-char chunks:
         Chunk A: "PASSWOR"
         Chunk B: "DOOOOOOO"
Step 4:  Each chunk → ASCII → Binary → 8 bytes
Step 5:  Each 8-byte block encrypted with DES
         Key used: "KGS!@#$%"  (hardcoded by Microsoft)
Step 6:  Concatenate both DES outputs → LM Hash stored in SAM
```


**② NT Hash** *(current standard)*

```
Password ──► UTF-16LE encoding ──► MD4 algorithm ──► NT Hash

Example:
  "Password" ──► MD4(UTF-16LE("Password")) ──► 8846F7EAEE8FB117...
```

> No padding, no splitting, case-sensitive. Still has **no salt** — identical passwords produce identical hashes across all machines.

**③ NTLM Hash** — same as NT hash, different naming context.

#### Where hashes are stored

| Location | Used By | Contents |
|----------|---------|----------|
| `SAM` file | Local machines (workstations) | NT hashes of local accounts |
| `NTDS.dit` | Domain Controller only | NT hashes of ALL domain accounts |

> 🛡️ **Defensive actions:**
> - Disable LM hash storage via Group Policy (`NoLMHash` = 1)
> - Enforce NTLMv2 minimum — disable LM and NTLMv1 responses
> - Alert on any non-DC process touching `NTDS.dit`
> - Offensive tools (Hashcat, John the Ripper) crack NT hashes offline — strong password policy is essential

---

### Part C: NTLMv2 Challenge/Response Protocol

When a user authenticates to a **remote** service (not locally), Windows uses NTLMv2.

#### Local authentication (PC login)

```
User types password
       │
       ▼
PC hashes it (MD4 → NT Hash)
       │
       ▼
Compare with NT hash stored in SAM file
       │
   ┌───┴───┐
   ✓ Match   ✗ No match
   │            │
Authenticated  Rejected
```

#### NTLMv2 remote authentication (7-step flow)

```
  CLIENT                    APP SERVER              DOMAIN CONTROLLER
    │                           │                           │
    │──① username ─────────────►│                           │
    │                           │                           │
    │◄─② Nonce (challenge) ─────│                           │
    │                           │                           │
    │  Encrypt Nonce             │                           │
    │  with NT hash             │                           │
    │──③ Encrypted Nonce ──────►│                           │
    │   (Response)              │                           │
    │                           │──④ username + Nonce ─────►│
    │                           │    + Response             │
    │                           │                           │ Lookup NT hash
    │                           │                           │ from NTDS.dit
    │                           │                           │ Re-encrypt Nonce
    │                           │                           │ Compare values
    │                           │◄─⑤ Approved / Rejected ──│
    │                           │                           │
    │◄─⑥ Access Granted/Denied ─│                           │
```

> 🛡️ **Defensive actions:**
> - Enforce **SMB Signing** and **LDAP Signing** — prevents relay attacks
> - Kerberos should be the default; NTLM is a fallback — investigate why NTLM is being used in modern environments
> - NTLM relay is still a primary attack vector in penetration tests

---

### Part D: Kerberos Authentication Deep Dive

Kerberos is the **primary** authentication protocol in Active Directory. It is ticket-based and stateless — all session state lives inside encrypted tickets.

> 💡 **The movie theatre analogy:**
> Think of it like an all-you-can-watch movie pass. You show your ID → get a pass (TGT) → use the pass to get a ticket for a specific movie (Service Ticket) → show the ticket at the door (AP-REQ). The ticket booth doesn't check whether you deserve to watch the movie — that's the door's job.

#### The three actors

| Actor | Role | Technical Name |
|-------|------|---------------|
| You (client) | Wants access to a service | Client |
| The ticket booth | Issues and validates tickets | KDC / Domain Controller |
| The cinema door | Decides if you get in | Application Server / Service |

#### Key definitions

| Term | Definition |
|------|-----------|
| **KDC** | Key Distribution Center — logical role on the DC. Contains the AS and TGS. |
| **AS** | Authentication Server — handles initial login (AS-REQ / AS-REP) |
| **TGS** | Ticket Granting Service — issues service tickets (TGS-REQ / TGS-REP) |
| **TGT** | Ticket Granting Ticket — your "movie pass". Encrypted with krbtgt hash. Valid 10h. |
| **ST / TGS** | Service Ticket — your "movie ticket". Encrypted with the target service's hash. |
| **PAC** | Privilege Attribute Certificate — inside every ticket. Contains User SID, Group SIDs, privileges. |
| **SPN** | Service Principal Name — unique identifier for a service. Format: `service/hostname` |
| **krbtgt** | The KDC's own account. Its hash encrypts ALL TGTs. Never expires unless manually rotated. |

#### The complete Kerberos flow

```
  CLIENT                              KDC / DC
    │                                    │
    │──① AS-REQ ────────────────────────►│
    │   Username                         │
    │   Timestamp encrypted with         │
    │   NT Hash (proves identity)        │ Decrypt timestamp
    │                                    │ Validate it's fresh
    │◄─② AS-REP ─────────────────────────│
    │   TGT (encrypted with krbtgt hash) │
    │   Session Key (encrypted with      │
    │   user's NT hash)                  │
    │                                    │
    │──③ TGS-REQ ───────────────────────►│
    │   TGT                              │
    │   Target SPN                       │ Decrypt TGT with krbtgt hash
    │   Authenticator (username +        │ Verify session key
    │   timestamp encrypted w/ session   │ Build Service Ticket
    │   key)                             │
    │                                    │
    │◄─④ TGS-REP ─────────────────────────│
    │   Service Ticket                   │
    │   (encrypted with service's        │
    │   NT hash — client can't read it)  │
    │                                    │
    │                           APPLICATION SERVER
    │                                    │
    │──⑤ AP-REQ ────────────────────────►│
    │   Service Ticket                   │ Decrypt with own hash
    │   Authenticator                    │ Read PAC
    │                                    │ Check permissions
    │◄─⑥ AP-REP (optional mutual auth) ──│
```

#### The three long-term keys

| Key Owner | Hash Used | Purpose |
|-----------|-----------|---------|
| **KDC** | `krbtgt` NT hash | Encrypt TGT (AS-REP) · Sign PAC |
| **Client** | User's NT hash | Verify AS-REQ timestamp · Decrypt session key |
| **Service** | Service account NT hash | Encrypt Service Ticket · Sign PAC in TGS-REP |

#### What's inside each ticket

**TGT contents** *(only KDC can read this)*
```
┌────────────────────────────────┐
│ Username                       │
│ Session Key                    │
│ TGT Expiry Time                │
│ User Nonce                     │
│ PAC (signed with krbtgt hash)  │
│   └─ User SID                  │
│   └─ Group SIDs                │
│   └─ Privilege flags           │
└────────────────────────────────┘
  Encrypted with: krbtgt NT hash
```

**Service Ticket contents** *(only the target service can read this)*
```
┌────────────────────────────────┐
│ Username                       │
│ Service Session Key            │
│ TGS Expiry Time                │
│ User Nonce                     │
│ PAC (signed with krbtgt hash)  │
│   └─ User SID                  │
│   └─ Group SIDs                │
└────────────────────────────────┘
  Encrypted with: Service account NT hash
```

> 🛡️ **Critical insight:** The KDC does NOT decide if you have access to the target service. It issues the ticket to any authenticated user. Access control is entirely the service's responsibility. This design choice is what Kerberoasting exploits.

---

### Part E: AD Attacks — Kerberoast, DCSync, DCShadow

---

#### ① Kerberoasting

**What it is:** An attack where any authenticated domain user requests service tickets for accounts that have SPNs, then cracks them offline.

**Why it works:**
- The KDC issues service tickets to ANY authenticated user — no permissions check
- Service Tickets are encrypted with the service account's NT hash
- Cracking happens offline — no lockouts, no time pressure
- Service account passwords are rarely rotated (often years old)
- Computer account hashes are random (uncrackable). User account hashes may be weak (crackable)

**Attack chain:**
```
Step 1: Query AD for user accounts with SPNs
        └─ Get-ADUser -Filter {ServicePrincipalName -like "*"}

Step 2: Request RC4 service tickets for those SPNs
        └─ impacket/GetUserSPNs.py  OR  Invoke-Kerberoast

Step 3: Extract tickets from memory / save to file

Step 4: Offline crack with Hashcat or John the Ripper
        └─ hashcat -m 13100 tickets.txt wordlist.txt
```

**Defences:**

| Control | What It Does |
|---------|-------------|
| Group Managed Service Accounts (gMSA) | Auto-rotating 120-char passwords — mathematically uncrackable |
| Enforce AES-256 on service accounts | Eliminates RC4 downgrade — AES tickets are much harder to crack |
| SPN Audit | `Get-ADUser -Filter {ServicePrincipalName -like "*"} -Properties SPN` |
| Alert on Event 4769 + Encryption Type 0x17 | RC4 service ticket request = Kerberoasting indicator |
| Minimise service account privileges | Least privilege — even if cracked, blast radius is limited |

---

#### ② DCSync

**What it is:** An attacker impersonates a Domain Controller using the AD replication protocol (DRSR) and pulls all password hashes from a real DC — without ever touching the DC itself.

**Why it works:**
- For availability, AD has multiple DCs that sync via replication
- Mimikatz implements the DRSR protocol to mimic a DC
- Requires Domain Admin rights or explicit replication permissions

**Attack chain:**
```
Attacker machine (with Mimikatz)
        │
        │  DRSR protocol (mimics DC-to-DC replication)
        ▼
Domain Controller
        │
        │  Returns NT hashes for requested accounts
        ▼
Attacker dumps: krbtgt, Administrator, all users

Command: lsadump::dcsync /user:krbtgt
         lsadump::dcsync /all           ← dumps everything
```

**Impact:** Equivalent to physically copying NTDS.dit. Once `krbtgt` hash is obtained → **Golden Ticket** attack → full domain persistence.

**Defences:**

| Control | What It Does |
|---------|-------------|
| Monitor DRSR traffic | Replication traffic from workstation to DC = anomalous. Alert immediately. |
| Audit replication permissions | Remove `DS-Replication-Get-Changes-All` from all non-DC accounts |
| Monitor Event 4662 | Replication GUIDs in event log signal DCSync activity |
| Rotate krbtgt **twice** | Invalidates all existing Golden Tickets (rotation alone isn't enough) |

---

#### ③ DCShadow

**What it is:** The reverse of DCSync — instead of pulling data out, the attacker **pushes malicious changes in**. Temporarily registers a workstation as a fake DC, then forces the real DC to replicate the attacker's changes.

```
Attacker's machine
        │
        │ Step 1: Register machine as DC in AD schema (Mimikatz)
        │ Step 2: Craft malicious change (e.g. swap krbtgt hash, add to Domain Admins)
        │ Step 3: Trigger replication → real DC commits the change
        ▼
Legitimate DC now contains the attacker's modification

⚠️  No Windows event logs generated — source DC doesn't exist
```

**vs DCSync:**
```
DCSync:    Attacker ◄── pulls data out ── DC
DCShadow:  Attacker ──── pushes changes in ──► DC
```

**Defences:**

| Control | What It Does |
|---------|-------------|
| Monitor AD schema for new DC registrations | Fake DCs must register themselves — this is detectable |
| Network detection | DRSR from a workstation IP = immediate alert |
| Canary accounts | Sensitive accounts with no legitimate use — any modification triggers alert |

---

## 📊 Attack & Defense Summary Table

| Attack | Prerequisite | What It Gets | Stealth Level | Primary Defence |
|--------|-------------|-------------|---------------|----------------|
| Kerberoasting | Domain user | Service account password | High (no noise) | gMSA + AES enforcement |
| DCSync | Domain Admin | All NT hashes in domain | Medium | Monitor DRSR traffic |
| DCShadow | Domain Admin | Persistent AD modification | Very High (no logs) | Schema monitoring |
| Pass-the-Hash | Local Admin + NT hash | Lateral movement | High | Credential Guard |
| Golden Ticket | krbtgt hash | Unlimited domain access | Very High | Rotate krbtgt twice |

---

## 🔍 Key Event IDs Cheatsheet

| Event ID | What It Signals | Attack Indicator |
|----------|----------------|-----------------|
| **4768** | TGT requested (AS-REQ) | Pre-auth disabled = AS-REP Roasting target |
| **4769** | Service ticket requested | + Encryption 0x17 (RC4) = Kerberoasting |
| **4770** | Service ticket renewed | — |
| **4624** | Successful logon | Baseline normal traffic |
| **4662** | Object operation on AD | + Replication GUIDs = DCSync |
| **4776** | NTLM authentication attempt | High volume = brute force or relay attack |

---

## 🗓️ Future Sessions

| Session | Planned Topic | Status |
|---------|--------------|--------|
| Session 2 | AS-REP Roasting & Pre-Auth Bypass | 🔲 Planned |
| Session 3 | Golden Ticket & Silver Ticket Attacks | 🔲 Planned |
| Session 4 | BloodHound — AD Attack Path Enumeration | 🔲 Planned |
| Session 5 | Lateral Movement — Pass-the-Hash, Pass-the-Ticket | 🔲 Planned |
| Session 6 | AD Hardening — Tiered Administration Model | 🔲 Planned |

---

## 📎 Resources

| Resource | Description |
|----------|------------|
| [Microsoft Learn — Kerberos Overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) | Official Kerberos flow and ticket structure |
| [SpecterOps — Kerberoasting Revisited](https://posts.specterops.io/kerberoasting-revisited-d434351bd4d1) | Deep dive on RC4 vs AES and detection evasion |
| [dcshadow.com](https://www.dcshadow.com) | Official DCShadow research site |

---

<div align="center">


</div>

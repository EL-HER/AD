---
icon: cloud-check
---

# DCSync & DCShadow

### Overview

DCSync and DCShadow are two related but opposite attacks against Active Directory replication:

```
DCSync:    pulls credential data OUT of the domain
DCShadow:  pushes malicious changes INTO the domain

Both require Domain Admin rights.
Both abuse the AD replication protocol.
They have opposite directions and different goals.
```

***

## DCSync

### What It Is

DCSync is an attack where an attacker impersonates a Domain Controller and uses the AD replication protocol to request credential data from a legitimate DC — extracting NT hashes for any or all accounts without ever touching the NTDS.dit file directly.

***

### The Replication Protocol It Abuses

Active Directory maintains multiple DCs for redundancy. When a user changes their password on DC-1, that change must propagate to DC-2. This synchronisation uses a protocol called **DRSR — Directory Replication Service Remote**.

```
Normal operation:
  DC-1 ──── DRSR replication ────► DC-2
  DC-2 ──── DRSR replication ────► DC-1

DCSync operation:
  Attacker's machine
  (running Mimikatz)  ──── DRSR request ────► DC
                      ◄─── credentials ────────

The DC cannot distinguish a real DC from Mimikatz
as long as the requesting account has replication permissions.
```

***

### Required Permissions

```
DS-Replication-Get-Changes
DS-Replication-Get-Changes-All

These permissions are granted to:
  → Domain Admins (by default)
  → Enterprise Admins (by default)
  → SYSTEM account on DCs
  → Any account explicitly delegated these rights

An attacker needs one of the above.
Most commonly obtained via: credential theft, Kerberoasting,
or leveraging an existing DA session.
```

***

### The Attack

```
Prerequisites: Domain Admin account or replication permission delegation

Mimikatz commands:
  lsadump::dcsync /user:krbtgt
    → Gets the krbtgt hash → enables Golden Ticket

  lsadump::dcsync /user:administrator
    → Gets the Domain Admin hash

  lsadump::dcsync /all /csv
    → Dumps NT hashes for every account in the domain

What the output contains:
  Hash NTLM (current password hash)
  Hash history (previous password hashes)
  Kerberos keys (AES128, AES256, DES)
```

***

### What Attackers Do With It

```
Scenario 1 — Immediate access:
  krbtgt hash obtained
  → Craft a Golden Ticket (forged TGT)
  → Authenticate as any user, to any service
  → Full domain persistence

Scenario 2 — Credential inventory:
  All NT hashes dumped
  → Crack offline to recover plaintext passwords
  → Password reuse across services, VPNs, cloud accounts
  → Long-term persistent access

Scenario 3 — Pass-the-hash:
  Admin NT hashes obtained
  → Authenticate directly using the hash
  → No need to crack — hash IS the credential in Windows
```

***

### Detection

```
DCSync leaves a network trace:
  DRSR traffic should only flow between Domain Controllers
  If DRSR is observed from a workstation IP → DCSync attack

Windows Event ID 4662:
  Object Type: domainDNS
  Properties contains one of these GUIDs:
    1131f6aa-9c07-11d1-f79f-00c04fc2dcd2  ← Replication-Get-Changes-All
    1131f6ad-9c07-11d1-f79f-00c04fc2dcd2  ← DS-Replication-Synchronize
  Subject: account that is NOT a DC computer account

SIEM rule:
  Event 4662
  + ObjectType = "domainDNS"
  + Properties contains "1131f6aa..."
  + SubjectUserName not ending in "$"  ← not a machine account
  = DCSync confirmed
```

***

### Defence

| Control                                                 | Why It Matters                                                              |
| ------------------------------------------------------- | --------------------------------------------------------------------------- |
| Monitor DRSR traffic                                    | Network-level detection — workstation to DC replication is always anomalous |
| Audit replication permissions quarterly                 | Remove DS-Replication-Get-Changes-All from all non-DC accounts              |
| Alert on Event 4662 with replication GUIDs              | Windows-level detection                                                     |
| Rotate krbtgt password twice after confirmed compromise | One rotation leaves existing Golden Tickets valid                           |
| Tiered administration                                   | Reducing DA credential exposure reduces DCSync opportunity                  |

***

***

## DCShadow

### What It Is

DCShadow is the reverse of DCSync. Instead of pulling data out of the domain, it pushes malicious changes in. The attacker temporarily registers their machine as a Domain Controller in AD, crafts a fraudulent change, then forces a legitimate DC to replicate and commit that change.

Introduced at BlackHat USA 2018 by Benjamin Delpy (author of Mimikatz) and Vincent Le Toux.

***

### Why It Is Uniquely Dangerous

```
Standard Windows security generates event logs on the SOURCE
of a change — the DC that initiated it.

In DCShadow:
  The source DC is the attacker's machine.
  It is temporarily registered as a DC.
  It is then deregistered after the attack.
  It never generated legitimate Windows event logs.

Result:
  → Malicious change is committed to the real DC
  → No Windows event logs exist for this change
  → Standard SIEM detection based on Windows logs misses it entirely
```

***

### The Four Steps

```
Step 1 — Register fake DC
  Mimikatz temporarily adds attacker's machine to AD
  as a DC by modifying specific AD schema attributes.
  This requires Domain Admin rights.

Step 2 — Craft the malicious change
  Examples:
    • Change the NT hash of a privileged account
      (password reset without logging in to the account)
    • Add a user to the Domain Admins group
    • Modify SIDHistory attribute to include privileged SIDs
      (grants invisible admin rights to a low-privilege account)
    • Add a new backdoor account

Step 3 — Trigger replication
  Attacker forces the legitimate DC to pull changes
  from the fake DC.
  The real DC receives and commits the changes
  as if they came from a peer DC.

Step 4 — Deregister fake DC
  Attacker's machine removes itself from the DC list.
  The change is now permanently in AD.
  The fake DC no longer exists.
```

***

### Mimikatz Commands

```
Two terminal windows required — one for each role:

Window 1 (pushes the change):
  lsadump::dcshadow /object:targetuser /attribute:unicodePwd /value:NewPassword

Window 2 (triggers replication):
  lsadump::dcshadow /push
```

***

### DCSync vs DCShadow — Direct Comparison

| Property           | DCSync                        | DCShadow                                        |
| ------------------ | ----------------------------- | ----------------------------------------------- |
| Direction          | Pulls data OUT                | Pushes changes IN                               |
| Goal               | Credential theft              | Persistence / privilege escalation              |
| What you get       | NT hashes, Kerberos keys      | Modified AD objects                             |
| Prerequisites      | DA rights                     | DA rights                                       |
| Windows event logs | Yes — Event 4662 (detectable) | No — source DC is fake                          |
| Network indicator  | DRSR from workstation         | DRSR from workstation                           |
| Stealth level      | Medium                        | Very high                                       |
| Recovery           | Rotate krbtgt twice           | Restore AD from clean backup or reverse changes |

***

### Detection

Because Windows event logs are not generated on the fake DC, detection must come from other sources:

```
Network detection (most reliable):
  DRSR traffic from a non-DC IP address
  → Any IP that is not a known DC generating DRSR = investigate immediately

AD schema monitoring:
  Track changes to nTDSDSA objects (DC registrations)
  Unexpected new DC object appearing and disappearing rapidly

Third-party tools:
  Microsoft Defender for Identity (formerly ATA)
  monitors for DCShadow-specific replication anomalies

Baseline comparison:
  Regular snapshots of AD object attributes
  Diff comparison catches unexplained attribute changes
  Especially: adminCount, member, unicodePwd, SIDHistory
```

***

### Defence

| Control                                | Why It Matters                                                           |
| -------------------------------------- | ------------------------------------------------------------------------ |
| Monitor AD schema for DC registrations | Fake DCs must register — this is detectable                              |
| Network-level DRSR monitoring          | Most reliable detection signal                                           |
| Microsoft Defender for Identity        | Specifically designed to detect DCShadow                                 |
| Regular AD attribute snapshots         | Catches changes that left no event log                                   |
| Canary accounts                        | Sensitive accounts with no legitimate use — any attribute change = alert |
| Least privilege                        | Reducing the number of DA accounts reduces attack opportunity            |

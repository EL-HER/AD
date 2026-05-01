# 🛡️Domain Controller Deep Dive

### <mark style="color:$primary;">What the Domain Controller Is</mark>

The Domain Controller is the server that runs Active Directory Domain Services. It is the central authority for:

* <mark style="color:red;">**Authentication**</mark> — verifying who you are
* <mark style="color:red;">**Authorisation**</mark> — determining what you can access
* <mark style="color:red;">**Policy enforcement**</mark> — pushing Group Policy to every machine
* <mark style="color:red;">**DNS resolution**</mark> — resolving domain names inside the network

Every Windows domain has at least one. Most production environments have two or more for redundancy.

***

### <mark style="color:$primary;">What Lives on the DC</mark>

#### <mark style="color:$warning;">NTDS.dit</mark>

The Active Directory database. <mark style="color:blue;">The most sensitive file in any Windows</mark> enterprise environment.

```
Location:   C:\Windows\NTDS\NTDS.dit
Contains:   NT hashes for every account in the domain
            Kerberos keys
            Password history
            All AD objects and their attributes
Access:     Locked by the OS while Windows is running
            Only the DC process reads it during normal operation
```

If an attacker obtains NTDS.dit, they have the NT hash of every account in the domain. They can crack passwords offline, perform pass-the-hash attacks, or use DCSync to extract it without touching the file directly.

#### <mark style="color:$warning;">SYSVOL</mark>

A shared folder replicated across all DCs. Contains:

* <mark style="background-color:blue;">Group Policy templates and settings</mark>
* <mark style="background-color:blue;">Login scripts</mark>
* <mark style="background-color:blue;">Domain-wide files that need to be accessible from any DC</mark>

#### <mark style="color:$warning;">The KDC</mark> (Key Distribution Center)

A logical role on the DC. Handles all Kerberos authentication. Contains two sub-services:

* <mark style="color:$primary;">**AS**</mark>**&#x20;(Authentication Server)** — issues Ticket Granting Tickets (TGTs)
* <mark style="color:$primary;">**TGS**</mark>**&#x20;(Ticket Granting Service)** — issues Service Tickets

***

### <mark style="color:red;">The krbtgt Account</mark>

Every domain has a special account called `krbtgt`. <mark style="color:blue;">It is the KDC's own service account</mark>.

```
Purpose:    Its NT hash is used to encrypt ALL Ticket Granting Tickets
Visibility: Hidden — not a real user, can't log in interactively
Password:   Never expires, never changes unless manually rotated
Risk:       If its hash is compromised → Golden Ticket attack
```

The `krbtgt` hash <mark style="color:red;">is the master key to the domain</mark>. Whoever has it can forge a <mark style="color:blue;">valid TGT for any user</mark>, <mark style="color:blue;">with any group memberships, with any expiry time</mark>. This is called a <mark style="color:$primary;">**Golden Ticket**</mark>.

The only recovery from a Golden Ticket attack is rotating the krbtgt password **twice** — because one rotation still leaves existing forged tickets valid.

***

### <mark style="color:$primary;">Multiple Domain Controllers</mark>

Production environments always run multiple DCs for availability. When one DC goes down, authentication continues through another.

#### <mark style="color:$primary;">Replication</mark>

All DCs must stay synchronised. <mark style="color:blue;">When</mark> a <mark style="color:blue;">user changes their password on DC-1, DC-2</mark> must learn about it. This synchronisation is <mark style="color:red;">called</mark> <mark style="color:$primary;">**AD Replication**</mark> and <mark style="color:$primary;">uses the</mark> <mark style="color:$primary;"></mark><mark style="color:$primary;">**DRSR protocol**</mark> (Directory Replication Service Remote).

```
DC-1  ──── replication (DRSR) ────►  DC-2
DC-2  ──── replication (DRSR) ────►  DC-1
```

This legitimate replication mechanism is what the **DCSync attack** abuses — Mimikatz implements DRSR and pretends to be a DC to extract credentials.

#### <mark style="color:$primary;">FSMO Roles</mark>

Not all DCs are equal. Five special roles exist called FSMO (Flexible Single Master Operations):

| Role                  | Purpose                                          |
| --------------------- | ------------------------------------------------ |
| PDC Emulator          | Primary DC — handles password changes, time sync |
| RID Master            | Issues unique IDs to new objects                 |
| Infrastructure Master | Tracks cross-domain references                   |
| Schema Master         | Controls AD schema modifications                 |
| Domain Naming Master  | Controls adding/removing domains                 |

***

### <mark style="color:$primary;">Static IP Requirement</mark>

The DC must have a static IP address. Every domain-joined machine has its DNS configured to point to the DC's IP. If the DC's IP changes:

* <mark style="background-color:blue;">Domain-joined machines cannot find the DC</mark>
* <mark style="background-color:blue;">Authentication fails across the entire domain</mark>
* <mark style="background-color:blue;">Group Policy stops applying</mark>

Beyond availability, a static IP prevents another machine from temporarily claiming the DC's IP and intercepting authentication traffic.

***

### <mark style="color:$primary;">Joining the Domain</mark>

When a machine joins a domain, it:

1. <mark style="background-color:blue;">Contacts the DC via DNS</mark>
2. <mark style="background-color:blue;">Creates a computer account object in AD</mark>
3. <mark style="background-color:blue;">Establishes a secure channel with the DC</mark>
4. <mark style="background-color:blue;">Starts receiving Group Policy</mark>
5. <mark style="background-color:blue;">Begins deferring all authentication to the DC</mark>

After joining, local accounts still exist but domain authentication takes priority.

***

### <mark style="color:$danger;">Defensive Takeaways</mark>

| Control                              | Why It Matters                                               |
| ------------------------------------ | ------------------------------------------------------------ |
| Two or more DCs                      | Single DC = single point of failure for the entire domain    |
| Dedicated DC hardware                | Never run other services on a DC — reduces attack surface    |
| Restrict physical access             | Physical access to DC = access to NTDS.dit via offline tools |
| Monitor DRSR traffic                 | Replication from a non-DC machine = DCSync attack            |
| Rotate krbtgt twice after compromise | One rotation is insufficient — Golden Tickets remain valid   |
| Backup NTDS.dit securely             | Backup is also a target — protect it like the live file      |

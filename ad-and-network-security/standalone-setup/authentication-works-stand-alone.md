# 🔓Authentication Works Stand-Alone

***

### How Authentication Works

Authentication on a standalone machine is entirely local. No network traffic involved.

```
User types username and password
               │
               ▼
Machine hashes the password
MD4 → NT Hash
               │
               ▼
Compare against NT hash stored in SAM file
C:\Windows\System32\config\SAM
               │
          ┌────┴────┐
       Match      No match
          │            │
    ✅ Logged in   ❌ Access denied
```

The SAM (Security Account Manager) file is the local credential database. It contains the NT hashes of all local user accounts. It is locked by the OS while Windows is running.

***

### The Workgroup

Standalone machines can be connected in a network called a **workgroup**. This allows them to share files and printers — but it does not centralise identity.

```
What a workgroup provides:
  ✓ Network connectivity between machines
  ✓ Ability to share folders and printers
  ✓ Basic network resource discovery

What a workgroup does NOT provide:
  ✗ Shared user accounts
  ✗ Centralised authentication
  ✗ Group Policy
  ✗ Single Sign-On
```

If user Alice wants to access a shared folder on PC-02, PC-02 must have its own local account for Alice — with its own password. If Alice changes her password on her machine, it has no effect on PC-02.

***

### Characteristics

| Property                 | Standalone                                       |
| ------------------------ | ------------------------------------------------ |
| User database            | Local SAM file — separate on every machine       |
| Authentication authority | The machine itself                               |
| Password policy          | Configured per machine — inconsistent by default |
| Group Policy             | Not available                                    |
| Single Sign-On           | No                                               |
| Centralised management   | No — changes must be made per machine            |
| Scalability              | Breaks down at 5–10 machines                     |
| Cost                     | Low — no server needed                           |
| Setup complexity         | Low                                              |

***

### Security Profile

#### The Advantage — Limited Blast Radius

Compromising one standalone machine does not automatically give access to others. An attacker who gains admin on PC-01 has PC-01. To move to PC-02, they need to attack PC-02 separately.

```
Attacker compromises PC-01
         │
         ▼
Gets local admin + SAM hashes from PC-01
         │
         ├─ IF same local admin password is reused → pivot to PC-02, PC-03
         │
         └─ IF unique passwords per machine → attacker is contained
```

The key is password reuse. If administrators use the same local admin password across all machines (a very common practice), the isolation disappears.

#### The Disadvantages

```
Problem 1 — No centralised visibility
  If PC-01 is compromised, there are no centralised logs.
  You may never know unless you physically check PC-01.

Problem 2 — Inconsistent security posture
  PC-01 might require 12-char passwords.
  PC-02 might have no password policy at all.
  PC-03's admin account might have no password.
  You have no way to enforce consistency.

Problem 3 — Password reuse risk
  Every machine needs a local admin account.
  Administrators reuse passwords because managing
  unique passwords per machine manually is impractical.
  One compromised machine → all machines.

Problem 4 — No centralised response capability
  Compromised user account?
  You must go to every machine and disable it separately.
  No way to lock out an account network-wide.
```

***

### When Standalone Makes Sense

```
✅ Use standalone when:
   • 1–5 machines, no plans to scale
   • No shared resources requiring centralised management
   • No compliance requirements
   • Home environment or small personal office
   • Air-gapped or isolated lab machines
   • Temporary test environments

❌ Do not use standalone when:
   • More than 5–10 machines
   • Multiple users need access to shared resources
   • Compliance or audit requirements exist
   • IT team needs centralised management
   • Security policy needs to be consistent across machines
```

***

### Defensive Takeaways

| Control                                                | Why It Matters                                               |
| ------------------------------------------------------ | ------------------------------------------------------------ |
| Unique local admin passwords per machine               | Prevents single compromise from spreading to all machines    |
| Microsoft LAPS (Local Administrator Password Solution) | Automates unique random passwords for local admin accounts   |
| Disable the default Administrator account              | Use a renamed admin account — default name is a known target |
| Enable Windows Firewall on all machines                | Each machine must defend itself independently                |
| Regular local security audits                          | No centralised logging — must check each machine manually    |

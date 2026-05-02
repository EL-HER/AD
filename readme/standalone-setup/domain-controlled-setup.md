# 🏢Domain-Controlled Setup

### What a Domain Environment Is

A domain environment introduces a central server — the **Domain Controller** — that owns all user identities, enforces all policies, and controls access across the entire network.

Every machine joins the domain. Every user authenticates against the DC. One identity works everywhere.

```
                  ┌──────────────────────────┐
                  │     DOMAIN CONTROLLER     │
                  │                          │
                  │   NTDS.dit               │
                  │   ┌──────────────────┐   │
                  │   │ alice : [hash]   │   │
                  │   │ bob   : [hash]   │   │
                  │   │ admin : [hash]   │   │
                  │   │ krbtgt: [hash]   │   │
                  │   └──────────────────┘   │
                  │                          │
                  │   Kerberos KDC           │
                  │   Group Policy Engine    │
                  │   DNS Server             │
                  └────────────┬─────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
       ┌────┴────┐        ┌────┴────┐        ┌────┴────┐
       │  PC-01  │        │  PC-02  │        │  PC-03  │
       │(joined) │        │(joined) │        │(joined) │
       └─────────┘        └─────────┘        └─────────┘

One user database for all machines.
Alice uses the same credentials on PC-01, PC-02, and PC-03.
Policy change on DC → instantly applies everywhere.
```

***

### How Authentication Works

In a domain environment, authentication flows through the DC via **Kerberos** (primary) or **NTLMv2** (fallback). The machine never validates credentials locally for domain accounts.

#### Kerberos (Primary)

```
1. User logs in to PC-01
2. PC-01 sends AS-REQ to the KDC on the DC
   (timestamp encrypted with user's NT hash)
3. KDC validates, returns TGT
4. User now has a TGT valid for 10 hours
5. To access a resource, PC-01 requests a Service Ticket
6. Service Ticket presented to target service
7. Service validates ticket and grants access
```

The password never leaves the machine. The NT hash is used as a key, not transmitted.

#### NTLMv2 (Fallback)

Used when Kerberos cannot be applied — for example, when connecting to a resource by IP address instead of hostname. The DC still validates the authentication via the challenge/response mechanism described in Module 1.

***

### Joining the Domain

When a machine joins a domain:

```
Step 1: Administrator enters domain name and DC credentials
Step 2: Machine contacts DC via DNS
Step 3: Computer account created in AD (under Computers container by default)
Step 4: Machine receives a computer password — auto-managed, rotates every 30 days
Step 5: Machine configured to use DC for DNS
Step 6: Group Policy applied on first restart
Step 7: Domain authentication now works on this machine
```

After joining, local accounts still exist on the machine. But domain accounts take precedence, and Group Policy manages the machine's behaviour.

***

### What the Domain Controls

#### Users

Every person in the organisation has a single user account in AD. One username, one password hash, one set of group memberships — valid on every machine in the domain.

#### Group Policy Objects (GPOs)

Rules defined centrally and pushed to every joined machine:

```
Examples of what GPO controls:
  • Minimum password length and complexity
  • Account lockout threshold and duration
  • Software installation and restriction
  • USB and removable media policies
  • Windows Firewall rules
  • Desktop wallpaper and screensaver settings
  • Mapped drives and printers
  • Certificate deployment
  • Security baselines
```

GPOs are applied in a specific order: Local → Site → Domain → OU. OUs allow you to apply different policies to different departments.

#### Organisational Units (OUs)

Containers that allow granular management:

```
Domain: bank.local
│
├── OU: IT Department
│   ├── Users: alice, bob
│   ├── Computers: IT-PC-01, IT-PC-02
│   └── GPO: Full admin tools, no USB restriction
│
├── OU: Finance Department
│   ├── Users: mary, john
│   ├── Computers: FIN-PC-01, FIN-PC-02
│   └── GPO: Restricted software, USB disabled, audit all file access
│
└── OU: HR Department
    ├── Users: sarah
    └── GPO: Standard user restrictions
```

***

### Characteristics

| Property                 | Domain-Controlled                                  |
| ------------------------ | -------------------------------------------------- |
| User database            | Centralised NTDS.dit on the DC                     |
| Authentication authority | Domain Controller                                  |
| Password policy          | Set once on DC, enforced everywhere                |
| Group Policy             | Full GPO support                                   |
| Single Sign-On           | Yes — log in once, access all authorised resources |
| Centralised management   | Yes — manage all users and machines from DC        |
| Scalability              | Scales to thousands of machines                    |
| Cost                     | Higher — Windows Server license + hardware         |
| Setup complexity         | Moderate to high                                   |

***

### Security Profile

#### The Advantages

**Centralised visibility:** Every authentication generates event logs on the DC. One SIEM integration gives visibility across the entire network.

**Rapid response:** Compromised user account? Disable it once on the DC. Locked out everywhere instantly — every machine, every service.

**Policy consistency:** Every machine receives the same security baseline. No machines with no password policy. No machines with USB ports open.

**Audit trail:** Every login, every privilege use, every group membership change is logged in one place.

#### The Risks

**Single point of catastrophic failure:** Compromise the DC and you compromise the entire domain. This is not theoretical — it is the end goal of most enterprise attacks.

```
Attack path:
  1. Phish a user → get low-privilege domain account
  2. Kerberoast a service account → elevate
  3. Lateral movement → reach a machine with DA credentials cached
  4. DCSync → dump all NT hashes from NTDS.dit
  5. Game over — every account compromised
```

**Centralised attack surface:** The same centralised control that makes defence easier also makes attacks more powerful. One credential in the right place can unlock the entire domain.

***

### When Domain Makes Sense

```
✅ Use domain when:
   • 10+ machines in the environment
   • Multiple users need access to shared resources
   • Compliance or audit requirements exist
   • IT team needs to manage users and machines at scale
   • Security policy must be consistent and enforceable

❌ Domain is overkill when:
   • Fewer than 5 machines, single administrator
   • No IT budget for server infrastructure
   • Temporary or isolated environment
```

***

### Defensive Takeaways

| Control                              | Why It Matters                                                       |
| ------------------------------------ | -------------------------------------------------------------------- |
| Tiered administration model          | Domain Admins only log into DCs — never workstations                 |
| Privileged Access Workstations (PAW) | Dedicated machines for admin tasks — reduces credential exposure     |
| Monitor NTDS.dit access              | Any non-standard access = immediate alert                            |
| Alert on Domain Admin group changes  | Unexpected additions = persistence attempt                           |
| Monitor DRSR traffic                 | Replication from a workstation = DCSync in progress                  |
| Enable Advanced Audit Policy         | Granular logging of authentication, privilege use, and object access |

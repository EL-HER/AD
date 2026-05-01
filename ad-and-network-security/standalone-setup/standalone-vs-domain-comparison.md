---
icon: code-compare
---

# Standalone vs Domain — Comparison

### The Fundamental Difference

```
Standalone:  each machine is its own authority
Domain:      the DC is the authority for everything

This single difference defines:
  → how users authenticate
  → how permissions work
  → how attacks succeed or fail
  → how defenders respond
```

***

### Side-by-Side Comparison

| Dimension               | Standalone                    | Domain-Controlled                   |
| ----------------------- | ----------------------------- | ----------------------------------- |
| **Identity store**      | SAM file — local, per machine | NTDS.dit — central, on DC           |
| **Authentication**      | Machine validates locally     | DC validates via Kerberos / NTLMv2  |
| **Who validates login** | The machine itself            | The Domain Controller               |
| **User accounts**       | Separate on every machine     | Single identity across all machines |
| **Password policy**     | Per machine — inconsistent    | Central — enforced everywhere       |
| **Group Policy**        | Not available                 | Full GPO support                    |
| **Single Sign-On**      | No                            | Yes                                 |
| **Remote management**   | Manual per machine            | Centralised from DC                 |
| **Scalability**         | Breaks down at 5–10 machines  | Scales to thousands                 |
| **Event logging**       | Fragmented — per machine      | Centralised on DC                   |
| **Incident response**   | Must touch each machine       | Disable once, blocked everywhere    |
| **Cost**                | Low                           | Higher — server + licensing         |
| **Complexity**          | Low                           | Moderate to high                    |

***

### Authentication Comparison

#### Standalone — Local Validation

```
User types password
        │
        ▼
Machine hashes it (MD4 → NT Hash)
        │
        ▼
Compares against SAM file
        │
   Pass / Fail decided locally
   No network traffic
   No central record of this login
```

#### Domain — Centralised Validation via Kerberos

```
User types password
        │
        ▼
Machine hashes it (MD4 → NT Hash)
        │
        ▼
Sends AS-REQ to KDC (timestamp encrypted with hash)
        │
        ▼
KDC validates, returns TGT
        │
        ▼
User presents TGT to get Service Tickets
        │
        ▼
Every step logged on the DC
```

***

### Attack Surface Comparison

This is where the difference matters most.

#### Standalone Attack Surface

```
Attacker's goal: access resources on the network

Path:
  Attack PC-01 → get local admin on PC-01
  └─ If local admin password reused on PC-02 → pivot
  └─ If not reused → must attack PC-02 separately

Blast radius of single compromise: ONE machine
Effort to compromise all machines: HIGH (each one separately)
Visibility to defenders: LOW (no central logs)
```

#### Domain Attack Surface

```
Attacker's goal: own the domain

Path:
  1. Low-privilege domain account (phishing / brute force)
  2. Kerberoasting → crack service account → elevated access
  3. Lateral movement to machine with DA token cached
  4. DCSync → krbtgt hash → Golden Ticket
  5. All machines, all users, all services compromised

Blast radius of DC compromise: ENTIRE DOMAIN
Effort to compromise everything after DC: MINIMAL
Visibility to defenders: HIGH (if logging is configured)
```

The domain centralises both attack power and defensive power. The outcome depends entirely on which side builds better.

***

### Real-World Scenarios

#### Scenario A — Small Café (Standalone is right)

```
3 machines: owner laptop + POS terminal + back-office PC
2 staff members
No shared login needed
Zero IT budget for servers

Decision: Standalone
Reason:   Domain infrastructure would cost more than it protects
Risk:     Limited — 3 machines, unique passwords, minimal exposure
```

#### Scenario B — Law Firm with 50 Staff (Domain is essential)

```
65 user accounts
File server + print server + email server + billing system
Compliance requirements (client confidentiality)
IT team of 2

Without domain:
  65 × 4 servers = 260 separate account configurations to manage
  No way to enforce consistent password policy
  No audit trail for who accessed client files when
  One departing employee = manual account deletion on 4 systems

With domain:
  One account per user
  Disable departing employee once → locked out everywhere
  GPO enforces security baseline on all machines
  Full audit trail on DC for compliance review
```

#### Scenario C — Attacker's Perspective

```
Target: Standalone environment (10 machines)
  Compromise one machine: 45 minutes
  Compromise all 10: requires attacking each individually
  Total effort: HIGH
  Worst case for attacker: local admin on 10 machines

Target: Domain environment (10 machines + 1 DC)
  Compromise one machine: 45 minutes
  Escalate to Domain Admin: variable (hours to days)
  After DA: dump NTDS.dit → all 10 machines + all users
  Total effort: MODERATE then LOW
  Worst case for attacker: owned the entire domain
```

***

### Making the Right Choice

```
Ask these questions:

1. How many machines?
   < 5   → standalone probably fine
   5–15  → consider domain
   15+   → domain is the right answer

2. Do multiple users need to log into multiple machines?
   No    → standalone may work
   Yes   → domain simplifies this enormously

3. Do you have compliance requirements?
   No    → either can work
   Yes   → domain with audit logging is nearly required

4. Do you have IT staff?
   No    → standalone is simpler to manage
   Yes   → domain scales much better with dedicated IT

5. What is your risk tolerance?
   Low   → domain with proper hardening
   High  → standalone with manual controls
```

***

### Defensive Posture by Architecture

#### Standalone — Priority Controls

```
1. Unique local admin passwords on every machine (LAPS)
2. Disable default Administrator account name
3. Full disk encryption (BitLocker)
4. Local firewall enabled and configured
5. Regular manual security audits per machine
```

#### Domain — Priority Controls

```
1. Protect the DC above all else (Tier-0)
2. Monitor DRSR traffic — workstation to DC = DCSync
3. Alert on Domain Admin group changes
4. Enforce AES Kerberos — disable RC4
5. gMSA for all service accounts
6. Alert on Event 4769 with EncryptionType 0x17
7. Tiered administration — admins never log into workstations
```

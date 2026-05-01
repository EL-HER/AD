---
icon: arrows-to-circle
layout:
  width: default
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
---

# Active Directory Fundamentals

## <mark style="color:$primary;">What is Active Directory?</mark>

### <mark style="color:blue;">The Core Idea</mark>

Active Directory (AD) is Microsoft's centralised identity and access <mark style="color:$warning;">management service</mark>. It controls every user, device, and permission in a Windows network from a single server — the **Domain Controller**.

<mark style="color:blue;">Before AD</mark> existed, each machine in a network managed its own users independently. If you had 100 machines, you had 100 separate user databases. Changing one password meant changing it 100 times. Controlling who had access to what was nearly impossible at scale.

AD solved this by introducing <mark style="color:$warning;">**centralised control**</mark>. One place. One database. One policy engine. Every machine in the network defers to it.

***

### <mark style="color:$primary;">What AD Actually Manages</mark>

```
Active Directory
       │
       ├── Users          → accounts, passwords (stored as NT hashes)
       ├── Computers      → every device that joins the domain
       ├── Groups         → collections of users sharing permissions
       ├── OUs            → containers for organising objects
       └── Group Policy   → rules enforced across all joined machines
```

<mark style="color:red;">**Users**</mark> — every person who logs into the domain has an account in AD. Their <mark style="color:blue;">identity</mark>, <mark style="color:blue;">password hash</mark>, <mark style="color:blue;">group memberships</mark>, and <mark style="color:blue;">permissions</mark> all live here.

<mark style="color:red;">**Computers**</mark> — every workstation and server that <mark style="color:blue;">joins the domain</mark> becomes an object in AD. This is how Group Policy reaches them.

<mark style="color:red;">**Groups**</mark> — instead of <mark style="color:blue;">assigning permissions to individual users</mark>, you assign them to groups. Users inherit permissions through group membership.

<mark style="color:red;">**Organisational Units (OUs)**</mark> — logical containers. You put <mark style="color:blue;">users</mark>, <mark style="color:blue;">computers</mark>, and <mark style="color:blue;">groups</mark> inside <mark style="color:red;">OUs</mark> <mark style="color:blue;">to organise them by department or function</mark>. Group Policy is applied at the OU level.

<mark style="color:red;">**Group Policy (GPO)**</mark> — <mark style="color:blue;">rules</mark> defined on the Domain Controller and <mark style="color:blue;">pushed</mark> to every joined machine. <mark style="background-color:blue;">**Control password complexity, disable USB ports, restrict software, configure firewall rules — all from one place.**</mark>

***

### <mark style="color:$primary;">The Domain</mark>

A **domain** is the <mark style="color:blue;">logical boundary of an AD</mark> environment. Every object inside the domain — users, computers, groups — belongs to it and is managed by its Domain Controller.

Domains are identified by a DNS name:

```
bank.local
company.internal
corp.example.com
```

Every device that joins the domain is configured to point to the DC's IP for authentication. This is why the <mark style="color:blue;">DC must have a</mark> <mark style="color:blue;"></mark><mark style="color:blue;">**static IP**</mark> — if it changes, every machine in the domain loses the ability to authenticate.

***

### <mark style="color:$primary;">Domain Trees and Forests</mark>

Large organisations can have multiple domains organised in a hierarchy:

```
                    [ bank.local ]              ← Root domain
                          │
              ┌───────────┴───────────┐
    [ egypt.bank.local ]    [ saudi.bank.local ]  ← Child domains
              │
      ┌───────┴──────────┐
[ cairo.egypt.bank.local ] [ alex.egypt.bank.local ]  ← Sub-child domains
```

<mark style="color:red;">**Tree**</mark> — a collection of domains that share a contiguous DNS namespace. <mark style="color:blue;">`egypt.bank.local`</mark> <mark style="color:blue;"></mark><mark style="color:blue;">is a child of</mark> <mark style="color:blue;"></mark><mark style="color:blue;">`bank.local`</mark> — they form a tree.

<mark style="color:red;">**Forest**</mark> — the top-level boundary. <mark style="color:blue;">A forest can contain multiple trees</mark>. All domains in the same forest share a common schema and can be configured to trust each other.

<mark style="color:red;">**Trust relationship**</mark> — <mark style="color:blue;">when two domains trust each other</mark>, users in one domain can authenticate to resources in the other. Child domains automatically have a two-way trust with their parent.

***

### <mark style="color:$primary;">Why AD Is the Highest-Value Target</mark>

Everything depends on the Domain Controller:

* <mark style="color:red;">Authentication</mark> — every login goes through it
* <mark style="color:red;">Authorisation</mark> — group memberships and permissions live in it
* <mark style="color:red;">Policy</mark> — every security rule is pushed from it
* <mark style="color:red;">Credentials</mark> — all password hashes are stored in NTDS.dit on it

If the DC is compromised, the entire domain is compromised. Not one machine. Not one user. Everything.

This is why protecting the DC is not a best practice. It is the foundation of enterprise Windows security.

***

### <mark style="color:$danger;">Defensive Takeaways</mark>

| Control                           | Why It Matters                                                            |
| --------------------------------- | ------------------------------------------------------------------------- |
| Static IP on DC                   | Prevents IP spoofing and impersonation                                    |
| Tier-0 admin model                | DC admins should only log into DCs — never use a DC for browsing or email |
| Monitor NTDS.dit access           | Any non-DC process reading this file is a credential dumping attempt      |
| Audit privileged group membership | Unexpected additions to Domain Admins = immediate alert                   |
| Restrict replication permissions  | Only DCs should have DS-Replication-Get-Changes-All                       |

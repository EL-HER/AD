---
icon: gear
---

# Standalone Setup

### What a Standalone Machine Is

A standalone Windows machine manages its own users, passwords, and resources entirely on its own — with no central authority. It does not join a domain. It does not communicate with a Domain Controller for authentication.

Every standalone machine is its own island.

```
PC-01               PC-02               PC-03
┌─────────┐         ┌─────────┐         ┌─────────┐
│ SAM DB  │         │ SAM DB  │         │ SAM DB  │
│ admin   │         │ admin   │         │ admin   │
│ user1   │         │ john    │         │ mary    │
└────┬────┘         └────┬────┘         └────┬────┘
     │                   │                   │
     └───────────────────┼───────────────────┘
                         │
                   [ Switch / Hub ]

Each machine has a completely separate user database.
"admin" on PC-01 is a different account from "admin" on PC-02.
No shared identity. No shared policy. No central control.
```

***

###

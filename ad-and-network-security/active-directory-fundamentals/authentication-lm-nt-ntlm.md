# 🔐Authentication — LM, NT, NTLM

### <mark style="color:$primary;">Why Hashing Exists</mark>

Windows never stores your password as plaintext. It stores the output of a one-way mathematical function called a **hash**.

```
Input: "Password123"

          │
          ▼  Hash Algorithm
          │
Output: "58a478135a93ac3bf058a5ea0e8fdb71"

Properties:
  One-way     → you cannot reverse the output back to the input
  Deterministic → same input always produces same output
  Fixed length  → output is always the same size regardless of input length
```

When you log in, Windows hashes what you typed and compares it to the stored hash. If they match — authenticated. The actual password is never compared directly, never stored, and never transmitted.

***

### <mark style="color:$primary;">The Problem No Hashing Solves Alone</mark>

Hashing prevents storing plaintext passwords. But it introduces a different risk: <mark style="color:red;">**offline cracking**</mark><mark style="color:red;">.</mark>

<mark style="background-color:blue;">If an attacker steals the hash, they do not need to break into your system again. They run the cracking tool on their own machine, trying millions of passwords per second, comparing the hash of each attempt to the stolen hash. No lockout. No alerts. No time pressure.</mark>

The strength of the protection comes entirely from the algorithm design and the password strength.

***

### Hash Type 1 — <mark style="color:$primary;">LM Hash</mark>

Used in Windows XP and Windows Server 2003. Disabled by default since Vista.

#### <mark style="color:violet;">How It Works — Step by Step</mark>

```
Input: "password"

Step 1: Force all characters to uppercase
        "password" → "PASSWORD"

Step 2: Pad to exactly 14 characters
        "PASSWORD" → "PASSWORDOOOOOOO"
        (O represents null padding)

Step 3: Split into two 7-character halves
        Half A: "PASSWOR"
        Half B: "DOOOOOOO"

Step 4: Convert each half to ASCII, then to binary
        Each half becomes an 8-byte block

Step 5: Encrypt each 8-byte block with DES
        Key used: "KGS!@#$%"  ← hardcoded by Microsoft, same always

Step 6: Concatenate both DES outputs
        Result: LM Hash → stored in SAM file
```

#### <mark style="color:violet;">Why LM Hash Is Broken</mark>

```
Problem 1 — Uppercase only
  Keyspace drops from 95 printable ASCII chars to 69
  Fewer combinations = faster to crack

Problem 2 — Split into two independent halves
  A 14-character password becomes two 7-character problems
  Attackers crack each half independently
  Maximum effective length is 7 characters

Problem 3 — Fixed DES key
  "KGS!@#$%" is hardcoded — public knowledge
  No secret involved in the encryption

Problem 4 — No salt
  Same password = same LM hash on every machine everywhere
  Rainbow tables (precomputed hash→password lookups) work perfectly
```

An attacker who obtains an LM hash can crack it in minutes using modern hardware.

***

### Hash Type 2 — <mark style="color:$primary;">NT Hash</mark>

The current standard. <mark style="color:blue;">Used in all modern Windows systems.</mark>

#### <mark style="color:violet;">How It Works</mark>

```
Input: "Password"

Step 1: Encode as UTF-16 Little Endian
        Each character becomes 2 bytes
        "Password" → b'\x50\x00\x61\x00\x73\x00\x73\x00...'

Step 2: Apply MD4 hash algorithm
        Output: NT Hash — "8846F7EAEE8FB117AD06BDD830B7586C"
```

<figure><img src="../../.gitbook/assets/NTLM AUTH.png" alt=""><figcaption></figcaption></figure>

#### Properties

```
✓  Case-sensitive — "password" ≠ "Password" ≠ "PASSWORD"
✓  Supports full Unicode — international characters work
✓  Simple, fast computation
✗  No salt — same password = same hash on every Windows machine
✗  MD4 is fast — offline cracking is fast
✗  No key stretching — no iteration count to slow down attackers
```

<mark style="color:$info;">The no-salt property is critical. If two users have the same password, they have the same NT hash. If an attacker cracks one, they've cracked both. And since NT hashes are consistent across all Windows machines globally, precomputed rainbow tables are effective.</mark>

***

### Hash Type 3 — <mark style="color:$primary;">NTLM Hash</mark>

Not a different algorithm. "NTLM hash" is just the <mark style="color:blue;">NT hash referred to in the context of network authentication.</mark>

The naming is historically confusing. The map is:

```
SAM file stores:     NT hash  (local machines)
NTDS.dit stores:     NT hash  (domain controller)
NTLM protocol uses:  NT hash  as the key to process challenges
NTLMv2 uses:         NT hash  as the HMAC-MD5 key
```

When people say "dump NTLM hashes," they mean "dump NT hashes."

***

### <mark style="color:$primary;">Where Hashes Are Stored</mark>

| File                                     | Location                         | Machine Type         | Contents                         |
| ---------------------------------------- | -------------------------------- | -------------------- | -------------------------------- |
| <mark style="color:red;">SAM</mark>      | `C:\Windows\System32\config\SAM` | All Windows machines | NT hashes of local accounts      |
| <mark style="color:red;">NTDS.dit</mark> | `C:\Windows\NTDS\NTDS.dit`       | DC only              | NT hashes of all domain accounts |

Both files are locked by the OS while Windows is running. Accessing them requires either:

* <mark style="background-color:blue;">Extracting them offline (boot from live USB)</mark>
* <mark style="background-color:blue;">Using a tool like Mimikatz that reads them from memory</mark>
* <mark style="background-color:blue;">Using Volume Shadow Copies (VSS) to access locked files</mark>
* <mark style="background-color:blue;">DCSync attack (domain environments only)</mark>

***

### <mark style="color:$primary;">Cracking NT Hashes — How Attackers Do It</mark>

Once a hash is obtained, cracking happens entirely offline:

```
Tools:
  hashcat  → GPU-accelerated, processes billions of hashes per second
  john     → CPU-based, flexible rule engine

Attack methods:
  Dictionary    → try every word in a wordlist (rockyou.txt has 14 million)
  Rule-based    → apply transformations: password → P@ssw0rd → P@55w0rd
  Brute force   → try every possible character combination
  Rainbow table → look up precomputed hash→password pairs instantly

Hash type for NT in hashcat:  -m 1000
```

A weak 8-character password can be cracked in seconds. A 12-character random password takes years with current hardware.

***

### <mark style="color:$danger;">Defensive Takeaways</mark>

| Control                     | Implementation                                                       | Effect                                           |
| --------------------------- | -------------------------------------------------------------------- | ------------------------------------------------ |
| Disable LM hash storage     | Group Policy → Network security: Do not store LAN Manager hash value | Eliminates LM hashes from SAM                    |
| Enforce NTLMv2 minimum      | Group Policy → LAN Manager authentication level → NTLMv2 only        | Blocks LM and NTLMv1 responses                   |
| Password length policy      | Minimum 12 characters, preferably 16+                                | Exponentially increases cracking time            |
| Monitor SAM/NTDS.dit access | Alert on any non-DC process reading these files                      | Detects credential dumping                       |
| Credential Guard            | Windows 10/11 enterprise feature                                     | Isolates credential material in a secure enclave |

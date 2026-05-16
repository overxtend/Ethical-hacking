# Workshop 3 — Password Cracking Sprint: NetNTLMv2 with Hashcat
### *Hands-on offline credential attack against a captured NTLMv2 hash*

> **Authorized training / lab use only.** The hash you will work with in this workshop is a publicly known training sample from Microsoft's free *Internet Explorer Test VM* programme (user `IEUser` on `IE9WIN7`). It is provided here for learning. Cracking hashes that belong to other people — even hashes you obtained legally — is not authorised by this course unless your instructor has explicitly added them to scope. Cracking hashes you stole is a crime.
>
> This is the **credential-access** track of the curriculum. Password cracking is taught because (a) defenders need to know how fast a real attacker can recover a weak password from a captured hash, (b) red teams use it under contract on every internal engagement, and (c) every entry-level offensive certification (OSCP, PNPT, CRTO, CompTIA PenTest+) examines candidates on hashcat.

---

## How to use this guide

- Work in pairs (one drives, one observes). Swap after each Phase.
- Type every command exactly as written. You will replace `WORKSPACE` only — everything else is literal.
- Each command block is followed by three short sections:
  - **What you should see** — sample output (your speeds will differ).
  - **What it means** — how to interpret it.
  - **Record this** — the field(s) that go into your Cracking Report.
- ✋ marks an **Instructor Checkpoint** — stop and confirm before continuing.
- 💬 marks a **Discussion Prompt** for the wrap-up.
- 🛠 marks a **Skill sidebar** — optional depth.
- 📎 marks a reference to the **Command Appendix** (§14).
- ⚠️ marks a **safety note** — read carefully.

**Estimated total time:** ~3 h 20 min (theory 30 min · labs ~2 h 15 min · discussion + report 35 min).
A 2-hour version is in §12.

**Prerequisite:** Workshops 1 and 2 helpful but not strictly required. The credentials track is parallel to the recon/exploitation track.

---

## 1. Theory primer — what are you actually cracking? (≈ 30 min)

### 1.1 Where password cracking fits in the lifecycle

| # | Phase | Goal | This workshop |
|---|---|---|---|
| 1 | Reconnaissance | Intel | — |
| 2 | Scanning | Find services | WS1 |
| 3 | **Gaining Access** | Code execution OR credentials | WS2 (code) **/ WS3 (creds)** |
| 4 | Maintaining / Lateral Movement | Persistence, pivot via creds | (next) |
| 5 | Reporting | Deliver findings | always |

📋 **MITRE mapping:** today's exercises live under **TA0006 — Credential Access**, specifically T1003 (OS Credential Dumping — assumed precondition) and **T1110.002 (Brute Force: Password Cracking)**.

### 1.2 The three things people call "NTLM" — disambiguate before you crack

This is the single most confused point in introductory password cracking. Three different artefacts, three different hashcat modes, three different ways to obtain them.

| Name | What it is | Where it lives | Hashcat mode |
|---|---|---|---|
| **NTLM hash** (a.k.a. NT hash) | `MD4(UTF-16-LE(password))` — the *stored* password representation | SAM database, ntds.dit, LSASS memory | **1000** |
| **NetNTLMv1** | A challenge–response computed during SMB/HTTP authentication, ≈ DES-based | Captured on the wire | **5500** |
| **NetNTLMv2** | A challenge–response, HMAC-MD5-based, with client-supplied entropy — the modern default | Captured on the wire | **5600** ← *today* |

Three different artefacts. Three different ways to crack. **If you use the wrong mode, hashcat will tell you "no hashes loaded" or — worse — happily run against the wrong algorithm and find nothing.** Picking mode 5600 is the *most important decision* in this workshop.

### 1.3 How NetNTLMv2 actually works (and why you can crack it offline)

NetNTLMv2 is a **challenge–response** protocol. The password itself is never sent. Roughly:

1. Client opens a session: *"hi, I am `IEUser@IE9WIN7`."*
2. Server replies with an 8-byte random **server challenge**.
3. Client computes:
   - `NTHash         = MD4(UTF-16-LE(password))`
   - `ResponseKeyNT  = HMAC-MD5(NTHash, UPPER(username) ‖ domain)`
   - `NTProofStr     = HMAC-MD5(ResponseKeyNT, ServerChallenge ‖ ClientChallengeBlob)`
4. Client sends back: `NTProofStr ‖ ClientChallengeBlob`.

A passive sniffer who captures the full exchange has **everything needed to verify a password offline**: server challenge, client challenge blob, username, domain, and the expected `NTProofStr`. Hashcat's job is to try candidate passwords, redo steps 3, and check whether the recomputed `NTProofStr` matches.

That is the entire algorithm. **Every password attempt is one MD4 + two HMAC-MD5s** — which is why modern GPUs can run *billions* of candidates per second against NetNTLMv2.

### 1.4 Where NetNTLMv2 hashes come from in real engagements

So that students understand why this skill matters operationally:

| Source | How |
|---|---|
| **`Responder`** (Laurent Gaffié) on the LAN | Poisons LLMNR / NBT-NS / mDNS, captures hashes from any host that *mistypes* a hostname |
| **SMB relay** chains | Capture the NetNTLMv2 from a victim, replay to another server live |
| **Forced-auth tricks** (PrintNightmare, PetitPotam, `.LNK` icon UNCs, MS-RPRN) | Coerce a Windows machine to authenticate to your listener |
| **HTTP NTLM auth proxies** | Intercept the `Authorization: NTLM …` headers |
| **From inside email or document files** | Embedded UNC paths in Office docs |

In short: **anywhere a Windows machine authenticates on a network you control, you can usually collect NetNTLMv2.** Today we work with one that has already been captured for you.

### 1.5 Legal and ethical guardrails

Before any candidate password gets hashed, answer **yes** to all four:

1. Was this hash obtained inside an authorised engagement (or, today, supplied by the instructor)?
2. Am I cracking only the hashes in `~/workshops/w03_crack/ntlm.txt` — nothing brought from home, no friends' Wi-Fi PCAPs?
3. Will I treat the cracked password as **sensitive material** (no Slack screenshots, no GitHub commits)?
4. After class, will I delete `~/workshops/w03_crack/`?

💬 **Discussion prompt 1:** Possessing a captured NetNTLMv2 hash is legally distinct from cracking it. Why might the **act of cracking** be the moment a court considers an investigation to have begun "actively"?

---

## 2. Preflight checklist (≈ 15 min)

### 2.1 Confirm your tooling

```bash
hashcat --version
hashcat -I              # list compute devices (GPU/CPU) hashcat can see
```

You need hashcat **6.0.0 or newer**. On Kali:

```bash
sudo apt update && sudo apt install -y hashcat
```

🛠 **Skill sidebar — GPU vs CPU:** hashcat will use whichever OpenCL/HIP/CUDA devices it can find. A real GPU will hash NetNTLMv2 at hundreds of MH/s; a VM-only CPU will do MH/s. If `hashcat -I` shows only `[CPU]` and your wordlist is small (today is fine), proceed. If you plan to throw `rockyou.txt` at this on CPU, get a coffee.

If hashcat refuses to start because the only device is CPU, you may need `--force` (it disables the "you probably want a GPU" warning). On a real engagement, **don't `--force` blindly** — it can mask genuine driver problems.

### 2.2 Benchmark mode 5600

```bash
hashcat -b -m 5600
```

**What you should see** (numbers will be very different on your hardware):

```
Hashmode: 5600 - NetNTLMv2

Speed.#1.........:   12345.6 MH/s (4.21ms) @ Accel:512 Loops:128 Thr:64 Vec:1
```

**What it means:** how many candidate passwords your hardware can compute per second against NetNTLMv2 specifically. Other algorithms (e.g. bcrypt, Argon2) will be **orders of magnitude slower** — by design. This number is your **time budget**: with `rockyou.txt` (≈14 million entries) at 100 MH/s you finish in ≈0.14 seconds; at 100 kH/s you finish in 140 seconds. The math is *that* direct.

**Record this:** your benchmark speed for mode 5600 — it goes in the report.

### 2.3 Create the workspace

```bash
mkdir -p ~/workshops/w03_crack && cd ~/workshops/w03_crack
```

### 2.4 Locate (and unzip if needed) the system wordlists

Kali ships with several. The classic `rockyou.txt` is gzipped by default:

```bash
ls -lh /usr/share/wordlists/
# If you see rockyou.txt.gz but no rockyou.txt:
sudo gunzip -k /usr/share/wordlists/rockyou.txt.gz
ls -lh /usr/share/wordlists/rockyou.txt    # ~133 MB, ~14M lines
```

✋ **Instructor Checkpoint 1:** Show the instructor your `hashcat -I` output, your `hashcat -b -m 5600` line, and `ls /usr/share/wordlists/rockyou.txt`.

---

## 3. Phase 0 — Identify the hash type (≈ 15 min)

**Objective:** prove the hash is NetNTLMv2 before running hashcat. Picking the wrong mode wastes hours.

### 3.1 The hash you will be working with

```
IEUser::IE9WIN7:4ee1d2e9de221a54:AA536EFA7896BAD02CFE2A4032DBBC7A:010100000000000080A10B8EC344DB01E93B71E5FE12EE5700000000020008004C0050003300360001001E00570049004E002D004E004400410051004D00370053004D0035005100590004003400570049004E002D004E004400410051004D00370053004D003500510059002E004C005000330036002E004C004F00430041004C00030014004C005000330036002E004C004F00430041004C00050014004C005000330036002E004C004F00430041004C000700080080A10B8EC344DB0106000400020000000800300030000000000000000100000000200000D3E8261D25C4ECE21FD9E908E8CC8385AF43163DF5F7B8A5DCAC7B943E2A443A0A001000000000000000000000000000000000000900120063006900660073002F006400610074006100000000000000000000000000
```

(One single line — copy it as one line.)

### 3.2 Anatomy of the line — read it before you hash it

Split on the colons:

```
IEUser :: IE9WIN7 : 4ee1d2e9de221a54 : AA536EFA7896BAD02CFE2A4032DBBC7A : 0101000000000000…
   ^         ^             ^                       ^                              ^
   |         |             |                       |                              |
   username  domain        8-byte server           16-byte NTProofStr             NTLMv2 client-challenge
   (empty    (or work-     challenge (16 hex)      (32 hex chars — the           blob (variable length;
   field     station)                              thing you'll match against)    contains timestamps,
   between                                                                        target info, GUIDs)
   :: is the
   "Domain"
   slot, conv-
   entionally
   empty here)
```

**Five colon-separated fields** = the visual fingerprint of NetNTLMv2. Compare to NetNTLMv1 (mode 5500), which is **shorter** and structured differently:

```
NetNTLMv1 example:  user::DOMAIN:24chars:48chars:16chars   (no long blob)
```

### 3.3 Cross-check with the hashcat example-hashes page

Open the official list of example hashes:

```
https://hashcat.net/wiki/doku.php?id=example_hashes
```

Use **Ctrl+F** → search `NetNTLMv2`. You'll find an example hash that has the same **shape** as the line above. Confirm: **mode 5600**.

🛠 **Skill sidebar — when you receive an unknown hash:**
1. Look at it. Count the parts, count the characters.
2. Run `hashid '<hash>'` or `name-that-hash <hash>` for an automated guess.
3. **Verify against the hashcat wiki example hashes page** — that page is canonical.
4. If still ambiguous, ask the person who gave you the hash where it came from. *"From Responder"* → 5600. *"From `secretsdump.py`"* → 1000.

### 3.4 Save the hash to a file

```bash
nano ntlm.txt
```

Paste the single long line. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

Verify it's exactly one line:

```bash
wc -l ntlm.txt        # should print: 1 ntlm.txt
head -c 60 ntlm.txt   # should start: IEUser::IE9WIN7:4ee1d2e9de221a54:...
```

⚠️ **Common formatting mistakes:**
- Pasted as **two lines** (line wrap in the editor) → hashcat will say "Token length exception."
- A trailing **space** or **newline** in the middle → same.
- Replaced the colons with semicolons by autocorrect → "Hashfile is empty or corrupted."

**Record this:** the username (`IEUser`), the domain field (`IE9WIN7`), and confirm "hashcat mode = 5600."

✋ **Instructor Checkpoint 2:** Show the instructor: your `ntlm.txt` is exactly one line, `head -c 60 ntlm.txt` looks right, and you can name the mode (5600) and justify it (NetNTLMv2 — five fields, includes long blob).

---

## 4. Phase 1 — Build the wordlist (≈ 15 min)

**Objective:** start small to learn the workflow, then scale to `rockyou.txt`.

### 4.1 A small custom wordlist to begin with

```bash
cat > pass.txt <<'EOF'
password
paswword
p4ssword
pass
p4ss
msfadmin
Passw0rd!
charley
abc123
EOF
wc -l pass.txt
```

**What you should see:** `9 pass.txt`.

🛠 **Skill sidebar — why deliberately weak wordlists for teaching:** you want today's first attack to **succeed** in a second, so students see the whole hashcat workflow end-to-end. After that, scaling to `rockyou.txt` is just changing one argument.

### 4.2 The full-fat option — `rockyou.txt`

`rockyou.txt` is the leaked dump from a 2009 incident, still the de-facto first-wordlist on every engagement:

```bash
ls -lh /usr/share/wordlists/rockyou.txt
wc -l /usr/share/wordlists/rockyou.txt
```

**You should see** ≈14.3M lines, ≈133 MB.

🛠 **Skill sidebar — wordlist hierarchies:** a professional cracking rig typically tries in this order:

1. **rockyou.txt** (fast win against weak passwords)
2. **HaveIBeenPwned NTLM/SHA1 corpus** (650M+ passwords, real-world breached)
3. **Custom wordlist** built from the target's website with `cewl`
4. **Rule-augmented** dictionaries (next section)
5. **Mask attacks** (next section)
6. **Pure brute force** (rarely worth it past 8 chars)

### 4.3 Record your wordlist choice

| Wordlist | Lines | Bytes | Source |
|---|---|---|---|
| pass.txt | 9 | tiny | hand-typed |
| rockyou.txt | ~14M | ~133 MB | shipped with Kali |

---

## 5. Phase 2 — Dictionary attack (`-a 0`) (≈ 25 min)

**Objective:** crack the password using a straight word-by-word comparison against `pass.txt`, then scale up.

### 5.1 Hashcat attack modes — pick the right one

| `-a` | Name | What it does |
|---|---|---|
| `0` | **Straight** (dictionary) | One candidate per line of the wordlist — **today's primary mode** |
| `1` | Combinator | Concatenates each line of wordlist 1 with each of wordlist 2 |
| `3` | Mask / brute-force | Generates candidates from a template like `?u?l?l?l?l?l?d?s` |
| `6` | Hybrid (dict + mask) | Word + mask suffix (`password` + `?d?d?d?d`) |
| `7` | Hybrid (mask + dict) | Mask prefix + word |
| `9` | Association | Match per-line, useful for username-as-password |

**A wordlist file is for `-a 0`. A mask string is for `-a 3`. They are not interchangeable.** Mixing them is the most common student mistake — hashcat may not error, it may just produce nothing.

### 5.2 Run the dictionary attack

```bash
hashcat -m 5600 -a 0 ntlm.txt pass.txt
```

**Read the command left to right:**
- `-m 5600` — hash mode is NetNTLMv2.
- `-a 0` — attack mode is "straight" (dictionary).
- `ntlm.txt` — the file containing the hash(es) to crack.
- `pass.txt` — the wordlist of candidates.

(If hashcat refuses to start because of the CPU-only warning, add `--force`. Don't make that a habit.)

**What you should see (truncated, real numbers depend on your hardware):**

```
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 …) - Platform #1 [...]
* Device #1: …

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, …

Hashfile 'ntlm.txt' on line 1 (IEUser::IE9WIN7:…): … Token length exception
                                        ^ ignore this if it doesn't appear
…

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5600 (NetNTLMv2)
Hash.Target......: IEUSER::IE9WIN7:4ee1d2e9de221a54:aa536efa7896bad…
Time.Started.....: …
Time.Estimated...: 0 secs
Speed.#1.........:   1234 H/s
Recovered........: 1/1 (100.00%) Digests
Progress.........: 9/9 (100.00%)
…

IEUser::IE9WIN7:4ee1d2e9de221a54:AA536EFA7896BAD02CFE2A4032DBBC7A:0101…  :  Passw0rd!
```

**Read the output bottom-up:**

- The final line is `<hash>:<password>`. The plaintext password for `IEUser` is **`Passw0rd!`**.
- `Status: Cracked` and `Recovered: 1/1` confirm the result.
- `Progress: 9/9` means hashcat tried all 9 candidates from your wordlist (one of which matched).

**What it means at a deeper level:** hashcat ran each of your 9 candidates through the NetNTLMv2 algorithm with the supplied server challenge and client blob, and one — `Passw0rd!` — produced the matching `NTProofStr`. That is conclusive proof: that password generated *this exact captured exchange*.

### 5.3 Retrieve the cracked password later — `--show`

Once cracked, hashcat saves the result in its **pot file** so you don't have to re-run. Recall it with:

```bash
hashcat -m 5600 --show ntlm.txt
```

**What you should see:**

```
IEUSER::IE9WIN7:4ee1d2e9de221a54:aa536efa7896bad02cfe2a4032dbbc7a:0101…:Passw0rd!
```

🛠 **Skill sidebar — the pot file lives at `~/.local/share/hashcat/hashcat.potfile`.** Every cracked hash on this machine is in there. On a multi-engagement workstation, **rotate / wipe it between engagements** — you don't want last client's hashes mixing with this client's.

### 5.4 Scale to `rockyou.txt`

```bash
hashcat -m 5600 -a 0 ntlm.txt /usr/share/wordlists/rockyou.txt
```

On a GPU this finishes in **seconds**; on a CPU-only Kali VM it takes a few minutes. Status updates: press `s` while it runs.

`Passw0rd!` happens to be inside `rockyou.txt` too — that's how prevalent it is.

**Record this:** time-to-crack for `pass.txt` vs `rockyou.txt`. The ratio between the two is your benchmark for "how aware of password reuse is my target audience."

✋ **Instructor Checkpoint 3:** Show the instructor: the `Status: Cracked` line, the `Passw0rd!` plaintext, and the output of `hashcat -m 5600 --show ntlm.txt`.

💬 **Discussion prompt 2:** `Passw0rd!` is the **default** password for the `IEUser` account in Microsoft's Internet Explorer test VMs. What does that tell you about the realistic likelihood of finding the *same* password used in production environments somewhere on Earth right now?

---

## 6. Phase 3 — Beyond the dictionary: rules and masks (≈ 30 min)

When a straight dictionary fails, you don't immediately jump to brute force. You **augment** the dictionary. Two techniques cover 90% of professional cracking work: rules and masks.

### 6.1 Rule-based attacks (`-a 0` + `-r`)

A **rule** is a tiny program that mutates each wordlist entry into many variants. Hashcat ships with several pre-built rule files in `/usr/share/hashcat/rules/`. The most famous is `best64.rule`:

```bash
hashcat -m 5600 -a 0 ntlm.txt /usr/share/wordlists/rockyou.txt \
    -r /usr/share/hashcat/rules/best64.rule
```

`best64.rule` contains 64 transformations: capitalise first letter, append `1`, `123`, `!`, `2023`, etc. Each line of rockyou is now tried **65 times** (the original + 64 variants), so the keyspace becomes ~900 million candidates.

🛠 **Skill sidebar — rule language in 90 seconds:**

| Rule | Meaning | Example: `password` → |
|---|---|---|
| `:` | do nothing | `password` |
| `l` | lowercase all | `password` |
| `u` | uppercase all | `PASSWORD` |
| `c` | capitalise | `Password` |
| `$1` | append `1` | `password1` |
| `^P` | prepend `P` | `Ppassword` |
| `sa@` | replace `a` with `@` | `p@ssword` |
| `r` | reverse | `drowssap` |
| `d` | duplicate | `passwordpassword` |
| `$!` | append `!` | `password!` |

Real rule files (`d3ad0ne.rule`, `OneRuleToRuleThemAll.rule`) chain dozens of these per line.

### 6.2 Mask attack (`-a 3`)

A **mask** generates candidates from a template. Use it when you know the password's **shape** (corporate policies often leak this):

| Mask | Built-in charset | Example: 4 lowercase letters |
|---|---|---|
| `?l` | abcdefghijklmnopqrstuvwxyz | |
| `?u` | ABCDEFGHIJKLMNOPQRSTUVWXYZ | |
| `?d` | 0123456789 | |
| `?s` | !"#$%&'()*+,-./:;…?@[\]^_`{|}~ | |
| `?a` | ?l + ?u + ?d + ?s | |
| `?l?l?l?l` | | `aaaa` … `zzzz` (456 976 candidates) |

`Passw0rd!` has the shape **Upper-lower-lower-lower-lower-digit-lower-symbol**, but a common policy mask is **`?u?l?l?l?l?l?l?s`** (one upper, then five letters, then one symbol — 8 chars total). To target *exactly* `Passw0rd!` (9 chars, one digit in the middle), use:

```bash
hashcat -m 5600 -a 3 ntlm.txt '?u?l?l?l?l?d?l?l?s'
```

That keyspace is `26^7 × 10 × 33 ≈ 2.7 × 10^12` candidates. At 1 GH/s that's about 45 minutes; at 1 MH/s it's about 31 days. **Mask attacks are exponential — choose carefully.**

For Metasploitable / IE-VM teaching, jump-start with this **smaller** mask that includes `Passw0rd!`:

```bash
hashcat -m 5600 -a 3 ntlm.txt 'Passw?ld!'
```

Here `?l` is the only unknown character. Keyspace = 26. Hashcat will find it in milliseconds. This is purely to **see the mask mechanic working** — it's not realistic.

### 6.3 Hybrid attacks (`-a 6` and `-a 7`)

Combine the strengths of both:

```bash
# Dictionary word followed by mask (e.g. password1, password12, password123…)
hashcat -m 5600 -a 6 ntlm.txt pass.txt '?d?d?d'

# Mask followed by dictionary word
hashcat -m 5600 -a 7 ntlm.txt '?u?l' pass.txt
```

`-a 6` matches the most common human pattern: a familiar word + a year or a number.

### 6.4 If you already cracked it in §5

Hashcat won't re-crack what's already in the pot file. To re-run for demonstration, either:

```bash
hashcat --potfile-disable -m 5600 -a 0 ntlm.txt pass.txt
# or wipe the pot:
rm ~/.local/share/hashcat/hashcat.potfile
```

✋ **Instructor Checkpoint 4:** Run at least one rule-based attack (§6.1) and one mask attack (§6.2). Show the instructor the commands you ran and the resulting `Status:` lines.

💬 **Discussion prompt 3:** A common corporate policy is "uppercase + lowercase + digit + symbol + 8 chars minimum." Write the hashcat mask that matches that policy literally. How many candidates does that keyspace contain? *(Hint: `?u?l?l?l?l?l?d?s` → 26 × 26⁵ × 10 × 33 ≈ 3.9 billion. A modern GPU does that in seconds.)*

---

## 7. Phase 4 — The defender's view (≈ 15 min)

**Objective:** for every offensive technique we just used, name the blue-team control that would have killed it.

### 7.1 Map every step to a defensive control

| Offensive step | Defender's mitigation | Why it works |
|---|---|---|
| **Capture the NetNTLMv2 hash** (Responder on the LAN) | Disable LLMNR and NBT-NS at the OS or via GPO | Removes the broadcast fallback that Responder poisons |
| Same | Enforce SMB signing on all servers and clients | Mitigates SMB relay |
| **Crack a weak password offline** (today) | 14+ char passphrases; password manager mandate | Pushes keyspace beyond practical brute force |
| Same | Banned-password list (e.g. Azure AD's, or a `rockyou.txt`-derived one) | `Passw0rd!`, `Welcome1`, `Summer2024` never reach production |
| **Reuse a cracked password** elsewhere | MFA on every authentication path | Stolen password alone is insufficient |
| **Pass cracked creds to another system** | Network segmentation, least privilege | Limits blast radius even if a credential is recovered |

### 7.2 The single biggest control

If you only get one item on the board today: **the password `Passw0rd!` cracked in milliseconds because it is on every public wordlist that exists.** A **14-character random passphrase from a password manager** would take the same hardware roughly the age of the universe.

The mitigation is not "make hashing slower" — that helps but is fighting Moore's Law. The mitigation is **password length and uniqueness.**

📋 **MITRE mappings for defenders:** M1027 (Password Policies), M1032 (MFA), M1037 (Filter Network Traffic — for LLMNR/NBT-NS), M1041 (Encrypt Sensitive Information — SMB signing).

💬 **Discussion prompt 4:** A pentester captures NetNTLMv2 from a network and cracks it in 0.5 seconds. The breach report blames the IT team for the cracked password. **Is that the right place to put the blame?** Argue both sides.

---

## 8. Classroom discussion (≈ 20 min)

Bring your terminal scrollback and your pot file.

1. Of the three things called "NTLM," which one did you crack today, and how would the workflow have differed if you'd received a mode-1000 hash from `secretsdump.py` instead?
2. Compare the time-to-crack of `pass.txt` and `rockyou.txt`. What does the ratio tell you about a realistic password-policy threshold?
3. You ran a rule-based attack. Which rules in `best64.rule` (or your chosen file) were responsible for the **largest fraction** of cracks (if you cracked more than one)?
4. The supplied hash includes `IEUser` (the username) and `IE9WIN7` (the workstation). Both are fed into the HMAC. **Could you have cracked this without knowing them?** Why or why not?
5. From a defender's perspective: which **single host-level configuration change** (not a network control) would have meant this hash never existed?

---

## 9. Final deliverable — Cracking Report

Submit one Markdown or PDF file per pair, named `crack_<your>_<partner>.md`, **plus** the entire `~/workshops/w03_crack/` directory (`ntlm.txt`, `pass.txt`, and a clean copy of your `~/.local/share/hashcat/hashcat.potfile` showing only today's hash).

### 9.1 Report template — copy this block as your starting point

```markdown
# Cracking Report — Target hash: IEUser@IE9WIN7
**Authors:** <name 1>, <name 2>
**Date:** <YYYY-MM-DD>
**Workshop session:** Workshop 3 — Password Cracking Sprint
**Engagement type:** authorized lab simulation
**Credential class:** NetNTLMv2 challenge–response (hashcat mode 5600)

---

## Executive summary
<!--
3–4 sentences. A reader who stops here should know:
(1) the artefact type and what algorithm it is,
(2) the wordlist + attack mode that succeeded and how long it took,
(3) the one defender-side recommendation that would have killed the whole chain.
-->

<example>
The provided artefact is a NetNTLMv2 challenge-response for user IEUser on
workstation IE9WIN7, captured during a Windows SMB authentication exchange.
Using a straight dictionary attack (hashcat -a 0) against a 9-line custom
wordlist, the plaintext password `Passw0rd!` was recovered in under one
second on a single-CPU hashcat. The single defender-side action that would
have prevented this entire chain is a 14-character minimum passphrase
policy enforced through a banned-password list — `Passw0rd!` is in every
public wordlist on the internet.
</example>

---

## 1. Hash identification (Phase 0)

| Field | Value |
|---|---|
| Username | IEUser |
| Domain / workstation | IE9WIN7 |
| Server challenge | `4ee1d2e9de221a54` |
| NTProofStr | `AA536EFA7896BAD02CFE2A4032DBBC7A` |
| Client challenge blob | `0101000000000000…` (truncated) |
| Hashcat mode | **5600 (NetNTLMv2)** |
| How identified | Five-field colon structure + long blob → matched example on hashcat wiki |

---

## 2. Environment (Phase preflight)

| Field | Value |
|---|---|
| hashcat version | <e.g. v6.2.6> |
| Compute device(s) | <e.g. CPU only — `Intel Core i5-…`> |
| `hashcat -b -m 5600` speed | <e.g. 12.3 MH/s> |
| `--force` required? | <yes / no> |

---

## 3. Dictionary attack — `pass.txt` (Phase 1+2, §5.2)

**Command:**
```
hashcat -m 5600 -a 0 ntlm.txt pass.txt
```

**Time to crack:** <e.g. 0.3 seconds>
**Candidates tried:** <e.g. 6 of 9 before match>
**Plaintext recovered:** `Passw0rd!`

**Output snippet (paste verbatim from the Status: Cracked block):**
```
Status...........: Cracked
Recovered........: 1/1 (100.00%) Digests
Progress.........: 9/9 (100.00%)
…
IEUser::IE9WIN7:…:Passw0rd!
```

---

## 4. Dictionary attack — `rockyou.txt` (§5.4)

**Command:**
```
hashcat -m 5600 -a 0 ntlm.txt /usr/share/wordlists/rockyou.txt
```

**Time to crack:** <e.g. 4.2 seconds>
**Plaintext recovered:** `Passw0rd!`
**Why it cracked:** `Passw0rd!` exists at line <#> of rockyou.txt.

---

## 5. Rule-augmented attack (§6.1)

**Command:**
```
hashcat -m 5600 -a 0 ntlm.txt /usr/share/wordlists/rockyou.txt \
    -r /usr/share/hashcat/rules/best64.rule
```

**Effective keyspace:** ~14M × 65 = ~910M candidates
**Did it crack?** <yes / already in pot>

---

## 6. Mask attack (§6.2)

**Command:**
```
hashcat -m 5600 -a 3 ntlm.txt '?u?l?l?l?l?d?l?l?s'
```

**Keyspace size:** ~2.7 × 10¹²
**Estimated time on your hardware:** <from hashcat's `Time.Estimated`>
**Realistic for production?** <yes / no — one sentence reasoning>

---

## 7. Defender's view — three recommendations

Be specific. Name a policy or a control:

1. <e.g. "Enforce a 14-character minimum passphrase policy and a banned-password list including the entire Have I Been Pwned NTLM corpus — `Passw0rd!` would be auto-rejected at change time.">
2. <e.g. "Disable LLMNR (GPO `Turn off multicast name resolution = Enabled`) and NBT-NS on every endpoint — removes the most common path Responder uses to capture NetNTLMv2 in the first place.">
3. <e.g. "Enforce SMB signing on all servers and clients to block relay re-use of any hashes that *do* leak.">

---

## 8. Reflection (≈ 150 words)

- What surprised you?
- If you had to estimate "minimum acceptable password length" based on today's results, what would it be and why?
- What would the workflow look like if hashcat had said "Status: Exhausted" instead of "Cracked"?

---

## Submission checklist (delete before submitting)

- [ ] Hash identification table (§1) is fully filled in, **mode = 5600**
- [ ] Benchmark speed pasted in §2
- [ ] At least one **Status: Cracked** block pasted verbatim
- [ ] Rule and mask attacks both attempted (§§5, 6)
- [ ] Three **specific** defender recommendations (no "use better passwords")
- [ ] `ntlm.txt`, `pass.txt`, and a today-only potfile attached
- [ ] Examples and `<!-- comments -->` removed
```

### 9.2 Grading rubric (10 pts)

| Criterion | Pts |
|---|---|
| §1 Hash correctly identified with five-field rationale | 2 |
| §3 Dictionary attack — command, time, and plaintext shown | 2 |
| §4 `rockyou.txt` scaling — time-to-crack pasted | 1 |
| §5 Rule-augmented attack — command + reasoning | 1 |
| §6 Mask attack — keyspace math shown | 2 |
| §7 Three specific defender recommendations (not generic) | 1 |
| Report readable; artefacts attached; checklist complete | 1 |

---

## 10. Troubleshooting cheat sheet

| Symptom | Likely cause | Fix |
|---|---|---|
| `Token length exception` | Hash pasted as two lines, or stray space, or wrong mode | `wc -l ntlm.txt` should be 1; check `-m 5600` |
| `No hashes loaded` | Wrong mode for the file's format | Re-check §3 — is it really NetNTLMv2? |
| `CL_DEVICE_NOT_FOUND` or no GPU | Driver issues; running in VM | Add `--force`; accept CPU speed |
| Stuck at `Initializing device kernels…` | First-time kernel JIT — patience | Wait 30–60 s. Will be cached for next run |
| `Status: Exhausted` — wordlist exhausted, no crack | Password isn't in this wordlist | Try `rockyou.txt`; then rules; then masks |
| Hashcat finishes instantly without trying anything | The hash is already in the pot file | `hashcat --show -m 5600 ntlm.txt` to see it; or `--potfile-disable` to re-crack |
| Cracked password contains weird characters / mojibake | Wordlist encoding mismatch | Convert wordlist to UTF-8 with `iconv` |
| `hashcat -b -m 5600` reports ridiculously low speed (kH/s on a GPU) | OpenCL kernel running on CPU even though GPU present | Check `hashcat -I` for device #s, then `-d 1` (or 2/3) to pin device |
| `hashcat --show` prints nothing | Pot file is empty / wrong mode | `cat ~/.local/share/hashcat/hashcat.potfile` |
| Mask attack runs forever | Keyspace too large | Re-count your `?l/?u/?d/?s` — shave a character |

---

## 11. Bonus — using John the Ripper as a second opinion

`hashcat`'s sibling, `john`, can crack the same hash with slightly different syntax. Useful when you want to cross-check, or when `hashcat` doesn't recognise an unusual format:

```bash
john --format=netntlmv2 --wordlist=pass.txt ntlm.txt
john --show --format=netntlmv2 ntlm.txt
```

Same answer (`Passw0rd!`), different tool. In a professional report, **showing two tools agreed** is much stronger evidence than one tool's word.

---

## 12. Compressed 2-hour schedule

| Block | Time |
|---|---|
| Theory primer (§1) | 20 min |
| Preflight (§2) | 10 min |
| Phase 0 — identify hash (§3) | 10 min |
| Phase 1 — wordlist (§4) | 10 min |
| Phase 2 — dictionary attack (§5.2 + §5.3) | 25 min |
| Phase 3 — one rule and one mask attack (§§6.1, 6.2) | 20 min |
| Phase 4 — defender's view (§7) | 10 min |
| Report submission | 15 min |
| **Total** | **2 h 00** |

The `rockyou.txt` scaling and full rule/mask exploration become Workshop 4.

---

## 13. Further study

- **Hashcat example hashes** — <https://hashcat.net/wiki/doku.php?id=example_hashes> (the canonical mode lookup)
- **Hashcat rule writing** — <https://hashcat.net/wiki/doku.php?id=rule_based_attack>
- **Mask attack reference** — <https://hashcat.net/wiki/doku.php?id=mask_attack>
- **Responder** (where NetNTLMv2 hashes usually come from) — <https://github.com/lgandx/Responder>
- **HaveIBeenPwned NTLM corpus** — <https://haveibeenpwned.com/Passwords> (download)
- **Specops free password auditor / banned-list** — common enterprise mitigation
- **John the Ripper** — <https://www.openwall.com/john/> (second-opinion cracker)
- After class: load the pcap from a real Responder session in Wireshark and find the `Type 3 Authenticate Message`. Identify each of the five hash fields directly in the bytes.

---

## 14. Appendix A — Consolidated command reference

> Every command students will run today, in execution order. Keep this section open in a second tab.

### A.0 Preflight (§2)

| # | Command | Purpose | Output |
|---|---|---|---|
| 1 | `hashcat --version` | Confirm hashcat ≥ 6.0 | screen |
| 2 | `hashcat -I` | List compute devices visible to hashcat | screen |
| 3 | `hashcat -b -m 5600` | Benchmark NetNTLMv2 specifically | screen — record the H/s figure |
| 4 | `sudo gunzip -k /usr/share/wordlists/rockyou.txt.gz` | Unzip rockyou (keep the .gz) | `rockyou.txt` |
| 5 | `mkdir -p ~/workshops/w03_crack && cd ~/workshops/w03_crack` | Create + enter workspace | new dir |

### A.1 Phase 0 — Identify the hash (§3)

| # | Command | Purpose | Output |
|---|---|---|---|
| 6 | `nano ntlm.txt` (paste single-line hash) | Save the hash | `ntlm.txt` |
| 7 | `wc -l ntlm.txt` | Verify exactly one line | should print `1 ntlm.txt` |
| 8 | `head -c 60 ntlm.txt` | Eyeball the first 60 bytes | starts `IEUser::IE9WIN7:4ee1d2e9...` |

### A.2 Phase 1 — Build the wordlist (§4)

| # | Command | Purpose | Output |
|---|---|---|---|
| 9 | `cat > pass.txt <<'EOF' ... EOF` | Create the 9-line teaching wordlist | `pass.txt` |
| 10 | `wc -l pass.txt` | Verify 9 lines | `9 pass.txt` |

### A.3 Phase 2 — Dictionary attacks (§5)

| # | Command | Purpose | Output |
|---|---|---|---|
| 11 | `hashcat -m 5600 -a 0 ntlm.txt pass.txt` | Straight dictionary attack — should crack `Passw0rd!` in <1 s | Status: Cracked |
| 12 | `hashcat -m 5600 --show ntlm.txt` | Re-display the cracked plaintext from the pot file | `…:Passw0rd!` |
| 13 | `hashcat -m 5600 -a 0 ntlm.txt /usr/share/wordlists/rockyou.txt` | Scale to a real-world wordlist | Status: Cracked |
| 14 | `cat ~/.local/share/hashcat/hashcat.potfile` | Inspect the pot file directly | one line per cracked hash |

### A.4 Phase 3 — Rules and masks (§6)

| # | Command | Purpose | Output |
|---|---|---|---|
| 15 | `hashcat -m 5600 -a 0 ntlm.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule` | Rule-augmented dictionary | ~910M candidates |
| 16 | `hashcat -m 5600 -a 3 ntlm.txt 'Passw?ld!'` | Tiny mask — teaching only; should crack in milliseconds | Status: Cracked |
| 17 | `hashcat -m 5600 -a 3 ntlm.txt '?u?l?l?l?l?d?l?l?s'` | Realistic mask matching the password's shape (slow on CPU!) | Status: Cracked or `q` to quit |
| 18 | `hashcat -m 5600 -a 6 ntlm.txt pass.txt '?d?d?d'` | Hybrid: word + 3 digits | Status: variable |
| 19 | `hashcat --potfile-disable -m 5600 -a 0 ntlm.txt pass.txt` | Force re-cracking for demonstration | Status: Cracked |

### A.5 Optional / auxiliary

| # | Command | Purpose |
|---|---|---|
| 20 | `hashid '<hash>'` | Automated hash-type guess (install: `pip install hashid`) |
| 21 | `john --format=netntlmv2 --wordlist=pass.txt ntlm.txt` | Cross-check with John the Ripper |
| 22 | `john --show --format=netntlmv2 ntlm.txt` | John's equivalent of `hashcat --show` |
| 23 | `rm ~/.local/share/hashcat/hashcat.potfile` | Wipe the pot file (use between engagements, never mid-engagement!) |

### A.6 Methodology summary — the whole workshop in 6 lines

```
Phase 0: identify mode (5600 = NetNTLMv2)        → hashcat wiki example hashes
Phase 1: save the hash                            → ntlm.txt (one line)
Phase 2: prepare wordlist                         → pass.txt, then rockyou.txt
Phase 3: dictionary attack                        → hashcat -m 5600 -a 0
Phase 4: scale with rules and masks               → -r best64.rule  /  -a 3 <mask>
Phase 5: retrieve, verify, document               → --show, write the report
```

---

## 15. Appendix B — Quick reference card

### B.1 The "three NTLMs"

| What | Where | Hashcat mode |
|---|---|---|
| **NTLM hash** (MD4 of password) | `secretsdump`, mimikatz, SAM | **1000** |
| **NetNTLMv1** | wire (rare in 2026) | **5500** |
| **NetNTLMv2** | wire (Responder, SMB relay) | **5600** ← today |

### B.2 Hashcat attack modes

| `-a` | Use it for |
|---|---|
| `0` | Dictionary (wordlist file) — **default** |
| `1` | Combinator (two wordlist files) |
| `3` | Mask / brute force (mask string) |
| `6` | Hybrid: word + mask |
| `7` | Hybrid: mask + word |

### B.3 Mask placeholders

| Mask | Meaning |
|---|---|
| `?l` | lowercase a–z |
| `?u` | uppercase A–Z |
| `?d` | digit 0–9 |
| `?s` | symbol (printable ASCII) |
| `?a` | all of the above |

### B.4 The four commands you will use 90% of the time

```
hashcat -b -m <MODE>                                  # benchmark
hashcat -m <MODE> -a 0 hashes.txt wordlist.txt        # dictionary
hashcat -m <MODE> -a 0 hashes.txt wordlist.txt -r rules/best64.rule
hashcat -m <MODE> --show hashes.txt                   # retrieve cracked
```

### B.5 Methodology in one sentence

> **Identify the mode → start with a tiny wordlist to validate the workflow → scale to rockyou → augment with rules → finish with masks → retrieve with `--show` → write the defender's recommendation.**

---

> **Reminder:** cracked passwords are sensitive material. Once your report is submitted and reviewed, **delete `~/workshops/w03_crack/`** and wipe `~/.local/share/hashcat/hashcat.potfile`. Do not commit either to any repository. The single biggest signal of a professional pentester is **handling other people's secrets with at least as much care as the people who created them.**

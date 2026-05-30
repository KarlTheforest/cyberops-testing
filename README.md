# cyberops-testing

Cybersecurity coursework: lab guides, simulation walkthroughs, defense
implementation, analysis report, and group-presentation script for a
controlled VM-to-VM attack/defense exercise.

## Lab Setup

| Component   | Specification                                    |
|-------------|--------------------------------------------------|
| Attacker VM | Kali Linux — IP: `192.168.56.102`                |
| Victim VM   | Metasploitable 2 / Ubuntu — IP: `192.168.56.101` |
| Network     | VirtualBox Host-Only Adapter (`192.168.56.0/24`) |
| Tools       | Nmap, Hydra, Metasploit, Wireshark, hping3       |

> All activities documented here are performed in a **local, isolated lab**
> environment with **explicit authorization** for educational purposes.
> Never run these techniques against systems you do not own or are not
> authorized to test.

## Documents

All deliverables live in [`docs/`](docs):

| # | Document | Description |
|---|----------|-------------|
| 1 | [Attack Simulation Guide](docs/01-attack-simulation-guide.md)     | Beginner-friendly, step-by-step reconnaissance and access attack walkthrough. |
| 2 | [Analysis Report](docs/02-analysis-report.md)                     | Full report covering malware classification, attack simulations, and defense effectiveness. |
| 3 | [Defense Implementation Guide](docs/03-defense-implementation-guide.md) | Step-by-step hardening of the victim VM (iptables, Fail2Ban, Snort, ACLs, system hardening). |
| 4 | [Presentation Demo Script](docs/04-presentation-demo-script.md)   | Walkthrough script and timing for the group's live demo. |
| 5 | [Access Attack — Windows 7 Walkthrough](docs/05-access-attack-windows7-walkthrough.md) | Tailored access-attack steps based on the actual Windows 7 victim recon results (EternalBlue, Telnet, RDP, VNC, SMB). |
| 6 | [Actual Lab Results](docs/06-actual-lab-results.md) | Complete documentation of what actually happened in the lab: EternalBlue + SSH successes, Telnet/RDP/Beast Trojan failures, evidence collected, and lessons learned. |

## Assignment Coverage

These documents collectively address the assignment tasks:

- Research and threat classification (malware types, attack categories)
- Network attack simulation (reconnaissance, access, DoS/DDoS)
- Defense implementation (firewall, IDS, ACLs, Fail2Ban, hardening)
- Analysis report and group presentation materials

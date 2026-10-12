# 🛡️ Virtual Network Security Audit & Cleartext Traffic Analysis

## 📌 Executive Summary
This project is an internal network security audit performed in an isolated virtual lab environment on VMware Fusion[cite: 4]. Using **Kali Linux** as the auditor host and **Metasploitable 2** as the target system[cite: 4], I conducted host discovery, full service enumeration, and real-time packet analysis[cite: 4, 5, 8]. 

The audit identified multiple critical vulnerabilities, including legacy unencrypted protocols (Telnet) exposing administrative credentials in cleartext, exposed database services, outdated file-sharing daemons, and an unauthenticated root bindshell.

---

## 📐 Lab Architecture & Host Verification

The environment was deployed across an isolated VMware Fusion virtual private network (`172.16.5.0/24`)[cite: 4, 6, 8, 9].

* **Auditor Host (Kali Linux):** `172.16.5.149` (Interface: `eth0`, MAC: `00:0c:29:85:9a:2b`)[cite: 6]
* **Target Host (Metasploitable 2):** `172.16.5.148` (Interface: `eth0`, MAC: `00:0c:29:92:9e:87`)[cite: 9]
* **Analysis Suite:** Nmap v7.99 & Wireshark v4.x

```text
+-------------------------------------------------------------------------+
|                      VMware Fusion Private Network                      |
|                            (172.16.5.0/24)                              |
|                                                                         |
|  [ Kali Linux (Auditor) ] <-------------------> [ Metasploitable 2 ]    |
|     IP: 172.16.5.149                               IP: 172.16.5.148     |
|   (Running Nmap/Wireshark)                       (Vulnerable Target)    |
+-------------------------------------------------------------------------+

Host Interface Verification
To verify network connectivity and obtain base interface parameters before scanning, host interface details were pulled from both endpoints:   

+ 1
Auditor Interface (Kali Linux)	Target Interface (Metasploitable 2)

# <img src="images/kali-ip .png" >  <img src="images/target-ip .png">

🔍 Methodology & Execution
Phase 1: Subnet Host Discovery (Nmap)
To map active devices on the /24 subnet without triggering security blocks, an ICMP/ARP sweep was executed from Kali Linux:   
sudo nmap -sn 172.16.5.149/24
# <img src="images/nmap-sweep .png" >
Result: Discovered target host at 172.16.5.148 with MAC address 00:0C:29:92:9E:87 (VMware OUI).   

Phase 2: Detailed Service & Version Enumeration
A comprehensive service version scan was executed against the target to enumerate running daemons and build information:
sudo nmap -sV -sC -p- 172.16.5.148
 # <img src="images/nmap-scan .png">
## 📊 High-Risk Findings & Vulnerability Inventory

Deep service enumeration on host `172.16.5.148` identified multiple unencrypted protocols, legacy daemons, and unauthenticated listeners.

### Service Vulnerability Matrix

| Port / Protocol | Service | Version / Banner | Key Vulnerability & Risk | Severity |
| :--- | :--- | :--- | :--- | :--- |
| **139 / 445** | Samba | `smbd 3.0.20-Debian` | Remote Code Execution via `username map script` (CVE-2007-2447). | **CRITICAL** |
| **1524** | Bindshell | `Metasploitable root shell` | Unauthenticated backdoor listener offering immediate root access over TCP. | **CRITICAL** |
| **23** | Telnet | Linux telnetd | Transmits administrative credentials and session data in unencrypted text. | **HIGH** |
| **512 / 513** | exec / login | `netkit-rsh` / `rlogind` | Obsolete trust protocols vulnerable to cleartext interception and session takeover. | **HIGH** |
| **2121** | FTP | `ProFTPD 1.3.1` | Legacy file transfer service containing unpatched command execution flaws. | **HIGH** |
| **3306** | MySQL | `MySQL 5.0.51a-3ubuntu5` | Database daemon listening on public interface without network access control. | **HIGH** |

---

### Proof of Concept: Unencrypted Traffic Capture

To confirm the operational risk of running legacy management services (Port 23 Telnet), live network packets were captured on interface `eth0` during an active login attempt.

![Wireshark Telnet Packet Stream](05-wireshark-stream.png)

#### Packet Inspection Notes (`tcp.stream eq 3`):
* **Information Leakage:** The target host reveals host signatures (`metasploitable2`) prior to authentication.
* **Plaintext Credential Exposure:** Both login (`msfadmin`) and password (`msfadmin`) passed across the wire in raw ASCII text, making session hijacking trivial for local network eavesdroppers.

🦈 Packet-Level Inspection & Evidence (Wireshark)
During an active administrative session over Telnet (Port 23), packet capture was recorded on Kali's eth0 interface and inspected using Wireshark.
# Filter applied in Wireshark:
tcp.stream eq 3
Key Traffic Findings:
Cleartext Banner Leakage: The target host explicitly disclosed system parameters and OS details (metasploitable2, DISPLAY.kali:0.0) prior to authentication.   
# <img src="images/wireshark-stream .png">
Credential Sniffing: Administrative username and password inputs (msfadmin / msfadmin) were transmitted in cleartext ASCII across TCP stream 3.   

Eavesdropping Risk: Any adversary positioned on the local segment can passively capture session payloads and compromise administrative credentials without brute-forcing.   

🛡️ Recommended Security Mitigations
Disable Unencrypted Protocols: Decommission Telnet (Port 23) and replace it with encrypted SSH (Port 22) using strong key exchange algorithms.   

Close Unnecessary Backdoors: Terminate the root bindshell listener running on TCP port 1524[cite: 7].
Patch Outdated Daemons: Upgrade Samba 3.0.20 and ProFTPD 1.3.1 to current vendor-supported security builds[cite: 7].
Network Access Control: Restrict database access (MySQL Port 3306) to local loopback (127.0.0.1) or enforce strict firewall ACLs[cite: 7].
# Metasploitable-network-audit
# Metasploitable-network-audit
# Metasploitable-network-audit
# Metasploitable-network-audit

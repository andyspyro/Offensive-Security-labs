# Hack The Box — Blue

## Executive Summary

This lab focused on identifying and exploiting a vulnerable Windows SMB service. The target was identified as Windows 7 Professional SP1 with SMB exposed on TCP/445. Initial SMB enumeration showed five shares and permitted limited guest access, but the accessible shares did not contain useful sensitive data. The important finding was not the share contents; it was the combination of an older Windows host and an exposed SMB service.

I researched the SMB attack surface, verified that the target was vulnerable to MS17-010, and then used the Metasploit EternalBlue module to obtain remote code execution. Because I was working from WSL and the HTB VPN was routed through Windows rather than a local `tun0` interface, I used a Meterpreter bind TCP payload rather than relying on a reverse callback. Successful exploitation resulted in a Meterpreter session running as `NT AUTHORITY\SYSTEM`.

This lab taught me how to connect **port → service → vulnerability → exploit → payload → post-exploitation evidence** rather than treating an open port as a vulnerability by itself.

## Scope

- Platform: Hack The Box
- Machine: Blue
- Environment: Authorized lab
- Operating system identified: Windows 7 Professional SP1
- Primary vulnerable service: SMB
- Primary port: TCP/445
- Vulnerability: MS17-010
- Exploit technique: EternalBlue
- Result: Remote code execution and SYSTEM-level access

No flags, passwords, private keys, or reusable credentials are included in this report.

---

## 1. Reconnaissance

I began with Nmap service enumeration.

```bash
nmap -sC -sV <TARGET_IP>
```

The important ports were:

| Port | Service | Interpretation |
|---|---|---|
| 135/tcp | MSRPC | Microsoft RPC Endpoint Mapper |
| 139/tcp | NetBIOS Session Service | Legacy SMB transport through NetBIOS |
| 445/tcp | Microsoft-DS / SMB | Direct SMB over TCP |
| 49152+ | Dynamic RPC | Windows RPC services using dynamically assigned ports |

The most useful early lesson was that ports are not vulnerabilities. A port tells me **where a network service is listening**. I still need to determine what that service is, how it is configured, and whether a known vulnerability actually applies.

### Mental model

```text
IP address
    ↓
open port
    ↓
service
    ↓
version / configuration
    ↓
possible vulnerability
    ↓
verification
    ↓
exploit
```

---

## 2. SMB Enumeration

TCP/445 was identified as SMB. I used `smbclient` to enumerate shares without supplying a password.

```bash
smbclient -L //<TARGET_IP> -N
```

Five SMB shares were exposed:

```text
ADMIN$
C$
IPC$
Share
Users
```

I then tested accessible shares individually.

```bash
smbclient //<TARGET_IP>/Share -N
smbclient //<TARGET_IP>/Users -N
```

The `Share` share was accessible but effectively empty. The `Users` share exposed standard Windows profile content such as `Default` and `Public`.

### What I learned here

My first reaction was that getting into an SMB share meant I had found the attack path. That was incorrect.

I learned to separate these concepts:

```text
Port open
≠ authenticated access
≠ useful access
≠ vulnerability
≠ compromise
```

SMB was working exactly as a file-sharing service should. Guest access was interesting, but the content I could read did not immediately provide credentials or sensitive files.

That was not wasted work. It eliminated one hypothesis.

---

## 3. Connecting the Service to a Vulnerability

The stronger clue was:

```text
Windows 7 SP1
+
SMB exposed on TCP/445
```

This made historical SMB vulnerabilities worth investigating. Research led to **MS17-010**, a Microsoft security bulletin covering critical SMBv1 vulnerabilities affecting older Windows systems.

Instead of assuming the machine was vulnerable simply because it was Windows 7, I verified the condition directly:

```bash
nmap -p445 --script smb-vuln-ms17-010 <TARGET_IP>
```

The target was reported as vulnerable.

This was the point where the workflow changed from **enumeration** to **exploitation**.

### Key reasoning

```text
445/tcp
    ↓
SMB
    ↓
Windows 7 SP1
    ↓
MS17-010 is a plausible candidate
    ↓
specific vulnerability test
    ↓
target confirmed vulnerable
    ↓
select an exploit designed for MS17-010
```

This was one of the most important lessons from the lab: **the exploit follows the vulnerability, not the port number**.

---

## 4. Exploitation

I started Metasploit:

```bash
msfconsole
```

I selected the EternalBlue module:

```text
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <TARGET_IP>
```

I initially worked through a reverse-payload setup, but WSL did not have its own HTB `tun0` interface because the VPN path existed through the Windows host. That made the callback configuration unnecessarily complicated.

Since WSL could already initiate connections to the target, I used a bind payload instead:

```text
set payload windows/x64/meterpreter/bind_tcp
check
run
```

The distinction became clear:

```text
Reverse payload:
Target → connects back → attacker
Requires a callback address reachable from the target

Bind payload:
Attacker → connects to payload → target
Useful here because WSL already had a route to the HTB target
```

The exploit succeeded and opened a Meterpreter session.

---

## 5. What EternalBlue Was Doing

I did not write EternalBlue. Metasploit automated the exploit logic.

At a high level:

```text
Metasploit connects to SMB/445
        ↓
sends specially crafted SMBv1 traffic
        ↓
triggers the MS17-010 memory-corruption flaw
        ↓
exploit manipulates kernel memory
        ↓
execution is redirected
        ↓
Meterpreter payload runs
        ↓
interactive session opens
```

This helped me understand three terms that I previously mixed together:

### Vulnerability

The programming/security defect already present on the target.

```text
MS17-010-related SMBv1 memory corruption
```

### Exploit

The technique/code that deliberately triggers the vulnerability and turns it into useful control.

```text
EternalBlue
```

### Payload

The code executed after exploitation succeeds.

```text
windows/x64/meterpreter/bind_tcp
```

So I did **not** install the vulnerability. The vulnerable SMB implementation was already present. I used an exploit designed to trigger that existing defect.

---

## 6. Post-Exploitation

Once Meterpreter opened, I verified the security context:

```text
getuid
```

The session was running as:

```text
NT AUTHORITY\SYSTEM
```

This is a highly privileged Windows security principal.

At this point the assessment had changed completely. Before exploitation I was communicating with network services from outside the machine. After exploitation I had command execution inside the Windows operating system.

I navigated the Windows filesystem and verified access to both the standard user and Administrator desktop locations required by the lab.

The actual flag values are intentionally omitted from this repository.

---

## 7. Attack Chain

The complete path was:

```text
Nmap reconnaissance
        ↓
135 / 139 / 445 discovered
        ↓
445 identified as SMB
        ↓
SMB shares enumerated
        ↓
Guest-readable content checked
        ↓
No useful sensitive files found
        ↓
Windows 7 + SMB recognized as a stronger clue
        ↓
MS17-010 researched
        ↓
MS17-010 verified with Nmap
        ↓
EternalBlue Metasploit module selected
        ↓
Meterpreter bind payload configured
        ↓
Remote code execution
        ↓
NT AUTHORITY\SYSTEM
        ↓
User and Administrator proof obtained
```

---

## 8. What I Learned

### Ports are entry points to services, not vulnerabilities

I initially thought of an open port almost like an unlocked door. A better model is:

```text
Port = numbered network entrance
Service = application answering behind that entrance
Vulnerability = defect in that application/configuration
Exploit = method for abusing that defect
```

TCP/445 being open only told me that SMB was reachable.

### Enumeration is about reducing uncertainty

Checking SMB shares that turned out to contain nothing useful was still valid work. The result was:

```text
[+] SMB reachable
[+] Guest/anonymous enumeration possible
[+] Five shares identified
[+] Some shares readable
[-] No useful credential or sensitive-file path discovered there
```

A negative result means I can move to the next hypothesis instead of repeatedly wondering whether the path was missed.

### 139 and 445 are related

I learned that they are not necessarily two independent attack surfaces:

```text
139 → SMB through NetBIOS
445 → SMB directly over TCP
```

Both can lead to the same SMB resources.

### 135 is different

TCP/135 is the Microsoft RPC Endpoint Mapper. It behaves more like a directory of RPC services than a file share.

```text
135 → RPC Endpoint Mapper
         ↓
      dynamic RPC endpoints
         ↓
      49152+
```

This explained why the high-numbered Windows RPC ports appeared in the original scan.

### Research is part of the job

I do not need to memorize every port, CVE, or exploit.

The useful workflow is:

```text
See something
→ identify it
→ research how it normally works
→ enumerate it
→ research applicable weaknesses
→ verify the weakness
→ exploit only when evidence supports it
```

### Exploit development and penetration testing are different disciplines

On this machine I acted primarily as a penetration tester: I found the service, identified and verified a known vulnerability, then used an existing exploit.

Developing EternalBlue from scratch would require a much deeper process involving protocol analysis, debugging, Windows kernel internals, memory corruption, controlled memory manipulation, and exploit reliability engineering.

---

## 9. Defensive Analysis

The fundamental defensive problem was an outdated Windows system exposing a vulnerable SMB implementation.

Key mitigations include:

- Apply the Microsoft security updates associated with MS17-010.
- Disable SMBv1 where it is not explicitly required.
- Upgrade unsupported operating systems such as Windows 7.
- Restrict SMB exposure with host and network firewalls.
- Do not expose TCP/445 directly to untrusted networks.
- Segment systems so workstation/file-sharing traffic cannot freely traverse security boundaries.
- Restrict guest and anonymous SMB access.
- Require appropriate authentication and least-privilege share permissions.
- Monitor for abnormal SMB traffic and exploitation attempts.
- Maintain vulnerability-management processes capable of identifying missing critical patches.

The most important mitigation would have been preventing the vulnerable SMB implementation from remaining exposed and unpatched.

---

## 10. Skills Practiced

- Nmap service and version enumeration
- NSE vulnerability checks
- Windows network-service identification
- SMB enumeration with `smbclient`
- SMB share and permission analysis
- MSRPC/NetBIOS/SMB relationship mapping
- Vulnerability research and prerequisite matching
- MS17-010 validation
- Metasploit Framework
- Payload selection
- WSL networking troubleshooting
- Meterpreter
- Windows post-exploitation navigation
- Windows security-context identification
- Root-cause and remediation analysis

---

## Final Takeaway

The biggest lesson from Blue was not memorizing EternalBlue.

It was learning this chain:

> **A port tells me where a service is. The service tells me what technology I am dealing with. Research tells me what weaknesses may apply. Verification tells me whether the target actually has that weakness. The exploit takes advantage of the verified weakness, and the payload determines what access I receive afterward.**

That reasoning process is reusable even when the ports, operating systems, vulnerabilities, and exploits are completely different.

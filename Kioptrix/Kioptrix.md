## Penetration Test Report
**Target**: Kioptrix Level 1
**Assessment Type**: Black-Box Internal Network Penetration Test
**Date**: July 2026
**Prepared By**: Rajeev Thapa
**Classification**: Confidential

---
### 1. Executive Summary

This report represents the findings of a penetration test conducted against the Kioptrix Level 1 target system. The assessment was performed under black-box conditions, simulating a threat actor with network access but no prior environment knowledge.

The assessment was performed under black-box conditions and identified multiple critical vulnerabilities across several network services, including an unpatched remote code execution vulnerability in the Samba service that allowed full system compromise without authentication. Successful exploitation resulted in root-level access to the target system, complete control over all data and services, and the ability to modify system credentials at will.

Additionally, the web services running on the target were found to disclose sensitive version information through default installation pages and verbose error responses, providing an attacker with significant reconnaissance advantage prior to exploitation.

These findings represent a critical risk to any organization operating a system with this configuration and require immediate remediation.

---
### 2. Scope & Rules of Engagement

| Item                  | Detail                                                                              |
| --------------------- | ----------------------------------------------------------------------------------- |
| Target                | Kioptrix Level 1                                                                    |
| Target IP             | Unknown at engagement start - discovered during assessment                          |
| Identified Target IP  | 192.168.119.128                                                                     |
| Network Range         | 192.168.119.0/24s                                                                   |
| Assessment Type       | Black Box Penetration Test                                                          |
| Credential Assumption | No Credentials assumed - unauthenticated attacker perspective                       |
| Testing Environment   | Isolated lab network (VMware)                                                       |
| Out of Score          | Denial of Service, Social Engineering                                               |
| Tools Used            | Nmap, Netdiscover / arp-scan, Metasploit Framework, Dirb/Ffuf/Gobuster, Web Browser |
The Assessment was conducted with no prior knowledge of the target system's IP address or credentials. The target was first located through local host discovery method (`sudo arp-scan -l`) before any service enumeration or exploitation was attempted. Any credentials discovered during testing were obtained through exploitation, not prior disclosure. Password modification of the user account `john` was performed solely as a proof-of-concept to demonstrate post-exploitation capability and administrative control over the system.

---
### 3. Methodology

The assessment followed a structured penetration testing methodology consisting of the following phases:

```
Phase 0 -> Host Discovery
			└── Network sweep to identify live3 hosts 
				and locate the target system
			
Phase 1 -> Reconnaissance and Enumeration
			└── Port Scanning, Service fingerprinting, web content discovery
				SMB enumeration
				
Phase 2 -> Vulnerability Identification
			└── Service version analysis, CVE mapping, public expoloit research
			
Phase 3 -> Exploitation
			└── Remote Code Execution via Samba trans2open buffer overflow
				(CVE-2003-0201)
				
Phase 4 -> Post-Exploitation
			└── Privilege confirmation, credential modification, 
				system enumeration
			
Phase 5 -> Reporting
			└── Documentation of findings, impact analysis, remediation
				recommendations
```

All testing was conducted in accordance with ethical penetration testing principles. Tools and techniques used are consistent with industry-standard offensive security practices.

---
### 4. Findings Summary

| ID   | Findings                                               | Severity      | CVSS Score |
| ---- | ------------------------------------------------------ | ------------- | ---------- |
| F-01 | Samba Trans2open Stack Buffer Overflow (CVE-2003-0202) | Critical      | 10.0       |
| F-02 | Outdated and Unpatched Operating System                | Critical      | 10.0       |
| F-03 | Apache Version Disclosure via 404 Error Pages          | High          | 7.5        |
| F-04 | Weak Local Account Security                            | High          | 8.1        |
| F-05 | Default Apache Installation Page Exposed               | Medium        | 5.3        |
| F-06 | SMB Version Enumerable via Verbose Response            | Informational | 0.0        |
| F-07 | RPCBind Service Exposed                                | Informational | 0.0        |

---
### 5. Technical Findings
#### F-01: Samba trans2open Stack Buffer Overlflow
```
Severity:        Critical
CVE:             CVE-2003-0201
CVSS Score:      10.0
Affected Host:   192.168.119.128
Affected Port:   TCP 139 (NetBIOS/SMB)
Service:         Samba 2.2.1a
Tool Used:       Metasploit Framework
Module:          exploit/linux/samba/trans2open
```
##### **Description**

A critical remote code execution vulnerability exists in the Samba service running on the target system. Samba version 2.2.1a contains a stack-based buffer overflow in the trans2open transaction handler. An unauthenti9cated remote attacker can send a specially crafted SMB transaction request to overflow the stack buffer, overwrite the return address, and execute arbitrary shellcode with the privileges of the Samba process - which in this configuration runs as root.

This vulnerability requires no authentication, no user interaction, and no special network position beyond TCP connectivity to port 139. It represents a complete and direct path to full system compromise from zero access.

##### **Evidence**

###### <b>Step 1 - Service Identification</b>:
[[Kioptrix/Screenshots/nmap_full_scan.png|Initial port scanning]] confirmed the presence of a NetBIOS/SMB service on TCP port 139. Subsequent enumeration using the Metasploit SMB version scanner (with verbose output enabled) identified the running Samba version:

![[smb_enum.png]]
###### **Step 2 - Exploitation**:
The Metasploit trans2open exploit module was configured with a non-staged 32-bit reverse-shell as a payload (payload/linux/x86/shell_reverse) and executed against the identified target:
![[Exploit.png]]
###### **Step 3 - Root Access Confirmed**
Exploitation was successful, returning a root-level shell on the target system, which can be confirmed from an image in [[#**Step 2 - Exploitation**]] section.

##### **Proof of Exploitation - Password Modification**
To demonstrate complete administrative control over the system, the password for the local user account `john` was modified as a proof-of-concept:
![[john_owned.png|700]]
> Note: Password modification was performed solely as a proof-of-concept demonstrate of post-exploitation capability. In a production engagement, credential modification would only be performed with explicit written client authorization. The account was restored to its original state following the test.
##### Impact
Successful exploitation of this vulnerability results in:
- Complete system compromise - root-level shell access to the target with no restrictions
- Full data exposure - unrestricted read/write access to all files, databases, and configurations on the system
- Credential access - ability to read /etc/shadow and extract all local account password hashes for offline cracking
- Lateral movement - the compromised system can be used as a pivot point to attack other systems on the same network segment
- Persistence - an attacker can install backdoors, create new accounts, or modify system binaries to maintain long-term across
- Service disruption - ability to stop, modify, or destroy any service running on the system

##### Recommendation
###### Immediate Actions:
1. Decommission or isolate this system immediately if it is accessible from any production network segment
2. Upgrade Samba to a currently supported version (6.x or later)
3. Implement network-level firewall rules restricting SMB access (TCP 139, TCP 445) to only explicitly required hosts
**Long-Term Actions:** 
4. Implement a patch management program ensuring all services are kept current with security updates
5. Conduct a network wide audit for additional instances of Samba 2.x or other end-of-life service version 6. Restrict Samba from running as root - configure it to run under a dedicated, least-privileged service account.

---
#### F-02 - Outdated and Unpatched Operating System
```
Severity:            Critical
CVE:                 Multiple (Kernel 2.4.x vintage)
CVSS Score:          10.0
Affected Host:       192.168.119.128
OS Identified:       Linux kernel 2.4.7-10 (September 2001)
```
##### **Description**

The target system is running a Linux kernel released in September 2001 - over two decades past end of life with no available security updates. All software on this system, including the kernel itself, network services, and system utilities, is critically outdated and exposed to a large body of publicly known, weaponized vulnerabilities.

Beyond the Samba vulnerability exploited in F-01, the kernel version (2.4.7) is known to be vulnerable to numerous local privilege escalation exploits, meaning that even a low-privileged user account on this system would likely be able to escalate to root through kernel-level vulnerabilities independently of the Samba path.
##### Evidence
![[os-kernel-proof.png]]
##### Impact
- Local account credentials are fully exposed to an attacker with root access
- Password reuse across systems could enable lateral movement beyond this target
- Account takeover without prior credential knowledge
##### Recommendation
1. Enforce strong password policies across all local accounts
2. Implement centralized credential management
3. Audit all local accounts and remove or disable any not required for system operation
4. Monitor for unauthorized password changes via audit logging

---
#### F-03 - Apache Version Disclosure via 404 Error Pages
```
Severity: High
CVE: N/A (configuration issue)
CVSS Score: 7.5
Affected Host: 192.168.119.128
Affected Ports: TCP 80 (HTTP), TCP 443 (HTTPS)
Service: Apache 1.3.20
```
##### Description
The Apache web server running on the target system returns verbose error pages that disclose the exact server version, operating system, and installed module versions in the HTTP response body. This information was confirmed by navigating to a non-existent resource, triggering a 404 response that disclosed the full software stack.

This information materially reduces an attacker's reconnaissance burden, allowing direct targeting of known vulnerabilities for the identified version rather than requiring broader enumeration.

##### Evidence

![[404_page.png]]
##### Impact
Version disclosure enabled direct identification of:
- Apache 1.3.20 - multiple known remote vulnerabilities
- mod_ssl2.8.4/OpenSSL 0.9.6b - `OpenFuck` (CVE-2002-0082), a known remote exploit against this exact version combination
- PHP 4.0.6 - multiple known remote code execution vulnerabilities
##### Recommendation
1. Disable server version disclosure by setting the following in httpd.conf:
   ```
   ServerTocken Prod
   ServerSignature Off
   ```
2. Upgrade Apache to a currently supported version
3. Implement custom error pages that do not reference server software or version information
---
#### F-04 - Weak Local Account Security
```
Severity:             High
CVE:                  N/A
CVSS Score:           8.1
Affected Host:        192.168.119.128
Affected Account:     john
```
##### Description
Following successful exploitation and root access, a local user account (john) was identified on the target system. The account was accessible and its password was modifiable without knowledge of the existing credential, demonstrating that root-level access grants complete control over all local account security on the system.
In a real-world scenario, an attacker with root access would extract `/etc/shadow` and submit all password hashes for offline cracking, potentially recovering credentials reused across other systems in the environment.
##### Evidence

![[user_john_pwned.png]]
##### Impact
- Local account credentials are fully exposed to an attacker with root access
- Password reuse across systems could enable lateral movement beyond this target
- Account takeover without prior credential knowledge
##### Recommendation
1. Enforce strong password policies across all accounts
2. Implement centralized credential management
3. Audit all local accounts and remove or disable any not required for system operation
4. Monitor for unauthorized password changes via audit logging

---
#### F-05 - Default Apache Installation Page Exposed
```
Severity:           Medium
CVE:                N/A
CVSS Score:         5.3
Affected Host:      192.168.199.128
Affected Ports:     TCP 80 (HTTP)
```

##### Description
The default Apache installation test page is publicly accessible at the root of the HTTP services. This confirms the server is running a default, unconfigured Apache installation. Additionally, web content discovered revealed accessible files including a `test.php` file, suggesting unintended context exposure.
##### Evidence![[default_page.png]]

![[test.php_page.png]]
##### Impact
- Confirms unconfigured/default installation to any visitor
- Discloses operating system (Red Hat Linux)
- Exposed directories may contain sensitive information
- Increases attacker confidence and reduces reconnaissance effort
##### Recommendation
1. Replace default installation page with appropriate content or a blank page
2. Audit and remove unintended web content (test.php)
3. Implement directory listing restrictions in httpd.conf:

---
#### F-06 - SMB Version Enumerable via Verbose Response
```
Severity:         Informational
CVE:              N/A
CVSS Score:       0.0
Affected HOst:    192.168.119.128
Affected Port:    TCP 139
```
##### Description
The Samba service responded to SMB negotiation requests in a manner that allowed identification of the exact service version (2.2.1a) through protocol-level enumeration. While version identification alone does not constitute a vulnerability, it facilitated direct identification of CVE-2002-0201 as documented in F-01.
##### Recommendation
1. Upgrade Samba to a supported version
2. Implement network-level controls restricting SMB access to only required hosts.

---
### F-06 SMB Version Enumerable via Verbose Response
```
Severity: Informational
CVE: N/A
CVSS Score: 0.0
Affected Host: 192.168.119.128
Affected Port: TCP 139
```
##### Description
The Samba service responded to SMB negotiation requests in a manner that allowed identification of the exact service version (2.2.1a) through protocol-level enumeration. While version identification alone does not constitute a vulnerability, it facilitated direct identification of CVE-2003-0201 as documented on F-01.
##### Recommendation
1. Upgrade Samba to a supported version
2. Implement network-level controls restricting SMB access to only required hosts

---
### F-07 - RPCBind Service Exposed
```
Severity: Informational
CVE: N/A (hiscorical: CVE-2000-0666 for rpc.statd)
CVSS Score: 0.0
Affected Host: 192.168.119.128
Affected Ports: TCP/UDP 111, TCP/UDP 32768
```
##### Description
The RPCBind service (portmapper) is running and accessible on TCP/UDP port 111. Enumeration revealed two registered RPC services:

| Program | Service                   | Port  | Protocol |
| ------- | ------------------------- | ----- | -------- |
| 100000  | rpcbind (portmapper)      | 111   | TCP/UDP  |
| 100024  | rpc.statd (status daemon) | 32768 | TCP/UDP  |
RPCBind acts as a service locator (portmapper) for RPC-based services - clients query port 111 to discover the dynamic port assignment of registered services. The `rpc.statd` daemon running on port 32768 is a component of the NFS lock manager suite, responsible for crash recovery coordination between NFS peers.
On systems of this vintage, `rpc.statd` was historically vulnerable to a format string vulnerability (CVE-2000-0666) enabling remote code execution - representing a plausible secondary attack path not pursued during this assessment as root access was achieved via F-01.
#####  Recommendation
1. Disable RPCBind and RPC service if NFS is not required
2. If required, restrict access via firewall rules to only authorized hosts
3. Ensure `rpc.statd` is patched or replaced with a supported NFS stack

---
### 6. Attack Narrative - Kill Chain
The following describes the complete attack chain from initial network access to full  system compromise. The assessment began with zero knowledge of the target environment - no IP address, no service list, no credentials.
```
╔═══════════════════════════════════════════════════════=═══════=═══════=═══╗
║                         PHASE 0 - HOST DISCOVERY                          ║
╚════════════════════════════════════════════════════════════════=═══════=══╝
Objective: Identify the target system's IP address on the network segment with
		   no prior knowledge.
		   
Tool:      netdiscover / arp-scan
Command:   sudo netdiscover -r 192.168.119.0/24
					OR
		   sudo arp-scan -l
		   
Result:    Multiple hosts identified on the network segment. Target system
		   identified at 192.168.119.128 based on subsequent service
		   fingerprinting confirming the expected vulnerable servic profile.
		   
Why this matters:
On a real engagement, host discovery is almost always the first step inside a network. You know you're on the network but you don't know what's there. ARP-based tools like netdiscover and arp-scan are reliable for this because ARP operates at Layer 2 and cannot be blocked by host-based firewalls the way ICMP (ping) can - a host that ignores pings will still respond to ARP requests because it has to in order to function on the network at all.

╔═══════════════════════════════════════════════════════=═══════=═══════=═══╗
║                 PHASE 1 - RECONNAISSANCE AND ENUMERATION                  ║
╚════════════════════════════════════════════════════════════════=═══════=══╝

Objective: Build a complete service map of the target.

Tool:      nmap
Command:   nmap -p- -T4 -A 192.168.119.128

Flags explained (future reference):
-p-       Scan all 65535 ports, not just top 1000
-T4       Aggressive timing (fast, noisy - lab appropriate)
-A        Enables OS detection, version detection, script scanning, and traceroute
		  and traceroute in a single flag.

Result:   7 open ports identified:
  22/tcp    OpenSSH 2.9p2
  80/tcp    Apache 1.3.20
  111/tcp   RPCBind
  139/tcp   NetBIOS-SSN (Samba — version not shown by nmap)
  443/tcp   HTTPS (Apache/mod_ssl 2.8.4/OpenSSL 0.9.6b)
  32768/tcp RPC (rpc.statd)

Web enumeration:
  Browsing to http://192.168.119.128 → Default Apache page
  404 errors → Full version stack disclosed in response body
  Directory fuzzing → test.txt (200), usage/ (200)

SMB enumeration:
  Module: auxiliary/scanner/smb/smb_version (verbose mode)
  Result: Samba 2.2.1a confirmed on TCP 139

RPCBind enumeration:
  Tool:   rpcinfo -p 192.168.119.128
  Result: Program 100000 (rpcbind) on port 111
          Program 100024 (rpc.statd) on port 32768

╔═══════════════════════════════════════════════════════=═══════=═══════=═══╗
║                  PHASE 2 - VULNERABILITY IDENTIFICATION                   ║
╚════════════════════════════════════════════════════════════════=═══════=══╝

Objective: Map identified service versions to known CVEs.

Samba 2.2.1a → CVE-2003-0201 (trans2open buffer overflow)
  Public Metasploit module: exploit/linux/samba/trans2open
  Authentication required:  None
  Complexity:               Low (automated via Metasploit)
  Impact:                   Remote code execution as root
  
mod_ssl 2.8.4 / OpenSSL 0.9.6b → CVE-2002-0082 (OpenFuck)
  Note: Secondary attack path identified but not pursued
  as trans2open provided direct root access
  
╔═══════════════════════════════════════════════════════=═══════=═══════=═══╗
║                          PHASE 3 - Exploitation                           ║
╚════════════════════════════════════════════════════════════════=═══════=══╝

Objective: Exploit CVE-2003-0201 to achieve remote code execution on the target
		   system.
Tool:      Metasploit
Module:    exploit/linux/samba/trans2open
Payload:   linux/x86/shell_reverse_tcp
Result:    Root shell returned after buffer overflow

Confirmation: 
  whoami     → root
  uname -a   → Linux kioptrix.level 1 2.4.7.10
  hostname   → kioptrix.level 1
  
╔═══════════════════════════════════════════════════════=═══════=═══════=═══╗
║                       PHASE 4 - POST-EXPLOITATION                         ║
╚════════════════════════════════════════════════════════════════=═══════=══╝

Objective: Demonstrate impact of root-level compromise.

Actions taken:
  → System information gathered (uname -a, hostname, whoami)
  → Local user enumeration (/etc/passwd)
  → Password modification of john account (PoC only)
  
Actions available but not taken (future reference):
  → /etc/shadow extraction and offline hash cracking
  → SSH key installation for persistent access
  → Network pivot to other hosts on the segment
  → Full filesystem exfiltration
```
Total authentication required at any stage: None Attack path complexity: Low-single automated exploit Time from host discovery to root shell: `7 days`

---
### 7. Recommendations Summary

| Priority   | Action                                | Finding    |
| ---------- | ------------------------------------- | ---------- |
| Immediate  | Isolate or decommission this system   | F-01, F-02 |
| Immediate  | Upgrade Samba to supported version    | F-01       |
| Immediate  | Upgrade all end-of-life software      | F-02       |
| Short-term | Disable Apache version disclosure     | F-03       |
| Short-term | Implement patch management program    | F-02       |
| Short-term | Remove default pages and test content | F-05       |
| Short-term | Audit and harden local accounts       | F-04       |
| Long-term  | Restrict SMB/RPC access via firewall  | F-01, F-07 |
| Long-term  | Implement network segmentation        | All        |

---
### 8. Conclusion

The Kioptrix Level 1 system presents a critical security posture. Beginning with zero knowledge of the target environment, host discovery located the system within minutes of joining the network segment. A single unauthenticated connection to TCP port 139 was then sufficient to achieve complete root-level compromise using a publicly available, automated exploit - with no credentials, no user interaction, and no special access required at any stage.

The root cause is not a single misconfiguration but a systemic failure of patch management - every service identified on this system is critically outdated, collectively representing a large and well-documented body of public exploits.  Remediation of any individual finding in isolation would provide minimal security improvement without addressing the underlying lifecycle management failure.

The immediate recommendation is isolation or decommissioning of this system, followed by replacement with a currently supported, hardened equivalent.

---
###                                      9. Personal Study Notes

---

_(This section is for your own reference — remove before submitting to a real client)_
## What I Learned on This Box
```
Host Discovery:
- ARP-based tools (netdiscover, arp-scan) work even when
  ICMP is blocked because hosts must respond to ARP
- Always do host discovery before port scanning in a
  real engagement — you won't always know the target IP

Enumeration:
- nmap -p- scans all 65535 ports — slower but thorough
- -A flag combines OS detection + version + scripts in one
- SMB version may require verbose mode in MSF to display
- RPCBind (port 111) is a service locator — program numbers
  100000 = rpcbind itself, 100024 = rpc.statd

Exploitation:
- trans2open is non-deterministic — may need multiple runs
- linux/x86/shell_reverse_tcp (non-staged) is more reliable
  than linux/x86/shell/reverse_tcp (staged) on old targets
- MSF version matters — 6.4.135 works, 6.4.145 broke this
- Hold MSF version with: sudo apt-mark hold metasploit-framework

Post-Exploitation:
- Always whoami + uname -a immediately after shell
- Take VMware snapshot the moment shell appears
- Screenshot before touching anything else

Tool Issues Encountered:
- SMB version blank without verbose → set VERBOSE true in MSF
- trans2open stopped working after Kali upgrade → MSF version issue
- Invisible mouse on fresh Kali → hardware compatibility level
  issue, fixed by upgrading VMware HW version or ISO install
```
## Secondary Attack Paths Identified (Not Exploited)

```
1. OpenFuck — CVE-2002-0082
   mod_ssl 2.8.4 / OpenSSL 0.9.6b
   Remote exploit via HTTPS (port 443)
   Tool: searchsploit openssl 0.9.6 / OpenFuck exploit

2. rpc.statd format string — CVE-2000-0666
   rpc.statd on port 32768
   Remote exploit via RPC
   Tool: historical PoC exploits

Both would have achieved the same root access via different paths.
Try these next time for practice on additional attack vectors.
```
## Things to Practice on the Next Box

```
→ Manual enumeration without Metasploit (pure nmap + searchsploit)
→ Exploit a secondary path (OpenFuck or rpc.statd)
→ Post-exploitation: extract /etc/shadow, crack hashes
→ Post-exploitation: establish SSH persistence
→ Time the full engagement start to finish
→ Complete the report within 24 hours of the engagement
```

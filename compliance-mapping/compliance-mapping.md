# Compliance Mapping — Security Events to Frameworks

## PCI-DSS v4.0 Mapping

| PCI-DSS Req | Requirement Description | Wazuh Detection Method | Attack Simulated |
|-------------|------------------------|------------------------|------------------|
| 2.2 | Develop configuration standards for all system components | CIS Benchmark SCA scan (30% score) | Automated scan |
| 2.2.2 | Enable only necessary services | SCA check for running services | Automated scan |
| 2.2.3 | Implement security features for insecure services | SCA protocol and service checks | Automated scan |
| 2.2.4 | Configure system security parameters | SCA hardening parameter checks | Automated scan |
| 5.1 | Deploy anti-virus/anti-malware on all systems | EICAR test file detection (Rule 100004) | EICAR drop in /tmp |
| 7.1 | Limit access to system components on need-to-know basis | Privileged access attempt alerts (Rule 100002) | su root with wrong password |
| 8.1.5 | Manage IDs used by third parties | Unauthorized sudo detection (Rule 100006) | sudo cat /etc/shadow |
| 10.2.4 | Log invalid logical access attempts | SSH brute force / failed login alerts (Rule 100001) | Hydra SSH attack |
| 10.5 | Secure audit trails so they cannot be altered | File integrity monitoring on log files (Rule 100003) | Modify /etc/hosts |
| 10.6.1 | Review logs at least daily | Automated alert generation + Wazuh dashboard | Continuous monitoring |
| 11.4 | Use IDS/IPS to detect and/or prevent intrusions | Port scan / network recon detection (Rule 100005) | Nmap SYN scan |
| 11.5 | Deploy change-detection mechanism (FIM) | FIM on /etc, /bin, /sbin directories | touch /etc/test-tamper |

## ISO 27001:2022 Annex A Mapping

| Control ID | Control Name | Wazuh Detection Method | Attack Simulated |
|-----------|-------------|------------------------|------------------|
| A.9.2.3 | Management of privileged access rights | Sudo abuse / root access alerts | su root / unauthorized sudo |
| A.9.2.6 | Removal or adjustment of access rights | Unauthorized privilege escalation | sudo to restricted commands |
| A.9.4.2 | Secure log-on procedures | Failed authentication monitoring | SSH brute force |
| A.12.2.1 | Controls against malware | EICAR / malware signature alerts | EICAR test file |
| A.12.4.1 | Event logging | Centralized log collection via Wazuh agent | All log types collected |
| A.12.6.1 | Management of technical vulnerabilities | Vulnerability assessment module | Automated scan |
| A.13.1.1 | Network controls | Port scan / recon detection | Nmap scan |
| A.14.2.5 | Secure system engineering principles | CIS Benchmark compliance checks | SCA automated scan |

## MITRE ATT&CK Mapping

| Tactic | Technique ID | Technique Name | Wazuh Rule Group | Simulated Attack |
|--------|-------------|----------------|-------------------|------------------|
| Credential Access | T1110 | Brute Force | authentication_failures | Hydra SSH brute force |
| Privilege Escalation | T1548 | Abuse Elevation Control | privilege_escalation | Unauthorized sudo |
| Reconnaissance | T1046 | Network Service Scanning | recon, network_scan | Nmap port scan |
| Defense Evasion | T1036 | Masquerading | malware | EICAR test file |
| Impact | T1565 | Data Manipulation | file_integrity | /etc/hosts modification |
| Persistence | T1547 | Boot/Logon Autostart Execution | rootcheck | Hidden file in /tmp |

## CIS Benchmark Results

| Benchmark | Version | Passed | Failed | Not Applicable | Score |
|-----------|---------|--------|--------|----------------|-------|
| CIS Debian Linux 7 | v1.0.0 | 44 | 101 | 10 | **30%** |

### Key Failed Controls

- AppArmor not activated
- nodev option missing on /home partition
- nodev option missing on /run/shm partition
- rsyslog not configured for remote log reception
- Various service hardening checks failed

> Note: Low score is expected in a lab environment. In production, these controls would be remediated as part of system hardening.

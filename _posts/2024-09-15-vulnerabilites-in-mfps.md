---
title: Beyond Paper Jams - Managing Vulnerabilites in MFPs
description: MFPs are more than just office tools - they can be a security risk. Let’s take a closer look at how to identify, scan, and mitigate vulnerabilities to keep network secure.
date: 2024-09-15 18:00:00 +0000
#future: true
media_subpath: /assets/media/posts/2024-09-15-vulnerabilites-in-mfps/
image: vulnerabilites-in-mfps.jpg
categories: [Posts, Vulnerability Management]
tags: [scanning, mfp, scripting]
---

Multifunction printers (MFPs) are often overlooked in vulnerability management. During a network breach, attackers don’t limit themselves to high-value assets like servers or databases - they look for any system that can provide **valuable information** or help them escalate privileges. **Multifunction printers (MFPs)** often fall into this category because they are:
- Less protected – MFPs are often **not included** in security monitoring and vulnerability management programs.  
- Weakly monitored – Many organizations **lack logging and alerting** for printer-related activities.  
- Storage of sensitive data – MFPs can contain **scanned documents, print job histories, and cached credentials**, all of which may help attackers gain further access.

A compromised MFP can serve as a pivot point for an attacker, making it a potential entryway for further exploitation. This is why securing MFPs should be a critical component of any modern vulnerability management strategy.

### Common MFP Vulnerabilities

We should keep in mind that MFPs are often used for scanning documents to network shares or email inboxes. This requires integration with Active Directory or another authentication database, meaning that **credentials may be stored** on these devices.

To effectively secure multifunction printers, organizations should follow the **same structured vulnerability management process** as they do for other IT assets.

When it comes to MFPs, we should take into account the following common security issues:
-   **Default or Weak Credentials** – It is often possible to find an MFP connected to the network without a password configured for the management web interface or using a default password that is easy to guess.
-   **Unsecured Network Protocols** – Protocols like SNMP v1/v2 or Telnet have known security vulnerabilities and can be exploited in an attack.
-   **Buffer Overflow and Remote Code Execution (RCE) Vulnerabilities** – These risks may arise due to unpatched or outdated software versions, making MFPs an easy target for exploitation.

### Asset Discovery & Inventory

It is important to maintain an **updated inventory** of MFPs, including the model, firmware version, and connectivity status.

To identify all MFPs in the network, we can use **traffic analysis** with protocols like **NetFlow** and **analyze DHCP logs** to search for relevant MAC addresses.

Another effective method is **network scanning**. In addition to **vulnerability scanners**, we can use simpler solutions like **network scanners**. For example, [Nmap](http://www.nmap.org) can be utilized to scan the network and gather valuable information about connected MFPs.

The following command will scan the network using Nmap and execute a script to retrieve the title header of web applications, helping us identify the device vendor:
```bash
sudo nmap -P0 -vv -sS -script=http-title.nse -p 80,443 192.168.12.0/24
```

Many MFPs have the SNMP protocol enabled by default. The command below will scan the network, connect to the SNMP service, and retrieve information using the `snmp-info.nse` script.
```bash
sudo nmap -P0 -sU -script=snmp-info.nse -p 161 -vv 192.168.21.0/24
```

It is also a good idea to categorize certain MFPs as **critical assets** if they handle sensitive information.


### Vulnerability Assessment

We can perform **regular vulnerability scans** using a scanner like [Nessus](https://www.tenable.com/products/nessus), for example. Unfortunately, this is not always possible because printers are fragile devices, and in some cases, scanning them or sending data to certain ports may cause malfunctions or even disrupt their functionality.

An alternative approach to assessing vulnerabilities in MFPs is to **monitor vendor security bulletins for updates and CVE disclosures**.

#### Check for misconfigurations and vulnerabilities using self-crafted scripts.

As an additional step, we can create our own script if a vulnerability and its exploit are publicly available. For example, I wrote a [Python script](https://github.com/ErSilh0x/scripts_vulncheck/tree/main/scan_kyocera_cve_2022_1026) that scans an IP network for the **[CVE-2022-1026](https://www.rapid7.com/blog/post/2022/03/29/cve-2022-1026-kyocera-net-view-address-book-exposure/)** vulnerability in Kyocera devices. I used a publicly available exploit, modified it, and integrated it into the scanning script.
```bash
python3 scan_kyocera.py 192.0.2.0/25
```

```python
from csv import reader, writer
import socket
import warnings
import argparse
from exp_kyocera import cve_kyocera
import ipaddress

warnings.filterwarnings('ignore', message='Unverified HTTPS request')

#######
#
#Port TCP 9091 should be accessible
#In address book of kyocera device there should be at least one record for exploit to work
#To run: python3 scan_kyocera.py 192.0.2.0/25
#
##################

# Save results
def write_results(inlist, resultname):
    fileResults = open(resultname + '.csv', mode='w', encoding='utf8', newline='')
    resultwriter = writer(fileResults, delimiter=',')
    resultwriter.writerows(inlist)
    fileResults.close()


#Args
parser = argparse.ArgumentParser()
parser.add_argument("network", help="network to scan with CIDR, example: 192.0.2.0/25")
args = parser.parse_args()

net = args.network
result_data = [['#', 'IP', 'Result']]
target_network = ipaddress.ip_network(net)
socket.setdefaulttimeout(1) #Timeout in seconds to wait on socket - 9091
n = 0
for t in target_network.hosts():
    n += 1
    target = str(t)
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    isopen = s.connect_ex((target, 9091))
    if isopen == 0:
        result = cve_kyocera(target)
    else:
        result = 'Timeout'
    result_data.append([n, target, result])

#Write results
write_results(result_data, 'results')

#Print vulnerable results
print(result_data[0:1])
for r in result_data[1:]:
    if r[2] == 'Vulnerable':
        print(r)
```

There was no other way to check all the devices for this vulnerability because the vulnerability scanner could not detect it.

### Final Thoughts

MFPs are often overlooked in cybersecurity strategy, yet they can be a critical weak link in corporate networks. These devices should be included in the risk prioritization process, where the impact and exploitability of detected vulnerabilities are assessed, and patches are prioritized accordingly.

The typical remediation & hardening process should include:
-   **Change default credentials** and enforce **role-based access control (RBAC)** if supported.
-   Disable **unnecessary services** like **Telnet, FTP, SNMP v1/v2, and web interfaces** whenever possible.
-   Apply **firmware updates and security patches** as soon as they become available.
-   Enable **secure communication protocols** (**HTTPS, SFTP, SNMP v3**) for necessary services.
-   Restrict **physical and remote access** to authorized personnel only.
-   If **syslog** is available, configure **log monitoring and auditing** with a **SIEM solution**.


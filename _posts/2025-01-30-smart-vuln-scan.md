---
title: Scanning Smart - How to Detect Vulnerabilities Efficiently
description: How to Perform Vulnerability Scanning Smartly- Less Noise, More Accuracy, and Maximum Asset Coverage.
date: 2025-01-30 22:00:00 +0000
#future: true
media_subpath: /assets/media/posts/2025-01-30-smart-vuln-scan/
image: smart-vuln-scan.jpg
categories: [Posts, Vulnerability Management]
tags: [scanning, vulnerabilities]
---

Vulnerability scanning is one of the key stages in the vulnerability management process. At first glance, it may seem like a straightforward task, but there are many nuances that can impact the accuracy and efficiency of this process. In this article, I will share my experience, discuss the main challenges, and suggest strategies to make scanning more effective and result-oriented.

## Critical Factors to Consider First

#### Automated Asset Discovery: How to Eliminate Blind Spots in Network

It is crucial to conduct a **comprehensive asset discovery** before starting vulnerability scanning. The discovery process should account for all types of assets, as vulnerabilities  exist not only on servers and workstations but also in less obvious devices.
-   **Workstations**
-   **Servers**
-   **Network devices** (switches, routers, access points)
-   **Security appliances** (firewalls, IDS/IPS, VPN gateways)
-   **Databases** (Oracle, MySQL, PostgreSQL, MS SQL)
-   **Web applications** (both external and internal)
-   **Telephony and VoIP systems**
-   **IoT, MFPs, surveillance cameras**
-   **Cloud services and containerized environments (Docker, Kubernetes)**

The more precise the asset discovery, the more effective the scanning process will be. Missed assets can become entry points for attackers.

>Asset discovery closely aligns with attack surface mapping. The more detailed the inventory, the more use cases it can support in the future, including other SOC processes.

Vulnerability scanners can be used as one of the tools for asset discovery, but relying solely on them is insufficient. They may miss certain assets, especially if devices do not respond to ICMP requests or use non-standard configurations. This is why it is crucial to apply a comprehensive approach to asset discovery and management.

Additional information can be gathered from:
- DNS and DHCP logs – help identify devices that were not detected by vulnerability scanners.
- Active Directory – data can be extracted using **[SharpHound](https://github.com/SpecterOps/SharpHound)** and analyzed in **[BloodHound](https://github.com/SpecterOps/BloodHound)**.
- IPAM (IP Address Management) – IP address management systems provide valuable insights into network resources, including devices that may not appear in other asset discovery sources.
- Network device configurations – can reveal unmanaged assets, undocumented connections, and additional subnets or interfaces that should be included in scanning tasks.

#### Asset Classification by OS and Access Methods

When performing vulnerability scanning, it is essential to consider the asset category and the available access methods.

For example, servers can run on different operating systems (Windows, Linux, BSD, Solaris, etc.), and each requires a dedicated scanning profile tailored to its specific characteristics.

Proper asset classification helps logically group devices, which in turn allows you to:
-   Create accurate scanning policies and profiles
-   Avoid unnecessary system load
-   Consider the specifics of each asset type and minimize the risk of missing vulnerabilities

![assetmindmap](MMapb.png)


#### Asset Classification by Risk Level: Priorities in Vulnerability Management

This classification can be used to prioritize vulnerability remediation and plan scanning strategies accordingly. Priorities are typically determined based on the following factors:
-   **Value of the information** - processed on the asset (e.g., databases containing personal or financial data).
-   **Criticality to system resilience** – if the host is a single point of failure, its compromise could lead to significant disruptions.
-   **Network location** – how easily an attacker can access the vulnerable server (e.g., external perimeter, internal network, or a high-security segment).

Additionally, among my tactical risk assessment metrics, I consider the following:
-   **Ease of exploitation** – how easily a vulnerability can be leveraged in an attack (availability of public exploits, required technical expertise).
-   **Exploit delivery methods** – whether local access is required, if the attack can be conducted remotely, or if exploitation is possible through phishing or malicious documents.
-   **Value of stored information from an attacker’s perspective** – how useful the data on this asset is to an adversary (credentials/tickets in memory, access to other systems).

>Workstations are often given a lower priority than servers, even though they can be significantly easier to compromise. I think this is a mistace.

Sometimes, a workstation may contain more valuable information than a server. For example, if it belongs to an IT department employee whose account has elevated privileges in the domain. Workstation often serves as the first foothold for attackers to escalate privileges or move laterally in the network.

#### Asset Classification by Functionality

Understanding an asset’s functionality, its purpose, and typical behavior on the network is essential. For example, a server is likely to run 24/7 and provide continuous services, while a workstation may be turned off outside of working hours. These differences impact when and how assets should be scanned, what access methods are available, and how critical the scanning windows are.

By grouping assets based on their function, you can tailor scanning policies more effectively, reduce false negatives, and avoid unnecessary strain on systems during operational hours.


#### Asset Location Matters

The location of an asset plays a crucial role, and there are two types of location should be considered:
-   **Network location** – defines which network segment the asset belongs to (external, internal, isolated).
-   **Geographical location** – the physical placement of the asset. This is particularly important for branch offices, remote locations, and cloud environments, as different regions may have varying risks and regulatory requirements.

If the scanning schedule is set according to **GMT+3**, based on the headquarters' time zone (where the scanner server and you are located), but a remote host (e.g., a workstation) is in a different time zone, there is a high chance that the scanner will miss it.

If the workstation is turned off at night and the scan is scheduled for a time when the host is unavailable, this will result in **recurring missed vulnerabilities** in that region.

>My experience over the past year has shown that adjusting the scanning schedule to align with local time and working hours increased workstation coverage by 29% during the peak month.

Another example: a server is located in the DMZ zone. We can scan it using an agentless audit task or an agent-based solution, but since some services are accessible from the Internet and the DMZ zone may have routing to the external network, it is useful to conduct an **external scan** to verify which services are actually exposed.
It is important to consider that even if a service does not have known vulnerabilities, some default configurations can lead to server compromise, potentially threatening the entire network.


## Vulnerability Scanning Methods

There are various methods of scanning hosts, each with its own advantages and disadvantages. The main ones include:
-   Audit scanning with agent
-   Audit scanning without agent
-   Traffic sniffing and analysis
-   Scanning in "pentest" mode

Each of these methods has its own strengths and weaknesses, complementing one another to achieve more comprehensive coverage and visibility. Therefore, it is recommended to use them in combination, adapting to specific assets while considering available resources and  capabilities.

#### Audit scanning with agent

In this type of scanning an agent is installed on the target host to collect data and send it to the scanner.

Advantages:
-   Comprehensive system information – data about installed packages, configurations, processes and user privileges.
-   Real-time change monitoring – the agent detects new vulnerabilities as soon as they appear.
-   Works without a constant network connection – data is synchronized on schedule, which is useful for laptops and remote devices.

Disadvantages:
-   Not all systems support agent-based scanning – for example, older Linux distributions, IoT devices, and embedded systems.
-   Requires installation of additional software, which may take time and administrative effort.
-   Increases the load on endpoint devices.

Best use cases:
- Remote clients and laptops – scanning is performed even with unstable network connection.
- User workstations – provides detailed vulnerability analysis without requiring a constant network connection.
- Cloud resources (AWS, Azure, GCP) – many cloud platforms support agent-based scanning for in-depth configuration analysis.

#### Audit scanning without agent

In audit-based scanning without agent, the scanner connects to the host over the network (typically using SSH, WMI, or RPC) and analyzes its services and configurations. Some scanners may temporarily upload and execute a file or script to gather additional information.

Advantages:
-   No need to install software on each system, simplifying deployment.
-   Lower impact on endpoint devices compared to the agent-based method.
-   It is possible to scan devices where agents cannot be installed, such as network equipment.

Disadvantages:
-   May provide incomplete information depending on the scanner’s capabilities and access level.
-   Can be blocked by firewalls and security systems, requiring prior network configuration.
-   May require weakening host security, such as enabling remote registry access in Windows.
-   Requires proper configuration for access protocols (SSH, WMI, RPC).

Best use cases:
-   Ideal for scanning servers and hosts within the network where access and security policies can be configured.
-   Not suitable for remote clients, especially hosts located behind NAT or VPN.


#### Traffic sniffing and analysis

There are solutions that analyze network traffic and identifying vulnerabilities on hosts without actively interacting with them. Data obtained through this method should be correlated with results from other scanning techniques to minimize false positives.

Advantages:
-   Does not create additional load on the network and endpoint devices.
-   Does not require software installation on hosts.
-   Helps identify hidden vulnerabilities and devices.

Disadvantages:
-   Does not always provide a complete picture, especially if traffic is encrypted.
-   Depends on the availability of network traffic and requires proper placement of sensors.
-   May miss application-level vulnerabilities if services do not transmit critical information in plaintext.
-   Higher risk of false positives.

Best use cases:
-   Within internal networks, especially for monitoring unmanaged devices (IoT, legacy systems, shadow IT).
-   As a supplementary analysis method that enhances active and agent-based scanning but does not replace them.



#### Scanning in pentest mode

When using the pentest method, the scanner connects to accessible services on the host, gathers information about their versions, and analyzes them for vulnerabilities. If authentication is supported, credentials can be provided to obtain more detailed information.

Advantages:
-   Identifies real attack vectors from an attacker's perspective.
-   Checks not only known vulnerabilities but also misconfigurations, weak passwords, and business logic flaws.

Disadvantages:
-   Higher risk of false positives.
-   Greater chance of missing vulnerabilities compared to audit-based scanning.
-   Often does not detect local privilege escalation vulnerabilities.

Best use cases:
-   Perimeter network scanning to determine which vulnerabilities are actually exposed externally.
-   Inter-segment connection analysis to verify firewall configuration accuracy.

It is important to note that pentest scanning does not replace traditional vulnerability scanning but rather complements it.


## Optimizing Vulnerability Scans for Better Coverage

The configuration parameters for scanning tasks depend on the selected scanning method and asset type. Below, I will summarize the key requirements for different categories. More detailed settings will vary depending on the specific solution and the capabilities of the systems used.

Key parameters to consider:
-   **Scanning method:** agent-based audit, agentless audit, pentest, passive monitoring
-   **Scanning frequency:** daily, weekly, monthly
-   **Scanning schedule:** daytime/nighttime, weekdays/weekends

These parameters help optimize the scanning process by reducing network and system load while improving the accuracy and timeliness of vulnerability detection.


#### Perimeter Network Scanning

The perimeter is not limited to publicly accessible IP address blocks. It also includes **wireless networks** and to some extent remote clients that connect to the corporate network.

Ideally, only essential services such as web applications, SMTP, DNS, and VPN should be accessible from the perimeter. However, in practice, it is common to find open RDP, SSH, web-api and even Telnet, which significantly expands the attack surface.

The perimeter is the first line of defense and requires continuous monitoring. It should be scanned internally using audit methods and externally using a pentest approach to identify potential threats and vulnerabilities that attackers could exploit.

Additionally, external reconnaissance services such as [Shodan](https://www.shodan.io/) and [Censys](https://censys.com) can be used to detect exposed services and potentially vulnerable systems. As an alternative, Nmap can be used to analyze open ports and manually review service responses.

-   **Scanning method**: agentless audit, pentest
-   **Scanning frequency**: weekly, monthly
-   **Scanning schedule**: daytime/nighttime, weekdays/weekends


#### Remote Asset (Workstation) Scanning

Remote clients are typically outside the local network and operate autonomously, making centralized monitoring challenging. The most effective way to scan them is through an **agent-based approach**, which enables regular vulnerability assessments even when the device is outside the corporate infrastructure.

However, in some cases, **agentless audit scanning** can be performed when a remote workstation connects to the corporate network via VPN. At this point, an audit can be conducted using standard remote scanning methods, similar to internal hosts.

-   **Scanning method**: agent-based audit, agentless audit (when connected via VPN)
-   **Scanning frequency**: daily, weekly
-   **Scanning schedule**: daytime, weekdays


#### Wireless Network Asset Scanning

In wireless networks, an **agent-based solution** is the most effective approach for vulnerability scanning. Additionally, **agentless audit** scanning can be performed from within the network by placing the scanner in the appropriate segment.

Scanning should only be conducted in wireless networks that contain corporate workstations, as they represent a potential attack vector. Avoid scanning guest networks, corporate devices should not be connected to open or public Wi-Fi networks, except for remote clients using a secure VPN connection.

-   **Scanning method**: agent-based audit, agentless audit, passive monitoring
-   **Scanning frequency**: daily, weekly
-   **Scanning schedule**: daytime, weekdays

#### Internal Server Scanning

Server scanning is not a complex case, as servers typically operate **24/7**, ensuring they remain accessible for assessments. This eliminates the risk of missing assets due to downtime. However, when planning scan schedules, it is essential to consider **maintenance windows** to avoid unnecessary load on critical services during peak operation periods.

-   **Scanning method**: agentless audit
-   **Scanning frequency**: daily, weekly, monthly
-   **Scanning schedule**: nighttime/daytime, weekdays/weekends

#### Internal Workstation Scanning

Workstation scanning comes with specific challenges. In **agentless audits**, workstations may be powered off outside working hours, so scan scheduling should align with **business hours and the time zone** of the office where the workstation is located. This minimizes the risk of missing assets and improves the efficiency of vulnerability detection.

Additionally, some vulnerability scanner vendors recommend disabling certain Windows security mechanisms to improve scan accuracy. While this can enhance vulnerability detection, it also increases the risk of compromise.

The account used for agentless scanning **often has elevated privileges**, making it a potential attack target. To mitigate risks, it is essential to:
-   Separate scanning accounts for workstations and servers
-   Minimize domain account privileges, restricting access to only necessary resources

>Excessive domain account privileges include membership in privileged groups and permissions to read and write attributes (such as _ms-MCS-AdmPwd, msDS-KeyCredentialLink, msDS-GroupMSAMembership_) of Active Directory objects through **DACLs**.

>For a deeper understanding, the [Active Directory Penetration Tester](https://referral.hackthebox.com/mz8tyJq) courses from **Hack The Box** provide hands-on practice in identifying and exploiting weak access control configurations in Windows domains.

-   **Scanning method:** agent-based audit, agentless audit, passive monitoring
-   **Scanning frequency:** daily, weekly
-   **Scanning schedule:** daytime, weekdays


#### Network Device Scanning

Network device scanning is most commonly performed using **agentless auditing**. The scanner connects to the device, executes a series of commands, extracts the configuration, and analyzes it for vulnerabilities and misconfigurations. The **pentest method** is not allways effective in this case.

-   **Scanning method:** agentless audit
-   **Scanning frequency:** weekly, monthly
-   **Scanning schedule:** nighttime/daytime, weekdays/weekends

#### Multifunction Printer (MFP) Scanning

Printers and similar devices may not handle network requests properly and could become unstable. In most cases, they are **not supported** by standard vulnerability scanners, and scanning them is **not recommended**, as it may cause malfunctions.

There are few standardized security solutions for these devices, but a custom approach is possible. For example, using Python, it is possible to create a script that checks a printer for a specific vulnerability, provided that the vulnerability details and a corresponding exploit are available.

However, the most reliable protection method remains **firmware updates**, as most attacks on these devices stem from outdated software and weak configurations.

-   **Scanning method:** pentest, passive monitoring
-   **Scanning frequency:** weekly, monthly
-   **Scanning schedule:** daytime, weekdays


#### Scanning Unsupported Devices

Unsupported devices cannot be audited using standard vulnerability scanners, as they lack integration with traditional scanning tools. In such cases, the best approach is to use pentest methods and custom scripting, accessing the device via known protocols and analyzing its configuration manually.

Additionally, it is crucial to monitor vendor security feeds and track the emergence of new vulnerabilities. This proactive approach enables timely mitigation, even when standard security tools do not support scanning these devices.

Examples of vulnerability feeds resources:
-   [Rapid7 Vulnerability Insights](https://www.rapid7.com/db/)
-   [NVD (National Vulnerability Database)](https://nvd.nist.gov/)
-   [Mitre CVE](https://cve.mitre.org/)
-   [Exploit Database](https://www.exploit-db.com/)
-   [Vulners.com](https://vulners.com/)

-   **Scanning method:** passive monitoring, manual audit, feed monitoring
-   **Scanning frequency:** weekly, monthly
-   **Scanning schedule:** nighttime/daytime, weekdays/weekends


## Final Thoughts

Smart vulnerability scanning is not about running tools blindly — it's about **understanding your environment**, **adapting scanning methods to real-world constraints**, and **prioritizing** what matters most.

To scan efficiently:
-   Focus on comprehensive asset visibility - you can't secure what you don't know exists.
-   Use a combination of methods (agent-based audit, agentless audit, passive, "pentest" mode) depending on asset type and location.
-   Align scan schedules with business operations and time zones to avoid missing critical assets.
-   Monitor vulnerability feeds and update scanning profiles to stay current.
-   Integrate scanning into your broader vulnerability management process, not as a one-off activity.

Smart scanning is not just about tools it’s about strategy. The more context-aware and tailored your approach is, the more secure and resilient infrastructure becomes.


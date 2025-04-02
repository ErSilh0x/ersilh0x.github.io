---
title: DHCP Data for Better Vulnerability Scanning Coverage
description: Enhancing vulnerability scanning coverage by leveraging DHCP lease data
date: 2024-12-10 21:36:00 +0000
#future: true
media_subpath: /assets/media/posts/2024-12-10-dhcp-data-for-scan/
image: dhcp-data-for-scan.jpg
categories: [Posts, Vulnerability Management]
tags: [scanning, dhcp]
---

The first step in vulnerability management is **asset discovery**, categorization, and prioritization.

Vulnerability scanners are commonly used to identify and track vulnerabilities. These scanners analyze hosts and detect software and configuration vulnerabilities. Often, scanning is performed actively over the network, where the scanner connects to hosts and collects necessary data.

This article focuses on workstations and similar endpoint devices that are connected to the corporate network and scanned for vulnerabilities over the network.

### Expanding Vulnerability Scanning with DHCP Data

End-user devices connect to access network segments and receive IP addresses **dynamically from a DHCP server**. Over time, the network topology may change, new user network segments may appear, and administrators managing the vulnerability scanner may not always be aware of these changes.

A practical approach is to extract lease data from DHCP servers and analyze the discovered network segments to ensure complete vulnerability scanning coverage.

For this example, I will use the MaxPatrol 8 vulnerability scanner and a Microsoft DHCP server. However, this approach can be applied to any vulnerability scanner and DHCP solution. I will extract the necessary data, analyze it using Python, and identify which network segments are missing from the scanning tasks.

We can view all leases on a specific DHCP server using the following PowerShell command:
```powershell
Get-DhcpServerv4Scope -ComputerName "DHCP-SERVER-NAME" | Format-Table -AutoSize -Wrap
```

It is better to export this data in .CSV format with tab separation:
```powershell
Get-DhcpServerv4Scope -ComputerName "DHCP-SERVER-NAME" | Select-Object ScopeId,SubnetMask,Name,State,StartRange,EndRange,LeaseDuration | Export-Csv DHCP-SERVER-NAME.csv -Delimiter "`t"
```

The next step is to export data from the MaxPatrol 8 vulnerability scanner. Let's export all scanning tasks in XML format, which will include scanning profiles and target addresses.
![MaxPatrol Tasks](mp_screen.jpg)

The exported file is structured as follows:
![MaxPatrol Task in XML](xml.jpg)

We are currently interested in the `tasks` element.

Then I created dedicated folders to store the exported data:
`/dhcpscopes` - for DHCP lease data 
`/tasks_mp` - for MaxPatrol scanning tasks

Next I used the Python script that I wrote for MaxPatrol tasks analyzes.

```python
#!/usr/bin/python3

import os
from csv import reader, writer
import ipaddress
import xml.etree.ElementTree as ET

# Save results
def save_results(inlist, resultname):
    fileResults = open(resultname + '.csv', mode='w', encoding='utf8', newline='')
    resultwriter = writer(fileResults, delimiter='\t')
    resultwriter.writerows(inlist)
    fileResults.close()

# Return list of IP ranges abscent in MP tasks
def showabscentranges(dhcpscopes, mpatroltasks):
    abscentscope = [['Network', 'ScopeId', 'SubnetMask', 'Name', 'State', 'StartRange', 'EndRange', 'LeaseDuration']]
    scopesintasks = []
    scopesnotintasks = []

    # Iterate dhcp dataset and get IP networks
    for sc in dhcpscopes[1:]:
        scope = sc[0] + '/' + sc[1]
        scope = ipaddress.IPv4Network(scope)

        # Iterate MaxPatrol tasks dataset and compare
        for tsk in mpatroltasks[1:]:
            if tsk[2] and tsk[3] and (tsk[2] != '127.0.0.1' and tsk[3] != '127.0.0.1'):
                startip = ipaddress.IPv4Address(tsk[2])
                endip = ipaddress.IPv4Address(tsk[3])
                tskranges = ipaddress.summarize_address_range(startip, endip)

                # Search IP from DHCP scope NOT in MP tasks
                for tskrange in tskranges: # Iterate IP ranges from MP tasks
                
                    # Make list with unique DHCP scopes present in MP tasks
                    for ipinscope in scope: # each IP in each scope
                        if ipinscope in tskrange:
                            if scope not in scopesintasks:
                                scopesintasks.append(scope)

        # Make list and dataset with unique DHCP scopes that are abscent in MP tasks
        if scope not in scopesintasks and scope not in scopesnotintasks:
            scopesnotintasks.append(scope)
            sc.insert(0, str(scope))
            abscentscope.append(sc)

    # scopesintasks: list with unique DHCP scopes present in MP tasks
    # scopesnotintasks: list with unique DHCP scopes that are abscent in MP tasks

    return abscentscope

pathtoscopes = os.getcwd() + '\\dhcpscopes\\' # For windows
pathtotasks = os.getcwd() + '\\tasks_mp\\' # For windows

# Parse dhcp scopes from powershell results
scopedataset = [['ScopeId', 'SubnetMask', 'Name', 'State', 'StartRange', 'EndRange', 'LeaseDuration']]
for f in os.listdir(pathtoscopes):
    filename = f
    print(filename)
    scopes = open(pathtoscopes + filename, mode='r', encoding='utf8', newline='\n')
    readscopes = reader(scopes, delimiter='\t')
    listscopes = list(readscopes)
    scopes.close()

    for row in listscopes[2:]:
        if row and row not in scopedataset:
            scopedataset.append(row)

# Parse maxpatrol tasks and add to dataset
mptasks = [['Task Name', 'Description', 'Start IP', 'End IP', 'Hostname']]
for f in os.listdir(pathtotasks):
    filename = f
    print(filename)
    tree = ET.parse(pathtotasks + filename)
    root = tree.getroot()

    for task in root.findall('{http://www.ptsecurity.ru/import}tasks'):
        print(task)
        for taskname in task:
            name = taskname.attrib.get('name')
            taskid = taskname.attrib.get('id')
            scanner = taskname.attrib.get('scanner')
            description = taskname.attrib.get('description')
            for taskprofile in taskname[0]:
                taskprofileid = taskprofile.attrib.get('id')
                for hostranges in taskprofile[0]:
                    #print(hostranges.attrib)
                    hostname = hostranges.attrib.get('primary')
                    mptasks.append([name, description, hostranges.attrib.get('from'), hostranges.attrib.get('to'), hostname])
                    #print(ipranges)

scopeabscentdataset = showabscentranges(scopedataset, mptasks)

# Print results
for row in scopeabscentdataset:
    print(row)

# Save results
save_results(scopeabscentdataset, 'scopes')

# Some stats
print('Total DHCP scopes: ', len(scopedataset) - 1)
print('DHCP scopes not in MP tasks: ', len(scopeabscentdataset) - 1)
```

The output should be a .CSV file with the following columns:
```csv
'Network', 'ScopeId', 'SubnetMask', 'Name', 'State', 'StartRange', 'EndRange', 'LeaseDuration'
```

As a result, out of 100 leases, I found around 20 that were missing from the scanning tasks. To put this into perspective, 20 IP leases with a /24 subnet mask could potentially represent 5,000 IP addresses. This means that over **1,000 hosts might be excluded** from the vulnerability scan coverage.














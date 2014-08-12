# [Company Name] Computers Names and Network Subnets Guidelines


## 1.0 Overview

Since names and addressing are used daily to identify nodes, it is important to be 
able to quickly and clearly determine which node a person or report is referring to. 


## 2.0 Purpose

This guideline defines naming conventions, numbering and addressing schemes for technical staff to support the IT operations of [Company Name].


## 3.0 Scope

This policy applies to the technical staff of [Company Name].


## 4.0 Guidelines

### 4.01 Domain naming

Type | Description
-----|------------
Public domain name  | A public website domain. The public will see this. Make it easy to type and easy to remember.
|                   | Example: company.com
Private domain name | A domain used internally to ease management of nodes and identity services. Staff may see this everyday and may be asked to type it. Make it short and do not use a TLD (no *.com, *.net, *.io, etc).
|                   | Example: acme.local

### 4.02 Networks (Data Center Subdomains)

Type | Description
-----|------------
Naming                 | City (Data Center Number) - Country - Region
|                      | Example: ny1-us-east-na.acme.local
|                      | sf1-us-west-na.acme.local
Private Network Subnet | Different for each DC site. Try to avoid common ones.
|                      | Example: ny1-us-east-na.acme.local is 10.0.10.0/32 (mask 255.255.255.0)
|                      | sf1-us-west-na.acme.local is 10.0.12.0/32 (mask 255.255.255.0)
Alternate Private Network Subnet | Different for each DC site. For use on servers as alias IPs for alternate access in case of subnet clash over VPN.
|                                | Example: ny1-us-east-na.acme.local is 10.0.11.0/32 (mask 255.255.255.0)
|                                | sf1-us-west-na.acme.local is 10.0.13.0/32 (mask 255.255.255.0)
Server Management Network Subnet | Different for each DC site. To keep management and client traffic separate.
|                                | Example: ny1-us-east-na.acme.local is 192.168.10.0/32 (mask 255.255.255.0)
|                                | sf1-us-west-na.acme.local is 192.168.12.0/32 (mask 255.255.255.0)
Jail Network Subnet    | Same for all jails on interface lo1.
|                      | Example: ny1-us-east-na.acme.local is 10.7.7.0
Virtual Network Subnet | Same for all virtual computers; especially for migration.

### 4.03 Rack / Cabinets

Type | Description
-----|------------
Naming                 | Use X, Y and Z co-ordinates as Numbers , Letters, and Numbers increment from North West. Like a spreadsheet.
|                      | Example: 2nd aisle (column), 3rd row, bottom shelf is 2C0.

### 4.04 Cluster of servers

Type | Description
-----|------------
Naming                 | Name the cluster after its function. Lowercase. 10 characters max.
|                      | Example: dhcpd.clstr.acme.local, pxe.clstr.acme.local, cdn.clstr.acme.local
Cluster Network Subnet | If segregating the cluster, should be different for each cluster from other traffic.
|                      | Example: alpha.acme.local is 172.16.1.0/32 (mask 255.255.255.0)
|                      | beta.acme.local is 172.16.1.0/32 (mask 255.255.255.0)

### 4.05 Servers

Type | Description
-----|------------
Naming        | Use  names of video game names, comic book characters, game characters. Alternatively, if in a rack, include cabinet and shelf unit. Lowercase. 10 characters max.
|             | Example: mario.acme.local, shyguy.acme.local, wolverine.acme.local
IP Addressing | Different for each server. Set as static. 100 to 199. Try to use even numbers.
IP Alias      | Useful for FreeBSD jails. Set as alias. 100 to 199. Try to use odd numbers.
IP Alternate  | Use Alternate Private Network Subnet in case of subnet clash over VPN. Set as static.
IP CARP       | A shared IP that gets passed on during failover. Set as alias using ucarp. 200 to 254. Try to use even numbers.

### 4.06 Shared Nodes (Printers, etc)

Type | Description
-----|------------
Naming        | Use printer brand + location / floor. Lowercase. 10 characters max.
|             | Example: hpstaff.acme.local, canonhr2.acme.local
IP Addressing | Different for each server. Set as static. 200 to 254. Try to use odd numbers.

### 4.07 RDP Nodes

Type | Description
-----|------------
Naming        | Use color + thing names. Lowercase. 10 characters max.
|             | Example: redbrick.acme.local, bluesky.acme.local, greengrass.acme.local
IP Addressing | Different for each server. Set as static. 100 to 199. Try to use even numbers.
IP Alternate  | Use Alternate Private Network Subnet in case of subnet clash over VPN. Set as static.

### 4.08 Staff Nodes

Type | Description
-----|------------
Naming        | Service Tag, or serial number.
IP Addressing | DHCP reservation.

### 4.09 Guest Nodes

Type | Description
-----|------------
IP Addressing | Automatic via DHCP.

### 4.10 External drives / USB flash drives

Type | Description
-----|------------
Naming | Phonetic alphabet, then animals. (alpha, bravo, charlie)

### 4.11 Connectivity devices (Switches, Hubs, Cellular Data sticks)

Type | Description
-----|------------
Naming        | Plants.
IP Addressing | Typically, a router is x.x.x.1

### 4.12 Other peripherals

Type | Description
-----|------------
Naming | Brand + Number.


## 5.0 Enforcement

N/A


## 6.0 Definitions

Terms             | Definitions 
------------------|------------
a node            | A device (PC, virtual computer, mobile) connected to a network.
PC / computer     | A device utilized by a user to run software.
desktop           | A computer with a non-portable form factor, typically does not move once setup at a Workstation.
laptop / notebook | A computer with a semi-portable form factor, typically can be shutdown and carried to work at another location.
tablet            | A computer with a very portable form factor, typically can be used while carried.
workstation       | A collection of assets Staff use to perform work; may include desk, chair, computer, external monitor.
server            | A computer running in a capacity where they serve other nodes.
a host            | A server which may run virtual servers.
client            | A node connecting to a server.
thin-client       | A computer without an OS, typically used to connect to the window manager session of a remote computer.
window manager    | The GUI of an OS.
GUI               | Graphical User Interface.
Operating system  | A collection of software installed on a node that allows a user to interact with the node.
OS                | See Operating system.
DC                | Data center.

## 7.0 Revision

Name           | Date       | Description
---------------|------------|------------
John Ko        | 2014-08-11 | Original

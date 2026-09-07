# Cybersecurity Home Lab Setup


## Objective

The purpose of this project was to build a safe virtual environment where I could develop practical cybersecurity, networking, Windows, and Linux administration skills.

Rather than performing security exercises on my physical computer or home network, I created virtual machines using Oracle VirtualBox.

The initial lab consists of:

- A Windows 11 host computer
- An Ubuntu Linux virtual machine
- A Windows 11 Pro virtual machine
- An isolated VirtualBox network for communication between the lab systems

This environment provides the foundation for future projects involving network reconnaissance, packet analysis, logging, security monitoring, Active Directory, and controlled attack-and-defense exercises.

---

## 1. Host Computer

The physical computer hosting the lab runs:

```text
Operating System: Windows 11
Architecture:     x64
Memory:           16 GB RAM
Virtualization:   Enabled
```

Oracle VirtualBox is used as the virtualization platform.

VirtualBox version used during the initial build:

```text
VirtualBox 7.2.8
```

### Why Virtualization?

Virtualization allows multiple operating systems to run on one physical computer.

Each virtual machine behaves like a separate computer with its own:

- Operating system
- CPU allocation
- Memory allocation
- Virtual storage
- Network interfaces

This makes it possible to create a controlled cybersecurity environment without requiring several physical computers.

---

## 2. Ubuntu-Lab Virtual Machine

The first virtual machine created was Ubuntu-Lab.

Configuration:

| Setting | Configuration |
|---|---|
| VM Name | `Ubuntu-Lab` |
| Operating System | Ubuntu 25.04 |
| Memory | 4 GB |
| CPUs | 2 |
| Virtual Disk | 40 GB |
| Hostname | `Ubuntu-Lab` |

Ubuntu-Lab is used as the primary Linux administration and security workstation in the lab.

Tools and activities performed from Ubuntu-Lab include:

- Linux command-line administration
- Network troubleshooting
- Nmap reconnaissance
- Wireshark packet analysis
- Connectivity testing

---

## 3. Ubuntu-Lab Networking

Ubuntu-Lab uses two virtual network adapters.

### Adapter 1 - NAT

The first adapter uses VirtualBox NAT.

Ubuntu interface:

```text
enp0s3
```

Example IPv4 configuration:

```text
IP Address:      10.0.2.15/24
Default Gateway: 10.0.2.2
```

The NAT interface provides Ubuntu-Lab with Internet access through VirtualBox.

This allows Ubuntu to perform tasks such as:

```bash
sudo apt update
sudo apt install <package>
```

without directly placing the virtual machine on the physical home network.

### Adapter 2 - Internal Network

The second adapter connects Ubuntu-Lab to the isolated cybersecurity network.

VirtualBox network name:

```text
CYBER-LAB
```

Ubuntu interface:

```text
enp0s8
```

Static IPv4 configuration:

```text
IP Address: 192.168.50.10
Prefix:     /24
```

No default gateway or DNS server is configured on this interface.

Internet traffic continues to use the NAT adapter, while cybersecurity lab traffic uses `enp0s8`.

---

## 4. Windows-Lab Virtual Machine

The second virtual machine created was Windows-Lab.

Configuration:

| Setting | Configuration |
|---|---|
| VM Name | `Windows-Lab` |
| Operating System | Windows 11 Pro |
| Memory | 4 GB |
| CPUs | 2 |
| Virtual Disk | 60 GB |

Windows-Lab is used as a Windows endpoint for:

- Windows administration
- PowerShell practice
- Windows Defender Firewall configuration
- Network testing
- Service analysis
- Security monitoring
- Future Active Directory and security exercises

---

## 5. Windows 11 Installation Challenges

Several issues were encountered while installing Windows 11 in VirtualBox.

### Windows 11 Hardware Requirements

Windows Setup initially reported that the virtual machine did not meet the Windows 11 system requirements.

The VM configuration was adjusted to support the required Windows features, including:

```text
UEFI/EFI
TPM 2.0
Secure Boot
```

This allowed the Windows installation process to continue.

---

## 6. VirtualBox Black Screen Troubleshooting

During the Windows installation, the VM displayed a black screen.

After troubleshooting the VirtualBox graphics configuration, the graphics controller was changed to:

```text
VMSVGA
```

After this change, Windows Setup displayed correctly and installation was able to continue.

### What I Learned

Virtual machine problems are not always caused by the guest operating system.

The virtualization platform itself can affect:

- Display behavior
- Boot behavior
- Hardware compatibility
- Networking
- Device availability

Troubleshooting therefore requires checking both the guest operating system and the VM configuration.

---

## 7. Windows-Lab Networking

Windows-Lab was intentionally configured differently from Ubuntu-Lab.

It uses only the isolated VirtualBox Internal Network:

```text
CYBER-LAB
```

Static IPv4 configuration:

```text
IP Address:  192.168.50.20
Subnet Mask: 255.255.255.0
```

No default gateway or DNS server is configured.

This means Windows-Lab can communicate with systems on the `192.168.50.0/24` lab network but does not currently have a normal route to the Internet.

---

## 8. Lab Network Architecture

The isolated lab network uses:

```text
Network: 192.168.50.0/24
Name:    CYBER-LAB
```

Current systems:

```text
Ubuntu-Lab
192.168.50.10
      |
      |
      +---------- CYBER-LAB ----------+
                                      |
                                      |
                               Windows-Lab
                               192.168.50.20
```

Ubuntu-Lab also has a separate NAT connection:

```text
                         Internet
                            |
                            |
                     VirtualBox NAT
                            |
                            |
                         enp0s3
                            |
                      Ubuntu-Lab
                     192.168.50.10
                            |
                          enp0s8
                            |
                       CYBER-LAB
                            |
                            |
                       Windows-Lab
                      192.168.50.20
```

A graphical version of the lab topology is available in:

```text
diagrams/network-topology.png
```

---

## 9. Isolation Strategy

One of the design goals of the lab was to separate cybersecurity testing traffic from the physical home network.

Windows-Lab therefore does not currently have a NAT adapter.

Its primary communication path is:

```text
Windows-Lab
     |
     v
CYBER-LAB
     |
     v
Ubuntu-Lab
```

Ubuntu-Lab has both:

```text
NAT         -> Internet access
CYBER-LAB   -> Isolated lab access
```

This allows Ubuntu-Lab to download software and updates while maintaining a separate interface for lab traffic.

---

## 10. Initial Linux Administration

During the setup process, I practiced several Linux commands used for system and network administration.

Examples include:

```bash
whoami
pwd
ls
hostname
ip addr
ip route
ip neigh
ping
resolvectl dns
```

These commands were used to identify the system, inspect network interfaces, examine routes, view neighboring systems, test connectivity, and troubleshoot DNS.

---

## 11. Network Troubleshooting

During the lab build, Ubuntu-Lab temporarily lost Internet connectivity.

The symptoms included:

```text
network unreachable
```

and package-management errors involving DNS resolution.

Investigation showed that the NAT interface `enp0s3` was not functioning with its expected IPv4 configuration.

After correcting the network configuration, Ubuntu-Lab regained:

```text
10.0.2.15/24
```

on `enp0s3`.

Connectivity was verified using:

```bash
ping 8.8.8.8
```

and a hostname-based ping.

This helped distinguish between:

- Interface configuration
- Routing
- Internet connectivity
- DNS resolution

### What I Learned

Testing connectivity in stages can help isolate the source of a network problem.

For example:

```text
Interface
    |
    v
IP Address
    |
    v
Route
    |
    v
Gateway / Network Reachability
    |
    v
Internet Reachability
    |
    v
DNS Resolution
```

---

## 12. VirtualBox Guest Integration

VirtualBox guest utilities were installed in Ubuntu-Lab.

Installed packages included:

```text
virtualbox-guest-utils
virtualbox-guest-x11
```

These provide integration features between the host and virtual machine.

One feature used during the lab was shared clipboard support.

The VirtualBox client utility was identified as:

```text
/usr/bin/VBoxClient
```

Clipboard functionality could be started with:

```bash
VBoxClient --clipboard
```

This troubleshooting exercise also reinforced that Linux commands and filenames are case-sensitive.

For example:

```text
VBoxClient
```

is different from:

```text
VboxClient
```

---

## 13. Initial Lab Validation

After configuring both virtual machines, connectivity between them was tested.

Ubuntu-Lab:

```text
192.168.50.10
```

Windows-Lab:

```text
192.168.50.20
```

The systems were eventually able to communicate in both directions across `CYBER-LAB`.

This confirmed that:

- Both VMs were connected to the same VirtualBox Internal Network
- Both systems were configured within the same IPv4 subnet
- The virtual network was functioning
- Host firewall behavior could be investigated separately from basic network connectivity

---

## 14. Tools Added to the Lab

After establishing the initial environment, I installed security and networking tools on Ubuntu-Lab.

### Nmap

Nmap was installed for network reconnaissance and port scanning.

The lab was used to practice:

- Host discovery
- Port scanning
- Service detection
- OS fingerprinting
- Targeted port scanning

### Wireshark

Wireshark was installed for packet capture and analysis.

The lab was used to examine:

- ICMP Echo Requests and Replies
- TCP SYN packets
- TCP SYN/ACK responses
- TCP three-way handshakes
- Open vs. filtered port behavior

Detailed experiments are documented separately in the later lab reports.

---

## 15. Current Lab Topology

At the completion of the initial setup stage:

| Device | Interface | Address | Network | Purpose |
|---|---|---|---|---|
| Ubuntu-Lab | `enp0s3` | `10.0.2.15/24` | VirtualBox NAT | Internet |
| Ubuntu-Lab | `enp0s8` | `192.168.50.10/24` | CYBER-LAB | Lab |
| Windows-Lab | Ethernet | `192.168.50.20/24` | CYBER-LAB | Lab |

The primary cybersecurity testing network is:

```text
192.168.50.0/24
```

---

## 16. Skills Practiced

This initial build provided hands-on experience with:

- Virtualization
- Oracle VirtualBox
- Windows 11 installation
- Ubuntu Linux installation
- Virtual network adapters
- NAT networking
- Internal networking
- Static IPv4 configuration
- IPv4 subnetting concepts
- Linux network administration
- Windows network administration
- Routing concepts
- DNS troubleshooting
- Connectivity testing
- Windows Defender Firewall
- Linux command-line usage
- Virtual machine troubleshooting
- Guest integration
- Technical documentation

---

## 17. Key Takeaways

Building the environment itself became an important part of the learning process.

Several problems occurred during installation and configuration, including Windows virtual hardware requirements, display problems, Internet connectivity problems, firewall behavior, and guest integration issues.

Instead of rebuilding the environment whenever a problem occurred, I practiced identifying the symptoms, collecting system information, testing possible causes, making controlled changes, and verifying the results.

This established a repeatable troubleshooting process that can be applied to future networking and cybersecurity problems.

---

## Next Steps

The next stage documents the configuration and testing of the isolated `CYBER-LAB` network in greater detail.

See:

```text
docs/02-network-configuration.md
```

Additional labs cover:

```text
docs/03-nmap-firewall-lab.md
docs/04-wireshark-packet-analysis.md
```

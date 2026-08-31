# Nmap Reconnaissance and Windows Firewall Lab

## Objective

The objective of this lab was to learn how Nmap observes a Windows endpoint across a network and how Windows Defender Firewall affects port-scanning results.

The experiment was performed entirely inside my isolated VirtualBox `CYBER-LAB` environment.

## Lab Systems

| System | Operating System | IP Address | Purpose |
|---|---|---|---|
| Ubuntu-Lab | Ubuntu 25.04 | `192.168.50.10` | Scanning / administration |
| Windows-Lab | Windows 11 Pro | `192.168.50.20` | Windows endpoint |

Lab network:

`192.168.50.0/24`

---

## 1. Initial Nmap Scan

From Ubuntu-Lab, I performed an Nmap scan against Windows-Lab:

```bash
nmap 192.168.50.20
```

Nmap initially reported:

```text
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
```

This demonstrated that host discovery can be affected by firewall configuration.

I then instructed Nmap to skip its normal host-discovery process:

```bash
nmap -Pn 192.168.50.20
```

The scan reported that all 1000 scanned TCP ports were filtered:

```text
All 1000 scanned ports on 192.168.50.20 are in ignored states.
Not shown: 1000 filtered tcp ports (no-response)
```

### What I Learned

The result did not necessarily mean that Windows had no services running.

`filtered` means that Nmap could not determine whether the ports were open or closed because network filtering prevented it from receiving the responses needed to make that determination.

---

## 2. Checking Listening Ports Locally

To compare Nmap's external view with what was actually happening inside Windows-Lab, I used PowerShell:

```powershell
Get-NetTCPConnection -State Listen
```

Windows showed several listening TCP ports, including:

```text
135
139
445
5040
49664-49669
```

Some important examples were:

- TCP 135 - Microsoft RPC
- TCP 139 - NetBIOS Session Service
- TCP 445 - SMB
- TCP 49664-49669 - Dynamic RPC ports

This produced an important observation:

> A service can be listening locally while still being unreachable from another system because of a host firewall.

---

## 3. Targeted Port Scan

I performed a targeted scan against several ports that Windows reported as listening:

```bash
nmap -Pn -p 135,139,445,5040 192.168.50.20
```

Nmap reported:

```text
135/tcp  filtered  msrpc
139/tcp  filtered  netbios-ssn
445/tcp  filtered  microsoft-ds
5040/tcp filtered  unknown
```

Windows showed these services as listening locally, but Ubuntu could not reach them through the firewall.

---

## 4. Creating a Restricted SMB Firewall Rule

To investigate the firewall behavior, I created a Windows Defender Firewall inbound rule for TCP port 445.

Instead of allowing every system to access SMB, I restricted the rule to the IP address of Ubuntu-Lab:

```text
Remote IP: 192.168.50.10
Protocol: TCP
Local Port: 445
Action: Allow
Direction: Inbound
```

This allowed only the Ubuntu lab system to reach the SMB service through this rule.

### Before the Firewall Rule

```text
445/tcp filtered microsoft-ds
```

### After the Firewall Rule

I repeated the scan:

```bash
nmap -Pn -p 445 192.168.50.20
```

The result changed to:

```text
445/tcp open microsoft-ds
```

### What I Learned

The SMB service had already been listening on Windows.

Creating the firewall rule did not start SMB. Instead, it changed whether Ubuntu-Lab could reach the existing service.

This demonstrated the difference between:

- a service listening on a host
- a firewall permitting network traffic to that service
- a service being externally reachable

---

## 5. Service Detection

I then used Nmap service/version detection:

```bash
nmap -Pn -sV -p 445 192.168.50.20
```

The result was:

```text
PORT    STATE SERVICE       VERSION
445/tcp open  microsoft-ds?
```

Nmap recognized the service as being associated with Microsoft networking but did not confidently identify a specific version.

### What I Learned

The `-sV` option asks Nmap to perform additional service-detection probes.

However, service detection does not guarantee that an exact application or version will be identified.

---

## 6. Operating System Detection

I attempted OS fingerprinting:

```bash
sudo nmap -Pn -O 192.168.50.20
```

Important results included:

```text
Not shown: 999 filtered tcp ports (no-response)

PORT    STATE SERVICE
445/tcp open  microsoft-ds
```

Nmap also identified the network adapter as:

```text
Oracle VirtualBox virtual NIC
```

Nmap's highest-confidence operating system guess was Microsoft Windows 11.

However, Nmap displayed the warning:

```text
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
```

It also reported:

```text
No exact OS matches for host (test conditions non-ideal).
```

### What I Learned

OS fingerprinting is based on characteristics of network responses.

Nmap was able to infer that the target was probably a Microsoft Windows system, but it could not confidently determine the exact operating system because the scan conditions were not ideal.

This reinforced an important security-analysis principle:

> Tool output should be treated as evidence, not automatically as fact.

---

## 7. Filtered-Port Experiment

I selected TCP port `12345` to further investigate the difference between firewall filtering and listening services.

Initial scan:

```bash
nmap -Pn -p 12345 192.168.50.20
```

Result:

```text
PORT      STATE     SERVICE
12345/tcp filtered  netbus
```

I then created another restricted inbound firewall rule:

```text
Remote IP: 192.168.50.10
Protocol: TCP
Local Port: 12345
Action: Allow
Direction: Inbound
```

I verified the Windows Firewall rule using PowerShell.

The rule showed:

```text
Enabled: True
Profile: Any
Direction: Inbound
Action: Allow
Protocol: TCP
LocalPort: 12345
RemoteAddress: 192.168.50.10
```

Despite the allow rule, Nmap continued to report TCP 12345 as filtered.

This became a useful troubleshooting exercise because it demonstrated that firewall behavior and Nmap port states can involve more than simply creating an inbound allow rule.

Further investigation of this behavior is part of the next stage of the lab.

---

## Key Takeaways

This lab helped me understand several concepts:

1. A locally listening service is not necessarily reachable across a network.
2. Host firewalls can significantly affect reconnaissance results.
3. Nmap's `filtered` state is different from `open` and `closed`.
4. `-Pn` allows scanning without relying on normal host discovery.
5. `-sV` performs service/version detection.
6. `-O` performs operating system fingerprinting.
7. OS and service detection results can contain uncertainty.
8. Firewall rules should be scoped as narrowly as practical rather than unnecessarily exposing services.
9. Internal system information can be compared with external reconnaissance results during troubleshooting.

---

## Next Steps

The next experiment will continue investigating TCP port states by creating a temporary controlled listener and observing how Nmap's results change.

Future stages of the home lab will also include Windows/Linux administration, logging, Active Directory, SIEM monitoring, detection, and controlled attack-and-defense exercises.

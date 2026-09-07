# Wireshark Packet Analysis Lab

## Objective

The objective of this lab was to use Wireshark to capture and analyze network traffic between two virtual machines in my isolated cybersecurity home lab.

This lab built on my previous Nmap and Windows Defender Firewall experiments by allowing me to observe the actual packets responsible for ICMP communication and TCP port-scanning results.

The main goals were to:

- Capture traffic on the correct network interface
- Analyze ICMP Echo Request and Echo Reply packets
- Observe a TCP three-way handshake
- Compare Nmap traffic against an open TCP port with traffic against a filtered TCP port
- Understand how packet-level behavior relates to Nmap port states

---

## Lab Environment

| System | Operating System | IP Address | Purpose |
|---|---|---|---|
| Ubuntu-Lab | Ubuntu 25.04 | `192.168.50.10` | Wireshark capture and Nmap scanning |
| Windows-Lab | Windows 11 Pro | `192.168.50.20` | Target Windows endpoint |

Internal lab network:

```text
CYBER-LAB
192.168.50.0/24
```

Ubuntu-Lab is connected to the internal network using:

```text
enp0s8
192.168.50.10/24
```

Windows-Lab is connected only to the isolated `CYBER-LAB` network.

---

## 1. Installing Wireshark

Wireshark was installed on Ubuntu-Lab using:

```bash
sudo apt update
sudo apt install wireshark
```

During installation, I allowed non-superusers to capture packets.

This allows authorized members of the `wireshark` group to perform packet captures without running the entire Wireshark application as root.

I added my Ubuntu user to the `wireshark` group:

```bash
sudo usermod -aG wireshark $USER
```

After rebooting, I verified the group membership:

```bash
groups
```

The output included:

```text
wireshark
```

I then verified the Wireshark installation:

```bash
wireshark --version
```

Installed version:

```text
Wireshark 4.6.4
```

### Security Concept

Allowing an authorized group to perform packet capture instead of running the entire Wireshark application as root demonstrates the principle of **least privilege**.

---

## 2. Selecting the Capture Interface

Ubuntu-Lab has more than one network interface.

The interface connected to the isolated cybersecurity lab is:

```text
enp0s8
```

I verified its configuration using:

```bash
ip addr show enp0s8
```

The interface showed:

```text
inet 192.168.50.10/24
```

I also verified which interface Ubuntu would use to communicate with Windows-Lab:

```bash
ip route get 192.168.50.20
```

The result showed:

```text
192.168.50.20 dev enp0s8 src 192.168.50.10
```

This confirmed that traffic between Ubuntu-Lab and Windows-Lab travels through `enp0s8`.

I therefore selected `enp0s8` as the Wireshark capture interface.

### What I Learned

Selecting the correct interface is important during packet analysis.

Capturing on Ubuntu-Lab's NAT interface would primarily show traffic associated with Internet access, while capturing on `enp0s8` shows traffic on the isolated `CYBER-LAB` network.

---

## 3. ICMP Packet Capture

To generate simple network traffic, I used the Linux `ping` command from Ubuntu-Lab:

```bash
ping -c 4 192.168.50.20
```

The `-c 4` option instructed Ubuntu to send four ICMP Echo Requests.

The test produced:

```text
4 packets transmitted, 4 received, 0% packet loss
```

I then applied the following Wireshark display filter:

```text
icmp
```

Wireshark displayed four ICMP Echo Requests and four corresponding Echo Replies.

The traffic followed this pattern:

```text
Ubuntu-Lab                         Windows-Lab
192.168.50.10                     192.168.50.20
     |                                  |
     | ---- ICMP Echo Request --------> |
     |                                  |
     | <---- ICMP Echo Reply ---------- |
     |                                  |
```

---

## 4. Analyzing an ICMP Echo Request

I selected an ICMP Echo Request in Wireshark and examined the packet details.

The IPv4 information showed:

```text
Source Address:      192.168.50.10
Destination Address: 192.168.50.20
```

Under Internet Control Message Protocol, Wireshark showed:

```text
Type: 8 (Echo (ping) request)
Code: 0
```

This packet represented Ubuntu-Lab asking Windows-Lab to respond.

Wireshark also identified the corresponding response frame.

---

## 5. Analyzing an ICMP Echo Reply

The corresponding packet traveled in the opposite direction:

```text
Source Address:      192.168.50.20
Destination Address: 192.168.50.10
```

The ICMP information showed:

```text
Type: 0 (Echo (ping) reply)
Code: 0
```

The request and reply therefore demonstrated:

```text
Type 8 = Echo Request
Type 0 = Echo Reply
```

### TTL Observation

The packet capture also showed different TTL values.

Ubuntu-Lab's Echo Requests showed:

```text
TTL = 64
```

Windows-Lab's Echo Replies showed:

```text
TTL = 128
```

This demonstrated that different operating systems can use different initial TTL values.

Characteristics such as these can contribute to network-based operating system fingerprinting, although an individual characteristic should not be treated as definitive proof of a particular operating system.

---

## 6. TCP Port 12345 Experiment

I next used Wireshark to observe the traffic generated during an Nmap scan of TCP port `12345`.

This built on my previous Windows Firewall and Nmap experiment.

Windows-Lab already had a restricted inbound firewall rule permitting Ubuntu-Lab (`192.168.50.10`) to communicate with TCP port `12345`.

I created a temporary TCP listener on Windows-Lab using PowerShell:

```powershell
$listener = [System.Net.Sockets.TcpListener]::new([System.Net.IPAddress]::Any, 12345)
$listener.Start()
```

I verified the listener with:

```powershell
Get-NetTCPConnection -LocalPort 12345 -State Listen
```

---

## 7. Capturing an Open TCP Port

With the temporary listener active, I started a Wireshark capture on `enp0s8`.

From Ubuntu-Lab, I performed the following scan:

```bash
nmap -Pn -p 12345 192.168.50.20
```

Nmap reported TCP port 12345 as open.

I then applied the following Wireshark display filter:

```text
tcp.port == 12345
```

The capture showed the following TCP packets:

```text
Ubuntu-Lab                         Windows-Lab
192.168.50.10                     192.168.50.20
     |                                  |
     | -------- SYN ------------------> |
     |                                  |
     | <------ SYN, ACK --------------- |
     |                                  |
     | -------- ACK ------------------> |
     |                                  |
     | ------ RST, ACK ---------------->|
```

---

## 8. TCP Three-Way Handshake

The first three packets demonstrated the TCP three-way handshake.

### Step 1 - SYN

Ubuntu-Lab sent a TCP SYN packet to Windows-Lab:

```text
192.168.50.10:<ephemeral-port> -> 192.168.50.20:12345 [SYN]
```

This initiated the attempt to establish a TCP connection.

### Step 2 - SYN/ACK

Windows-Lab responded:

```text
192.168.50.20:12345 -> 192.168.50.10:<ephemeral-port> [SYN, ACK]
```

The SYN/ACK response demonstrated that TCP port 12345 was reachable and actively listening.

### Step 3 - ACK

Ubuntu-Lab responded with:

```text
[ACK]
```

The three packets formed:

```text
SYN
 |
 v
SYN/ACK
 |
 v
ACK
```

This is the TCP three-way handshake.

### What I Learned

Instead of only reading that TCP uses a three-way handshake, I was able to generate the traffic myself and observe each packet in Wireshark.

---

## 9. Connection Reset

After the handshake, the capture also showed:

```text
[RST, ACK]
```

The reset terminated the connection after the scanning interaction.

This demonstrated that a reconnaissance tool can establish enough communication to determine the state of a TCP port without maintaining a normal application session.

---

## 10. Ephemeral Source Ports

The capture also demonstrated that Ubuntu-Lab used a temporary source port while connecting to Windows-Lab.

For example:

```text
Source:
192.168.50.10:43528

Destination:
192.168.50.20:12345
```

Windows-Lab was listening on TCP port `12345`, while Ubuntu selected a temporary source port for the connection.

This helped demonstrate how a system can maintain multiple simultaneous network connections.

---

## 11. Capturing a Filtered TCP Port

I then stopped the temporary TCP listener on Windows-Lab.

I verified that nothing was listening on TCP port 12345 using:

```powershell
Get-NetTCPConnection -LocalPort 12345 -State Listen
```

PowerShell reported that no matching listening connection was found.

I started another Wireshark capture on Ubuntu-Lab and repeated the Nmap scan:

```bash
nmap -Pn -p 12345 192.168.50.20
```

Nmap returned the port to:

```text
12345/tcp filtered
```

I applied the same Wireshark display filter:

```text
tcp.port == 12345
```

This time, the capture showed SYN packets traveling from Ubuntu-Lab to Windows-Lab but no TCP response from Windows-Lab.

Example:

```text
192.168.50.10:52398 -> 192.168.50.20:12345 [SYN]

192.168.50.10:52404 -> 192.168.50.20:12345 [SYN]
```

No SYN/ACK response was observed.

---

## 12. Open vs. Filtered Comparison

The two captures demonstrated a clear difference.

| Port State | Ubuntu Sends | Windows Response | Nmap Result |
|---|---|---|---|
| Listener running | SYN | SYN/ACK | `open` |
| Listener stopped | SYN | No TCP response observed | `filtered` |

### Open Port

```text
Ubuntu                         Windows
   |                              |
   | -------- SYN ------------->  |
   | <------ SYN/ACK -----------  |
   | -------- ACK ------------->  |
   |                              |

            OPEN
```

### Filtered Port

```text
Ubuntu                         Windows
   |                              |
   | -------- SYN ------------->  |
   |                              |
   |        No response           |
   |                              |
   | -------- SYN ------------->  |
   |                              |
   |        No response           |

          FILTERED
```

### What I Learned

This experiment provided a packet-level explanation for the Nmap results.

When the TCP listener was active, Windows responded to the connection attempt with a SYN/ACK. Nmap therefore had evidence that TCP port 12345 was open.

When the listener was removed, my capture showed Nmap's SYN probes leaving Ubuntu-Lab but no corresponding TCP response from Windows-Lab. From the scanner's perspective, it could not determine that the port was open or closed, resulting in a `filtered` classification.

---

## 13. Wireshark Display Filters Used

Two Wireshark display filters were used during this lab.

### ICMP Traffic

```text
icmp
```

This displayed ICMP packets such as ping requests and replies.

### TCP Port 12345

```text
tcp.port == 12345
```

This displayed TCP packets where either the source or destination port was TCP 12345.

### Display Filter vs. Capture Filter

A Wireshark display filter changes which packets are displayed from an existing capture.

It does not change which packets were originally captured.

This is different from a capture filter, which controls which traffic Wireshark records during the capture itself.

---

## 14. Troubleshooting During the Lab

During the first ICMP capture attempt, Ubuntu-Lab reported:

```text
Destination Host Unreachable
```

I verified the route using:

```bash
ip route get 192.168.50.20
```

The route correctly showed:

```text
192.168.50.20 dev enp0s8 src 192.168.50.10
```

The issue was ultimately that Windows-Lab was not running.

After starting Windows-Lab, the ping succeeded:

```text
4 packets transmitted, 4 received, 0% packet loss
```

### Troubleshooting Lesson

This reinforced a basic but important troubleshooting principle:

> Verify that the destination system is powered on and available before assuming that a network configuration problem exists.

It also demonstrated the importance of checking one layer at a time rather than immediately changing network settings.

---

## 15. Key Takeaways

- Wireshark can capture and inspect individual network packets.
- Selecting the correct network interface is essential for useful packet analysis.
- ICMP Echo Requests use Type 8.
- ICMP Echo Replies use Type 0.
- A successful ping consists of request and reply traffic.
- TCP connections begin with a three-way handshake: SYN, SYN/ACK, ACK.
- A SYN/ACK response provides evidence that a TCP port is reachable and listening.
- A filtered Nmap result can correspond to probes receiving no useful response.
- Firewalls and application listeners affect what a remote scanner observes.
- Temporary source ports are used when systems initiate TCP connections.
- Wireshark display filters make it easier to isolate relevant traffic.
- Packet captures can validate and explain results produced by tools such as Nmap.
- Tool output should be interpreted using supporting network evidence.
- Troubleshooting should involve verification and controlled changes rather than assumptions.

---

## Skills Practiced

This lab provided hands-on experience with:

- Wireshark
- Packet capture
- Packet filtering
- ICMP analysis
- TCP analysis
- TCP three-way handshake
- Nmap
- Windows Defender Firewall
- PowerShell networking commands
- Linux networking commands
- IPv4 addressing
- Network troubleshooting
- Port-state analysis
- Basic network reconnaissance

---

## Conclusion

This lab connected several networking and cybersecurity concepts that I had previously studied separately.

Nmap showed how a remote scanner classified TCP ports, while Wireshark allowed me to observe the actual packets responsible for those classifications.

By comparing ICMP traffic, an open TCP port, and a filtered TCP port, I gained practical experience analyzing network communication at the packet level.

The most important lesson from this lab was that security-tool output becomes much more meaningful when I can investigate and verify the underlying network behavior rather than simply accepting the result displayed by a tool.

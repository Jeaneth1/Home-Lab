# Restricted Zone Segmentation with OPNsense

**Home Lab** | Jeaneth Lliguicota

<br>

## Goal

Created a restricted zone where only the admin VM can reach the restricted server over SSH (port 22). All other devices are blocked on every port, including port 22.

<br>

## Build

| Machine | OS | Network | Role |
|---|---|---|---|
| Proxmox | Debian Linux | | Hypervisor |
| Kali | Kali Linux | vmbr1 | Attacker |
| OPNsense | FreeBSD | vmbr1 | Firewall |
| admin1 | Ubuntu Linux | vmbr1 | Admin |
| restricted-srv | Ubuntu Linux | vmbr3 | Restricted server |
| Greenbone | Ubuntu Linux | vmbr2 | Scanner |

<br>

### LAN

| Action | Protocol | Source | Source Port | Destination | Destination Port | Description |
|---|---|---|---|---|---|---|
| Pass | TCP | Admin IP | any | Restricted network | SSH (22) | Allow admin VM SSH to restricted server |
| Block | any | LAN Network | any | Restricted network | any | Block everyone in vmbr1 from entering the restricted server. |

![LAN rules](<images/LAN updated 2 rules.png>)

<br>

### Restricted

| Action | Protocol | Source | Source Port | Destination | Destination Port | Description |
|---|---|---|---|---|---|---|
| Block | any | Restricted network | any | any | any | Restricted block from communicating with anyone |

![Restricted rules](<images/restricted rules.png>)

<br>

### Scanner

| Action | Protocol | Source | Source Port | Destination | Destination Port | Description |
|---|---|---|---|---|---|---|
| Block | any | SCANNER network | any | RESTRICTED network | any | Scanner vmbr2 block from communicating with the Restricted zone. |
| Pass | any | SCANNER network | any | any | any | Scanners can access the internet and other targets. (existing rule) |

![Scanner rules](<images/final scanner block.png>)
*Firewall rules are read top to bottom, so the block catches restricted traffic before the pass rule can allow it.*

<br>

## Proof

| Test | From | To | Expected | Actual |
|---|---|---|---|---|
| `ssh user@10.xx.xx.x` | Admin VM | Restricted server | Logs in | Logged in. Return hostname restricted-srv |
| `nmap -Pn -p 22` | Kali | Restricted server | filtered | Nmap port 22: 22/tcp filtered |
| `nmap -Pn -p 1-1000` | Kali | Restricted server | all filtered | Nmap 1 to 1000: all 1000 filtered, 205.75 seconds |
| `ping` | Kali | Restricted server | 100% loss | 100% loss |
| `ping` | Scanner | Restricted server | 100% loss | 100% loss |
| `ping` | Restricted server | Kali | 100% loss | 100% loss |

![Admin SSH](<images/admin 300 sucessfully ssh into restricted server.png>)
*Admin successfully ssh into the restricted zone*

---

![Pass log](<images/ssh success log in updated finally.png>)
*OPNsense firewall log with pass rule for admin to restricted zone via port 22*

---

![Kali nmap port 22 log](<images/Kali Lan block from nmap port 22 to restricted server.png>)
*Kali nmap -Pn -p 22 logs*

---

![Kali nmap 1-1000](<images/1-1000 nmap.jpg>)
*Kali nmap scan 1-1000 to the restricted zone being dropped.*

---

![Kali ping](<images/ping 10.jpg>)
*Kali ping to restricted zone*

---

![Scanner ping](<images/greenbone terminal no ping transmitted into restricted server.png>)
*Scanner ping to the restricted server*

---

![Scanner ping log](<images/scanner block rule applied log .png>)
*OPNsense log from Scanner ping to the restricted server logs*

---

![Restricted ping](<images/restricted server can't contact kali and other lan block .png>)
*Restricted server ping to Kali*

---

![Restricted ping log](<images/restricted server ping kali logs .png>)
*OPNsense logs where restricted is block from pinging Kali*

<br>

### Wireshark

I captured traffic with OPNsense Packet Capture on the LAN interface, then opened the files in Wireshark on Kali to confirm what happened on the wire: admin1 completed the three way handshake and Kali attempts got no reply.

Kali attempted to communicate with the restricted zone, 2 requests sent and filtered out by OPNsense LAN rules.

![Wireshark blocked scan](<images/WireShark kali TCP SYN packets denied.png>)
*Wireshark: Kali kept sending SYN to the restricted server receiving no response since OPNsense LAN blocked it by dropping the packet.*

Admin completed the TCP three-way handshake with the restricted server and after the handshake the SSH session was encrypted.

![Wireshark allowed SSH](<images/allowed admin to restricted tcp wireshark.png>)
*Wireshark: admin1's TCP three-way handshake, followed by encrypted SSH traffic.*

<br>

## Lessons

1. **Empty firewall log.** Live View stayed empty for about 40 minutes, even though Kali was blocked. The reason: "Log packets handled by this rule" wasn't checked on the LAN block rule. Turning it on made the block entries appear.
2. **Address vs network issue.** My first scanner block rule used a SCANNER address, which is only OPNsense's own IP on that network. It needed a SCANNER network, the whole subnet. The network is the whole street; the address is one house.
3. **Old rule reaching the new zone.** The existing scanner "pass to any" rule also allowed traffic into the new restricted zone. Instead of deleting it, which the scanner still needs, I put a block rule above because the first matching rule wins.
4. **Capture missing the handshake.** My first SSH capture had no SYN packets, because I was already logged in. I exited and captured a fresh login to get the full three-way handshake.
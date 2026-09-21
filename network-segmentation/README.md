# Home Lab: Network Segmentation with OPNsense

**Goal of this phase:** Deploy a firewall and network segmentation layer between the attack/target VMs (Kali, Metasploitable2) and the rest of the home network, then prove it actually works with a before and after test. This directly targets a "Network/Perimeter Security, including Next-Gen Firewalls" preferred qualification on a cybersecurity internship JD.

---

## Understanding the setup

OPNsense is a firewall and routing operating system. It's a full OS that dedicates an entire machine or VM to sit at a network boundary and control what traffic is allowed to pass through it.

**Required concept:**

Proxmox (the host) owns the physical hardware. VMs don't, so they can't create bridges on their own. Proxmox VE is a hypervisor, meaning it's the actual OS installed directly on your physical machine. It has direct access to the real physical NIC and the real RAM/CPU. Bridges are a feature of Proxmox's own networking stack, and they live in the host kernel.

A firewall inspects traffic and decides what's allowed based on rules you write. An example of a rule in this lab: I allow Kali and Metasploitable2 to reach each other on any port, but I block both of them from reaching my home network at all.

**Rundown of how a router works, so it's easier to follow my project:**

A router has a component called a switch, which handles the ethernet ports. It also has an access point for wifi. The router internally connects the switch and the access point together into one single network, making it the same subnet and the same broadcast domain. Having the same subnet and broadcast domain means everyone on it has direct access to everyone else.

Physically, a router has two networks meeting inside it, WAN and LAN. Normally those merge into one network.

WAN is the connection out to your ISP, the broader internet.

LAN is everything inside your house: wired and wireless devices, all bridged together into that one network I just described.

The reason I'm walking through this is to explain how I'm adding a second NIC that sits between two networks and only moves traffic between them when the firewall rules allow it.

A NIC is a network interface card. On real hardware it's the physical ethernet port, and in a VM it's a virtual version of that same thing. Think of the NIC as the plug that belongs to a VM. Each VM has one or more NICs, and each NIC needs somewhere to plug in.

My home lab, running Proxmox, emulates a network card and presents it to the VM as if it were real hardware. That virtual NIC connects to a bridge, something like vmbr0. It works like a virtual network switch, so Proxmox can also create additional bridges.

Here's the actual setup: NIC 1 (the WAN side) stays connected to vmbr0, my existing home network.

NIC 2 (the LAN, or segmented, side) connects to a new, separate bridge, vmbr1, which has no connection to vmbr0 at all. From there I move Kali's and Metasploitable2's NICs off vmbr0 and onto vmbr1.

That means changing Kali's and Meta's virtual bridge from the one tied to my host's actual physical NIC, over to vmbr1, a virtual bridge with no physical NIC attached. The key point is that this bridge exists only inside Proxmox's software. It's a second, fully isolated virtual switch that only VMs can plug into.

Because of that, vmbr1 can only reach the outside world (my home network) through OPNsense. OPNsense is the only VM sitting on both vmbr0 and vmbr1, so it's the single choke point everything has to pass through.

---

## Step 1: Platform choice, OPNsense vs pfSense

**Decision: OPNsense**

Why:
- Fully open source (BSD-licensed) with a predictable release cycle, so no CE-vs-Plus licensing confusion like the recent pfSense/Netgate changes.
- Modern interface, with Suricata IPS support built in.
- Resource requirements are about the same as pfSense at this scale, so this wasn't a resourcing decision either way.
- Worth acknowledging: pfSense still has more name recognition in older SOC job postings and tutorials. Either platform is a fine answer to "why this firewall" in an interview. I picked OPNsense for technical and licensing reasons, not because pfSense is worse.

---

## Step 2: ISO download and VM creation

- Downloaded the OPNsense installer ISO (dvd image type, amd64/OpenSSL) from opnsense.org.
- Transferred the compressed `.iso.bz2` file from my Windows machine to the Proxmox host over Tailscale using `scp`, into `/var/lib/vz/template/iso/`.
- Decompressed it with `bunzip2` on the Proxmox host and confirmed the resulting `.iso` showed up in Proxmox's ISO Images list.
- Created the VM (opnsense-fw) with:
  - OS type: Other (it's FreeBSD-based, so it's not in the Linux/Windows presets)
  - Disk: 8GB on local-lvm (NVMe tier), SCSI bus, VirtIO SCSI single controller, IO thread on, Discard on, no cache
  - CPU: 2 cores
  - Memory: 2048MB, ballooning disabled
  - Network: VirtIO model, with Proxmox's per-VM firewall feature left off (OPNsense is meant to be the actual firewall layer here, not Proxmox's built-in one)
  - Didn't start the VM until both network interfaces were in place (see Step 3)

---

## Step 3: Dual-interface network segmentation

- Created a second Proxmox bridge, vmbr1, under Datacenter, node, System, Network (Linux Bridge type). Left it with no IP address and no physical NIC attached (empty "Bridge ports" field), so it's a fully isolated virtual switch with no path to my real home network. Autostart on, VLAN aware off.
- Verified with `ip a` on the Proxmox host: vmbr1 only shows an auto-assigned IPv6 link-local address (`inet6 fe80::...`), no IPv4 address, and state `UNKNOWN`, which is expected for a bridge with nothing attached yet.
- Added a second network device to the OPNsense VM (Net1, bridge=vmbr1, VirtIO model), while its original Net0 stayed on vmbr0. Now OPNsense has one interface on each network.
- Shut down Kali and Metasploitable2, then edited each VM's existing network device (Net0) to change its bridge from vmbr0 to vmbr1, rather than adding or removing a device. Both VMs are now isolated from the main network at the bridge level, and OPNsense is the only VM attached to both bridges.
- Next up: install OPNsense from the ISO, assign WAN (vmbr0 side) and LAN (vmbr1 side) inside OPNsense itself, then bring Kali and Metasploitable2 back online on the new segment.

---

## Step 4: Firewall rule configuration and verification

**OPNsense installation:**
- Booted the opnsense-fw VM and ran the guided installer: declined LAGG and VLAN setup, then assigned interfaces by matching MAC addresses shown in the installer against Proxmox's Net0/Net1. Confirmed WAN = vtnet0 (on vmbr0, my real home network) and LAN = vtnet1 (on vmbr1, the isolated segment). Installed with UFS on the full virtual disk, set a root password, and rebooted into the installed system.
- LAN came up on OPNsense's default address (192.168.1.1/24) with DHCP enabled. That's fine since it doesn't need to match my home network's subnet, vmbr1 has no physical NIC and is fully isolated by design anyway. WAN was left on DHCP and picked up an address from my real home router after a reboot, which confirmed it could actually reach the real network.

**Baseline test (before the firewall rule):**
- From Kali, now on the LAN segment, confirmed it got a DHCP lease from OPNsense and could reach the OPNsense web interface at its LAN address.
- Ran **ping -c 4** **<u>&lt;home-router-IP&gt;</u>** from Kali. Result: 4 packets transmitted, 4 received, 0% packet loss. That confirmed the actual security gap: Kali could reach my real home network completely unrestricted through OPNsense's default configuration.

**Switching to default-deny:**
- OPNsense automatically creates a default "allow LAN to any" rule on new LAN interfaces, just for convenience. For this lab I removed that default entirely instead of trying to reorder rules around it. That's a deliberate default-deny approach, which is the more secure model used in real enterprise firewalls: nothing is allowed unless you explicitly say so.

**Block rule configuration (Firewall, Rules, LAN, IPv4):**
- Action: Block
- Interface: LAN
- TCP/IP Version: IPv4
- Protocol: ICMP
- Source: LAN net
- Destination: WAN net
- Description: "Block LAN to WAN ICMP, segmentation test"
- Logging: enabled

**Verification test (after the firewall rule):**
- Re-ran the exact same command from Kali: **ping -c 4** **<u>&lt;home-router-IP&gt;</u>**.
- Result: 4 packets transmitted, 0 received, 100% packet loss.
- That confirms the rule is doing what it's supposed to. Kali's traffic to my home network is now actively blocked at the firewall, not just configured on paper but actually proven with a real before and after test.

**Going from one protocol to full lockdown:**

Once I confirmed the ICMP-only rule worked, I added a second, broader rule on top of it instead of replacing it, so I could show both protocol-specific control and full defense in depth:

- Action: Block
- Interface: LAN
- TCP/IP Version: IPv4
- Protocol: any
- Source: LAN net
- Destination: WAN net
- Description: "Block LAN to WAN all protocols, full segmentation"

This rule blocks all IPv4 traffic (TCP, UDP, ICMP, everything) from the LAN segment to my home network, not just ping. I'm deliberately holding off on adding internet-access exceptions (like for Kali tool updates) until I actually need one, so every rule in this lab has a real reason behind it instead of being speculative.

**Full-protocol verification (three separate tests from Kali):**

| Test | Protocol exercised | Result |
|---|---|---|
| **ping -c 4** **<u>&lt;home-router-IP&gt;</u>** | ICMP | 4 sent, 0 received, 100% loss |
| Loading a webpage in the browser | TCP | Page failed to load, timed out |
| `nslookup example.com` | UDP | Lookup timed out, no resolution |

All three failed like I expected, which confirms the "any" rule blocks LAN-to-WAN traffic across every protocol, not just the ICMP case I tested earlier.

**IPv6 scope check (a known limitation, not a vulnerability):**

While testing, I asked myself a fair question: since my rules were written for IPv4 specifically, could IPv6 traffic just walk right past all of this? I checked it directly instead of assuming either way:

- Ran `ip a` on Kali and found only a `fe80::...` address on the LAN-facing interface. That's a link-local IPv6 address, auto-assigned by the OS for same-segment communication only. It's not a routable, internet-capable address.
- Confirmed with `ping -6` to the home router's address: 6 sent, 0 received, 100% loss.
- Conclusion: this isn't a firewall bypass. IPv6 isn't fully configured for routing on this segment, so there's no functional IPv6 path for anything to travel through in the first place. Not having an IPv6-specific block rule is a scope limitation of this test (IPv4 only), not an active security gap, since there's currently no live IPv6 channel to exploit.
- Noting this here as a known limitation and something to revisit if IPv6 ever gets enabled on this segment.

---

## Step 5: Summary of what was built and verified

**What I built:** A second, isolated firewall layer using OPNsense, sitting between two intentionally vulnerable or offensive lab VMs (Kali Linux, Metasploitable2) and my real home network. This mirrors how a SOC or network security team segments high-risk systems (like a penetration-testing box or an intentionally vulnerable target) away from production or personal infrastructure, so a compromise of one doesn't automatically expose the other.

**Why it matters:** Without segmentation, an attacker who compromised Metasploitable2 (a deliberately vulnerable machine) or misused Kali could pivot from that foothold straight into my real home devices on the same network. Segmentation contains that risk. Even a fully compromised VM on the isolated LAN segment can't reach the home network, because the firewall boundary stops it cold.

**What I actually verified, not just configured:**
- A baseline test proved the security gap existed before I added any rule (Kali could reach my home router freely).
- I created and applied a single-protocol (ICMP) block rule, then proved it worked with a before and after ping test.
- I removed the default "allow LAN to any" rule in favor of an explicit default-deny model, the standard, more secure approach used in real firewall deployments, instead of leaving broad implicit trust in place.
- I added a second, broader rule (all protocols) on top of the ICMP one and independently verified it across three different protocol types (ICMP, TCP, UDP), rather than assuming it worked off a single test.
- I found a potential blind spot (IPv6), investigated it, and confirmed it was a non-issue in this specific setup since there's no routable IPv6 path in the first place, instead of either ignoring it or wrongly assuming it was a risk.

**Gaps I ran into, and how I worked through them:**
- **Rule ordering vs. default-deny:** OPNsense's default "allow" rule needed to be dealt with before my block rule could actually take effect, since rules are evaluated top to bottom and it's first match wins. Instead of fighting with the reordering UI, I just removed the default rule entirely, which is honestly the more correct, more secure practice anyway.
- **Understanding traffic direction:** I was initially confused about which direction was actually open (LAN to WAN) versus which was already closed by default (WAN to LAN). I worked through it methodically until it clicked: OPNsense's WAN interface blocks unsolicited inbound by default, while LAN was open outbound until I explicitly restricted it.
- **Protocol scope of a single rule:** Once I realized an ICMP-only block doesn't stop other traffic like web browsing, I deliberately layered a second, broader rule instead of assuming one rule covered everything.
- **IPv6 as a possible bypass:** Instead of assuming IPv6 was either safe or a vulnerability, I checked it directly, first the interface addressing, then a live ping-6 test, and confirmed it was out of scope for this phase since there's no functional IPv6 routing in place.
- **Kali credential lockout mid-build:** I lost Kali's login credentials partway through this session. I recovered it without ever touching the VM's own console, working entirely from the Proxmox host instead. I used `libguestfs-tools` to inspect the VM's disk offline (`virt-cat` to confirm the username), then reset the password directly in the disk image with `virt-customize`, without booting the VM into a live or recovery environment at all. Cleaned up the extra tools from the host afterward.

**Result:** I now have a working, independently verified two-tier segmentation boundary (default-deny WAN, default-deny LAN with explicit rules) between offensive/vulnerable lab systems and my home network. Both a narrow example rule (ICMP) and a full-lockdown rule (all protocols) are documented, tested, and explained, including a properly scoped limitation (IPv6) instead of an unexamined one. I also confirmed there are no unintended paths around the firewall: vmbr1 has no direct route to WAN except through OPNsense itself, so these block rules are the only thing standing between LAN and my home network.

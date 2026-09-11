# Virtualized security homelab

Three VMs in VMware Workstation Pro. pfSense at the edge, a Windows Server 2022 domain controller behind it, an Ubuntu box on the same protected segment.

I built it so I'd have somewhere to break things. Most of what's written up here is stuff that went wrong and what it took to get out of it, because honestly that was most of the work.

## Topology

```
                Physical host (Windows 11)
                          |
                VMnet8 (NAT)  ->  Internet
                          |
                +---------v-----------+
                |     pfSense CE      |
                |        2.7.2        |
                |  WAN: VMnet8        |
                |  LAN: 192.168.247.2 |
                +---------+-----------+
                          |
              VMnet2 (host-only, isolated)
                          |
       +------------------+------------------+
       |                                     |
+------v----------+                +---------v---------+
|      DC01       |                |    ubuntu-lab     |
| Win Server 2022 |                | Ubuntu Server     |
|  192.168.247.10 |                |  192.168.247.20   |
|   ad.lab.lan    |                | OpenSSH, key-only |
+-----------------+                +-------------------+
```

| Host | Role | Address |
|---|---|---|
| pfSense CE 2.7.2 | Edge firewall, DHCP, DNS resolver | 192.168.247.2/24 on LAN, VMnet8 NAT on WAN |
| DC01 | Windows Server 2022, AD DS for `ad.lab.lan` | 192.168.247.10, static |
| ubuntu-lab | Ubuntu Server 26.04.1, SSH target | 192.168.247.20/24, static |
| Host adapter | VMware's own VMnet2 interface | 192.168.247.1 |

Both servers point at 192.168.247.2 for gateway and DNS. DHCP hands out .100 through .200, so the statics sit outside the pool and nothing fights over a lease.

## Why host-only and not bridged

Bridged would have dropped every VM straight onto my home network. No thanks.

On VMnet2 the guests can't touch the physical LAN at all, so the only way out is through pfSense, which means there's one place to look when something breaks and one place to change when I want to see what a rule actually does.

One thing to know if you try this: VMware takes .1 on host-only segments for its own adapter. I spent a while confused about why .1 wasn't available before I worked that out.

## pfSense

Two adapters. WAN on VMnet8, LAN on VMnet2.

The install was more annoying than it should have been. ZFS failed against the virtual disk a few times before I gave up and went with UFS on a SATA controller, and I never did figure out exactly why.

LAN side is a static 192.168.247.2/24, DHCP from .100 to .200, and the DNS Resolver in forwarding mode so guests resolve through the firewall instead of reaching out on their own.

To check the chokepoint is actually holding I ping the firewall from a guest, run `nslookup` and confirm the answer comes back from 192.168.247.2, then open the state table in pfSense and look for the connection. If DNS resolves but nothing shows up in the state table, something is getting around the firewall and the adapter settings are the first place I go.

## DC01

Windows Server 2022 Standard Evaluation, single adapter on VMnet2, nothing facing outward. Static .10 with gateway and DNS pointed at pfSense, then promoted to a domain controller for `ad.lab.lan`.

I picked `.lan` on purpose. It isn't a real TLD, so nothing in here can wander off and resolve against a domain I don't own.

## ubuntu-lab

Static .20 on VMnet2, OpenSSH installed, then keys only with passwords off.

That last part took way longer than it should have. I wrote the config, restarted sshd, and password logins still worked. Checked the syntax. Restarted again. Still working. The file had been fine the entire time, and the problem was that everything in `/etc/ssh/sshd_config.d/` loads in lexical order with the first matching directive winning, so a `50-cloud-init.conf` snippet was sorting ahead of mine and quietly overriding it. Renaming my file to sort first fixed it immediately.

Next time I hit something like that I'll run `sshd -T` before anything else. It prints what the daemon actually ended up with, which would have saved me the whole detour.

## Getting the hypervisor stable

If I'm honest this was most of the project.

VMware wouldn't run reliably next to Windows' own virtualization stack. Turning off Hyper-V, Memory Integrity, and the hypervisor launch type sorted that out, but later a corrupted VMware kernel driver left the machine unable to boot at all. I got it back with a USB installer and Startup Repair, then repaired the EFI boot files with `bcdboot` instead of reinstalling Windows.

The board is a Gigabyte Z890 Aorus Elite WiFi7 and it fought me before any of that happened. Debug code 61 with the DRAM LED lit on first boot, which came down to seating plus a CMOS reset. Then repeated BSODs, which turned out to be AMD chipset drivers still sitting on a drive I'd reused from the old build. Clean install. A BIOS update from F9 to F20 through Q-Flash cleared up what was left.

Secure Boot took its own afternoon. Restore factory keys, set Key Management to Standard, turn on Intel PTT for TPM 2.0.

## Entra ID

There's a Microsoft Entra ID tenant running alongside the on-prem domain, mostly so I have a cloud directory to work against as well as a traditional one.

## Still to do

- ufw rules on ubuntu-lab, locked to the LAN subnet
- Docker on ubuntu-lab, Pi-hole first
- a Kali VM and a documented nmap sweep of the segment
- Conditional Access and an MFA policy in the Entra tenant

Built by Aiden Guyton. IT student at Towson University, Network Security concentration.

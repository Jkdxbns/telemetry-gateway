# Concepts cheat sheet

Each topic: the idea, where it showed up, and what to say. Stories are in `CHALLENGES.md`, commands
in `TOOLS.md`. Resources are titles to search for, in one block at the end.

## Our network

```
 main router (192.168.1.1, DHCP) ── wall port ── Linksys LAN port (switch/AP mode, 192.168.1.2)
        │                                              ├── port 1: BBB  eth0  192.168.1.101
        │ Wi-Fi                                        └── port 2: Jetson Ethernet
        ├── laptop        192.168.1.214
        └── Jetson Wi-Fi  192.168.1.110

 BBB usb0 ── USB cable ── Jetson   (separate link, 192.168.7.0/30)
```

Everything on `192.168.1.0/24` is one network. The Linksys only adds ports and a radio.

## 1. Subnets
- **Idea:** an address plus a mask defines the local network. `/24` is 256 addresses; `/30` is 4 (2 usable).
  Private ranges: `10/8`, `172.16/12`, `192.168/16`. `169.254/16` is link-local; `100.64/10` is Tailscale's.
- **Showed up as:** the Linksys and main router both at `192.168.1.1`; the laptop's Wi-Fi and wired
  adapter on one `/24`.
- **Say it:** two interfaces on one subnet are ambiguous because the kernel can't tell which one is "local".

## 2. Router vs switch vs access point
- **Idea:** a switch forwards by MAC on one network (layer 2); a router joins networks by IP (layer 3);
  an access point is a switch with a radio. A home router bundles all of them plus DHCP and NAT. The
  Linksys has **LAN** ports (the switch) and an **Internet** port (the router's outside).
- **Showed up as:** the Linksys in switch/AP mode behind the main router.
- **Say it:** the Internet port would add NAT and hide the BBB from the laptop; one cable between two
  switches only, because a second makes a loop.

## 3. DHCP
- **Idea:** a server leases a client an address, gateway and DNS. Discover, Offer, Request, Acknowledge
  (UDP 67 and 68).
- **Showed up as:** the Linksys's DHCP running beside the main router's, so it had to go off.
- **Say it:** two DHCP servers on one segment is a rogue server and gives clients conflicting settings.

## 4. ARP
- **Idea:** maps an IP to a MAC on the local network. A broadcast asks "who has X?", the owner
  answers, and the answer is cached.
- **Showed up as:** the neighbor table showing which device answered `192.168.1.1` on which interface.
- **Say it:** ARP doesn't cross a router; `FAILED` means no answer; ARP spoofing lies in the cache.

## 5. NAT
- **Idea:** a router rewrites source addresses (and ports) so private hosts can reach out, remembers
  each connection, and rewrites replies back. It blocks unsolicited inbound traffic.
  **MASQUERADE** is source NAT where the outside address can change.
- **Showed up as:** the MASQUERADE rule in ufw; why `192.168.7.2` means nothing outside its link.
- **Say it:** NAT isn't a firewall; double NAT breaks inbound connections.

## 6. Firewalls
- **Idea:** ufw writes iptables rules into tables and chains. A stateful rule ("accept ESTABLISHED,
  RELATED") lets replies back in; a policy of DROP blocks everything else. Each table is applied at
  its `COMMIT`, so a syntax error aborts the load.
- **Showed up as:** the whole ufw incident (CHALLENGES C2).
- **Say it:** `restart` can skip loading when chains already exist; `reload` forces a rebuild.

## 7. The `/sys` tree
- **Idea:** a virtual filesystem where the kernel shows its live device model: buses, devices and
  drivers. `/sys/bus/<bus>/devices` and `.../drivers` list them; each device has a `driver` link to
  its owner. `/sys/class/<type>` groups by function (e.g. `net`); `/sys/devices` is the real tree.
  Reading a file asks the kernel for a value; writing can change a setting. One driver owns a device
  at a time. `/dev` file permissions come from udev rules (here `/dev/i2c-2` is group `gpio`).
  Debug files live under `/sys/kernel/debug` (debugfs), e.g. pinctrl.
- **Showed up as:** checking whether a driver owns the MPU6050; reading pad ownership in pinctrl.
- **Say it:** bus, device, driver. When a device and a driver match, the kernel calls `probe()`.

## Also met
- **Routing metrics:** the most specific route wins; ties go to the lower metric.
- **DNS and mDNS:** DNS maps names to addresses; `.local` names use multicast DNS.
- **Tailscale:** an overlay VPN with stable `100.x` addresses; it worked when the LAN path was blocked.
- **NetworkManager:** it only runs DHCP through a connection profile.
- **UART and USB-gadget links:** see `docs/01-learning/NOTES.txt`.

## Resources
- **Books:** Kurose and Ross, *Computer Networking: A Top-Down Approach*; Stevens and Fall,
  *TCP/IP Illustrated Vol. 1*.
- **Video:** Professor Messer (Network+), Chris Greer (Wireshark), Ben Eater (networking),
  Practical Networking (fundamentals), Cisco Networking Academy (Networking Essentials).
- **Reading:** DigitalOcean's "A Deep Dive into Iptables and Netfilter Architecture"; Julia Evans's
  networking zine; Arch Wiki on iptables and ufw.
- **RFCs:** 826 (ARP), 1918 (private addresses), 2131 (DHCP), 3022 (NAT), 4632 (CIDR).

## Self-check, answer unaided
1. Why was the Internet port wrong for this setup?
2. Why avoid a second DHCP server on one segment?
3. Why did `ufw restart` skip a corrected file, and what did `reload` do?
4. What does NAT do to a reply packet?
5. Why do two interfaces on one subnet cause ARP trouble?
6. What does an `INPUT` policy of DROP with no ESTABLISHED,RELATED rule break?
7. Why can a device have link but no IP?
8. Why was UART4 silent while UART0 worked?
9. What do `/proc`, `/sys` and `/dev` each show?

# Tools cheat sheet

Read the one-line idea, then the commands. Stories are in `CHALLENGES.md`, ideas in `CONCEPTS.md`.

## `ip`: shows the kernel's real network state
Replaces `ifconfig`, `route` and `arp`. `UP` means enabled; `LOWER_UP` means a link is detected.

| Command | What it does |
|---|---|
| `ip -brief addr` | One line per interface: state and addresses (run this first) |
| `ip route` | Routing table: destination, gateway, interface, metric |
| `ip route get <addr>` | Which route and interface the kernel would use |
| `ip -d link show <if>` | MAC, flags, driver, parent bus |
| `ip -s link show <if>` | Packet and error counters |
| `ip neigh` | The ARP/neighbor table (below) |

## ARP, through `ip neigh`: IP to MAC on the local network
| State | Meaning |
|---|---|
| `REACHABLE` | Recently confirmed alive |
| `STALE` | Known, not confirmed lately |
| `DELAY` / `PROBE` | Being re-checked now |
| `INCOMPLETE` / `FAILED` | Asked, never answered: not there or unreachable |

Used it to prove which device answered `192.168.1.1` on each interface.

## `nmcli`: NetworkManager
**Devices** are hardware; **connections** are saved profiles. DHCP lives in a connection, so a device
with no profile gets no address. `unmanaged` means NetworkManager leaves it alone.

| Command | What it does |
|---|---|
| `nmcli device status` | Devices and their connections |
| `nmcli device show <if>` | Addresses, gateway, DNS, reason codes |
| `nmcli connection show` | Saved profiles |
| `nmcli radio wifi off` / `on` | Wi-Fi radio on or off |
| `nmcli connection add type ethernet ifname <if> con-name <name> ipv4.method auto` | New DHCP Ethernet profile. Used 2026-10-07: gave the Jetson's `enP8p1s0` `192.168.1.170` |
| `nmcli connection up <name>` | Activate a profile. Used on the Jetson; the connection came up |

## `ufw` and `iptables`: the firewall
ufw writes iptables rules into chains (`INPUT`, `FORWARD`, `OUTPUT`, `POSTROUTING`). Files:
`/etc/ufw/before.rules` (raw rules, where our NAT block lives), `user.rules` (from `ufw allow`),
`/etc/default/ufw` (default policies), `/etc/ufw/sysctl.conf` (forwarding).

| Command | What it does |
|---|---|
| `ufw status verbose` | Rules and default policies |
| `ufw reload` | Reload everything from the files |
| `journalctl -xeu ufw.service` | Why the service failed |
| `iptables -S INPUT` | The `INPUT` chain and its policy |
| `iptables -S ufw-before-input` | What ufw actually loaded |
| `iptables -t nat -S POSTROUTING` | The NAT rules |
| `cat /proc/sys/net/ipv4/ip_forward` | Is the kernel forwarding (1 = yes) |

The NAT block, line by line:
```
*nat                                        # start the nat table
:POSTROUTING ACCEPT [0:0]                   # declare the chain
-A POSTROUTING -o wlP1p1s0 -j MASQUERADE    # rewrite the source of packets leaving via Wi-Fi
COMMIT                                      # apply it (required per table)
```

## `i2cdetect` and `i2cget` (i2c-tools): talk to an I2C bus from userspace
Installed in `/usr/sbin` (I added it to `PATH` in `~/.bashrc`). They use `/dev/i2c-N`, so you need the
`i2c` group. Neither needs a kernel driver, and neither works if one owns the chip.

| Command | What it does |
|---|---|
| `i2cdetect -l` | List the I2C buses |
| `i2cdetect -y -r 2` | Scan bus 2. `68` = something answered at 0x68; `UU` = a kernel driver owns it |
| `i2cget -y 2 0x68 0x75` | Read one register: bus 2, device 0x68, register 0x75 (`WHO_AM_I`, should be `0x68`) |

`i2cdetect` only proves an address acknowledged. `i2cget` proves real data came back. (`i2cset`
writes and `i2cdump` dumps registers; neither used yet.) Pick the right pins first: I2C2 is P9.19 and P9.20.

## `ethtool`: Ethernet link, speed and duplex
Queries and configures the NIC and its driver. Used to record the BBB's `eth0` link.

| Command | What it does |
|---|---|
| `ethtool eth0` | Link detected, speed, duplex, auto-negotiation, what the other end offers |
| `ethtool -i eth0` | Driver, version, bus |
| `ethtool -S eth0` | NIC counters: errors and drops |
| `ethtool -s eth0 ...` | Force speed or duplex (root; can drop the link) |

Expect `Speed: 100Mb/s`, `Duplex: Full` on the BBB (a 10/100 NIC). A duplex mismatch shows as errors.

## Quick tools
| Command | What it does |
|---|---|
| `ping -c N <host>` | Reachability and round-trip time |
| `curl -s -o /dev/null -w "%{http_code}" <url>` | HTTP status only (`-u user:pass` for Basic auth, `--interface <if>` to force a path) |
| `timeout 4 bash -c '</dev/tcp/<host>/22'` | Is a TCP port open |
| `getent hosts name.local` | Resolve a name the way programs do (works for mDNS) |
| `ssh -o BatchMode=yes <host> '<cmd>'` | Run one command over key authentication |
| `sg <group> -c "<cmd>"` | Run one command with a group applied |
| `screen <dev> 115200` | Serial console (`Ctrl-A` then `k` to quit) |
| `tmux` | Keeps a session alive across SSH drops |

**Habits that paid off:** run read-only checks first, compare the config file with the live state, and
never type into a pane someone else is using.

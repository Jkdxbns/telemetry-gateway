# Challenges: what broke, why, and what it taught

Three stories for "tell me about a hard bug". Each has the symptom, the real cause, the evidence that
proved it, the fix, and a one-line way to say it. Commands are in `TOOLS.md`; the ideas behind them
are in `CONCEPTS.md`. No passwords, passphrases or MAC addresses are recorded on purpose.

| # | Challenge | Status |
|---|---|---|
| C1 | Silent serial console: wired to UART4 instead of UART0 | Fixed |
| C2 | ufw NAT edit broke the firewall, and the Jetson couldn't talk out | Fixed |
| C3 | Linksys WRT54G admin page "wouldn't load" | Fixed (the browser, not the router) |

---

## C1. Serial console showed nothing
- **Symptom:** the Pico bridge (`/dev/ttyACMx`) stayed blank even after swapping TX/RX and
  confirming GND.
- **Cause:** the jumpers were on UART4 (the general-purpose P9 header). The kernel console is fixed
  to UART0 (`console=ttyS0,115200n8` in `/proc/cmdline`), which is exposed only on the separate 6-pin
  debug header. Nothing listens on UART4 by default.
- **Evidence:** `/proc/cmdline` and `serial-getty@ttyS0.service` active on the BBB. The Pico side was
  confirmed from debugprobe's own source (GPIO4/5 on uart1).
- **Fix:** moved the jumpers to the debug header; `bbb login:` appeared at once.
- **Lesson:** check what is actually listening before blaming the wiring.
- **Say it:** "I confirmed the Pico side from the firmware source, then found the real fault was the
  wrong UART on the BBB."

## C2. ufw NAT edit broke the firewall; the Jetson couldn't talk out
- **Symptom:** `systemctl restart ufw` failed. After I fixed the file, restart "succeeded" but the
  Jetson still couldn't ping the router, resolve names or fetch pages. The laptop couldn't ping it
  either, though SSH on port 22 worked. My `jetson` shell function (ping first, then Tailscale)
  picked the wrong path or reported it offline.
- **Cause, in three steps:**
  1. My `*nat` block had no `COMMIT`. `iptables-restore` reached `*filter` first and aborted
     (`Bad argument '*filter'`).
  2. ufw clears its rules and creates empty chains before loading. The failed load left those
     chains empty, so the next start logged `Firewall already started, use 'force-reload'` and did
     nothing.
  3. With the `INPUT` policy on DROP and no "accept ESTABLISHED,RELATED" rule loaded, replies to the
     Jetson's own traffic were dropped. SSH still worked because ufw's user rules allow port 22,
     and Tailscale has its own rules.
- **Evidence:** `before.rules` on disk was correct, but `iptables -S ufw-before-input` held a single
  jump line. The ufw journal showed the `Bad argument` error and later the "already started" line.
- **Fix:** `ufw reload`, a force reload that rebuilds everything from the files.
- **Lessons:** compare the file on disk with the rules actually loaded. Test the service you care
  about (TCP 22), not ping, as a health check.
- **Say it:** "The symptom looked like a network fault, but the live firewall was empty after a
  failed load. Restart skipped loading; reload rebuilt it."

## C3. Linksys WRT54G admin page "wouldn't load"
- **Symptom:** `http://192.168.1.1` showed nothing, even after restarts.
- **Diagnosis, from the terminal:**
  - Wi-Fi was still on, so two interfaces shared `192.168.1.0/24`, and both the main router and
    the Linksys answer at `192.168.1.1`.
  - Wired-only checks showed the Linksys was healthy: a DHCP lease, ping, a MAC that belongs to
    Cisco-Linksys, HTTP `401` without a login and `200` with the default login.
- **Cause:** the browser. Chrome failed and Firefox worked. I didn't pin down exactly why. The
  candidates are a cached redirect, https-first behaviour, and Basic-auth handling on an old
  HTTP/1.0 server.
- **What I configured:** the router as a bridged switch and access point behind the main router: a
  new static LAN address so it no longer clashes with `192.168.1.1`, DHCP off, WPA2-AES, UPnP off.
  The wall cable goes into a **LAN** port, never the Internet port.
- **Result:** the BBB got an address from the main router's DHCP through the Linksys.
- **Lessons:** know which device answers an address before configuring it. Use `curl` to separate
  "router problem" from "browser problem". Change a router's address and DHCP *before* connecting
  it to a live network.
- **Say it:** "I proved the router was healthy from the terminal, then configured it as a bridged
  access point so it wouldn't compete with the main router."

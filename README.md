# Bridge Mode for LEDE / OpenWrt (4G IP passthrough)

> **Archived.** This was a proof of concept that ran in production in a small company network environment
> from **2017 to 2022** on a MikroTik RB912UAG-5HPnD running LEDE. It is no longer maintained and
> is kept for reference only.

> [!WARNING]
> `DNSMASQ-Modified/` is **dnsmasq 2.76 from 2017** and is **outdated and not safe to use**.
> Several serious vulnerabilities were fixed after this version, including remote code
> execution bugs in DNS and DHCP (e.g. CVE-2017-14491 and the others fixed in 2.78, and the
> "DNSpooq" bugs fixed in 2.83). If you want to rebuild this today, port the patch to a current
> dnsmasq release.

Many LTE routers have a "bridge mode" or "IP passthrough", where the modem simply hands the IP
address it got from the mobile operator to the router behind it. That router then handles NAT, firewalling and
the internal networks as if it had a normal WAN connection. LEDE/OpenWrt had no such feature at
the time, so this repository adds one.

## Why a normal bridge doesn't work

With an Ethernet uplink (cable or DSL modem), bridging is easy, because the router behind the modem just
sends a DHCP request and the ISP's DHCP server answers through the bridged interfaces.

LTE doesn't work that way. The modem gets its IP address from the network while it sets up the
data connection (the bearer), not via DHCP, and the link to the host carries raw IP packets
with no Ethernet frames, no ARP, and no DHCP server on the other side. There is nothing to bridge
to and nobody to answer a DHCP request from the downstream router.

So the LEDE box has to fake it. It reads the IP and gateway the operator has currently assigned,
runs a DHCP server that hands out that IP, and routes all traffic through. When the operator
assigns a new IP, it updates everything and hands out the new one.

## Hardware

| | |
|---|---|
| Board | MikroTik RB912UAG-5HPnD with OpenWRT/LEDE |
| LTE module | Sierra Wireless MC7455, later Sierra Wireless EM7565 |
| Module mode | **MBIM**, since the scripts use `umbim` on `/dev/cdc-wdm0` and the `mbim` netifd protocol. QMI is not supported with this hack, but it would be possible to implement |

## How it works

```
                 ((( ISP / mobile cell tower )))
                              |
                              |  4G / LTE (MBIM)
                              |  operator assigns a dynamic IP, e.g.
                              |    IP 203.0.113.42/30, GW 203.0.113.41
                              v
 +----------------------------------------------------------------+
 |  MikroTik RB912 running LEDE  (the "modem")                    |
 |                                                                |
 |  wwan0 (4G module)                                             |
 |    - keeps the current WAN IP, but its local route is removed, |
 |      so packets for that IP are forwarded instead of consumed  |
 |                                                                |
 |        ^  policy routing:                                      |
 |        |    in on eth0  -> out wwan0                           |
 |        v    in on wwan0 -> out eth0                            |
 |                                                                |
 |  eth0 (lan)                                                    |
 |    - 192.168.8.138/24   static management address              |
 |    - 203.0.113.41/30    operator gateway IP, copied here       |
 |    - patched dnsmasq offers the current WAN IP via DHCP:       |
 |        203.0.113.42, gateway 203.0.113.41                      |
 +----------------------------------------------------------------+
                              |
                              |  Ethernet, plain DHCP
                              v
 +----------------------------------------------------------------+
 |  Main router / firewall  (NAT, ACLs, VPN, ...)                 |
 |    WAN: 203.0.113.42   <- the operator's (public) IP           |
 |    internal networks, e.g.                                     |
 |      192.168.10.0/24  office                                   |
 |      192.168.20.0/24  servers                                  |
 |      192.168.30.0/24  guests                                   |
 +----------------------------------------------------------------+
```

*(Addresses are examples.)*

**The funny thing** is that the public IP lives in two places at once, on the LEDE box's `wwan0` *and* on
the main router's WAN port. That only works because the LEDE box deletes its own local route for
that IP, so it never answers for it and just forwards everything. On top of that, the LEDE box
takes the operator's gateway address on its LAN side. The main router sees the gateway the
operator assigned and believes it is connected straight to the mobile network, while the operator
only ever sees the IP it handed out. The LEDE box sits invisibly in between.

So it isn't a real layer-2 bridge, but a combination of routing and DHCP, built from four pieces.

1. **`mbim.sh`** brings up the 4G connection and passes the operator's IP, netmask and
   gateway to netifd correctly.
2. **`/etc/init.d/dnsmasq`** reads two extra UCI options, `bridge_mode` and `bridge_interface`.
   Every time it starts, it reads the current WAN IP, mask and gateway from netifd and then
   - puts the operator's gateway IP on the LAN interface,
   - removes the WAN subnet route and the local route for the WAN IP,
   - adds two policy routing tables (42/43) so everything coming in on one side goes out the other,
   - starts dnsmasq with a DHCP range containing only the current WAN IP, and the operator
     gateway as the default route.
3. **Patched dnsmasq 2.76**. Stock dnsmasq won't offer an address that the host itself already
   holds, and it pings an address before offering it. Both checks are commented out in
   `src/dhcp.c`.
4. **`wan_watcher`** handles the dynamic IP. It polls the modem every 5 seconds. If the WAN IP
   changes, it restarts dnsmasq (which picks up the new IP) and bounces the LAN link so the main
   router asks for a new lease. If the connection drops, it resets the USB modem and reconnects.
   Together with the 2-minute lease, the main router has the new address within about two
   minutes.

## Reaching the LEDE box itself

For management, the box stays reachable on its static address (`192.168.8.138` here) on the
LAN side. The main router has a second address, `192.168.8.1/24`, on its WAN port, so internal clients
can reach the box through it.

**Internet access for the box** (package updates, NTP, ...) works, but takes a detour. The public
IP belongs to the main router, so the box can't just use it for its own traffic. Instead, the
box's default route points to the main router. The main router NATs that traffic like any other
internal host and sends it back through the box to the LTE network, like this. It looks odd but worked fine.

```
 +--------------------+   1. own traffic (updates, NTP) +--------------------+
 |  LEDE box          | ------------------------------> |  main router (NAT) |
 |  192.168.8.138/24  |      default gw 192.168.8.1     |  192.168.8.1/24    |
 |                    | <------------------------------ |  WAN 203.0.113.42  |
 +--------------------+   2. NATed to 203.0.113.42      +--------------------+
           |
           |  3. forwarded to LTE with source 203.0.113.42
           v
      LTE network
```


## Repository layout

| Path | What it is |
|---|---|
| `LEDE-Scripts/lib/netifd/proto/mbim.sh` | MBIM protocol handler that sets IP, mask and gateway properly |
| `LEDE-Scripts/etc/init.d/dnsmasq` | Replacement dnsmasq init script with the `bridge_mode` logic |
| `LEDE-Scripts/etc/init.d/wan_watcher` | procd service that runs the watcher |
| `LEDE-Scripts/root/wan_watcher.sh` | Reconnect / IP change watchdog (interfaces are hardcoded) |
| `DNSMASQ-Modified/` | dnsmasq 2.76 **source code** with the patches described above (outdated, see warning) |

## Setup (as it was done back then)

1. **Back up** every file you replace.
2. **Management interface.** Give the LAN interface a static IP so you can still reach the box.
   In `/etc/config/network` that looked like this.
   ```
   config interface 'lan'
           option ifname 'eth0'
           option proto 'static'
           option ipaddr '192.168.8.138'
           option netmask '255.255.255.0'
           option delegate '0'
   ```
3. **Replace** `/lib/netifd/proto/mbim.sh` with the one from this repo.
4. **Replace** `/etc/init.d/dnsmasq` and enable bridge mode in `/etc/config/dhcp` like this.
   ```
   config dhcp 'lan'
           option interface 'lan'
           option leasetime '2m'
           option ra 'server'
           option ra_management '1'
           option bridge_mode '1'
           option bridge_interface 'wan'

   config dhcp 'wan'
           option interface 'wan'
           option ignore '1'
   ```
5. **Compile the patched dnsmasq for the device.** `DNSMASQ-Modified/` is source code only. It
   has to be cross-compiled for the router's architecture (for the RB912 that is the LEDE
   `ar71xx` MIPS target), for example with the LEDE/OpenWrt SDK. Then replace `/usr/sbin/dnsmasq`
   on the device with the resulting binary. The `.o` files in `src/` are leftovers from a local
   build and won't run on the router. On startup the patched binary logs *"Attention: This is a
   custom build for Blaubeere BridgeMode"*.
6. **Install `wan_watcher`** (`/etc/init.d/wan_watcher` and `/root/wan_watcher.sh`). This is needed
   for dynamic IPs, which is the normal case with mobile operators. Adjust the hardcoded
   interfaces, device (`/dev/cdc-wdm0`) and USB ID (`1199:9071`), then enable the service.
7. **Reboot**, then set the main router's WAN port to DHCP.

## Caveats

- The LEDE firewall stays active. To make the main router reachable from outside, the firewall
  rules on the LEDE box have to accept/forward that traffic.
- Only IPv4. Only one downstream device (the main router) gets the public address.
- `wan_watcher.sh` is a quick hack with interface names, device paths and the modem's USB ID
  hardcoded.
- Written for LEDE (2017) and dnsmasq 2.76. Current OpenWrt releases differ (e.g. `ifname` →
  `device`, different `mbim.sh`/dnsmasq init scripts), so don't expect a drop-in install.

## License

This repository is licensed under the [GNU General Public License v2.0](LICENSE). The scripts in
`LEDE-Scripts/` are partly based on OpenWrt/LEDE files, which are GPL-2.0 as well.

`DNSMASQ-Modified/` is a modified copy of [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html)
by Simon Kelley, licensed under GPL v2 or v3 (see `DNSMASQ-Modified/COPYING` and `COPYING-v3`).
The changes are marked with `BLAUBEERE` comments in `src/dhcp.c`, plus a startup log message in
`src/dnsmasq.c`.

Everything here is provided as is, without any warranty. Use it at your own risk.

# lwIP vendor notes (mp0rta/heiher-lwip)

Base: heiher/lwip @ `cd3007df7047555399a04e6e40214accfae0d324`, lwIP core
version 2.2.1 (both verified 2026-07-03; if the fork was re-synced since,
re-verify against the pinned tree's lwip/init.h version string).
License: BSD-3-Clause (SICS / Adam Dunkels), see LICENSE.

## What this fork adds over upstream lwIP
Wildcard TCP/UDP interception for transparent proxying:
- `NETIF_FLAG_PRETEND` (src/include/lwip/netif.h) + `pretend_netif_idx`
  pcb field (src/include/lwip/tcp.h, TCP_PCB_COMMON).
- `tcp_bind(pcb, NULL, 0)` on a pretend netif records the netif index;
  `tcp_input`'s listener lookup matches ANY destination IP/port arriving on
  that netif (src/core/tcp_in.c). UDP has mirror logic (src/core/udp.c).
- UDP pretend-path fixes: "Allow receiving packets are not destined to
  localhost" (src/core/udp.c, ipv4/ip4.c, ipv6/ip6.c), plus later
  use-after-free and destination-address-passing fixes, also in
  src/core/udp.c.
- Perf/capacity config bumps live in `src/ports/include/lwipopts.h`
  (MEMP_NUM_TCP_PCB 4096, MEMP_NUM_UDP_PCB 1024, `LWIP_TIMERS 0` to drop
  `sys_timeout`/internal cyclic timers under NO_SYS=1). This file is under
  `src/ports/`, which mqvpn does NOT vendor (see below) — treat these as
  reference values, not inherited config; mqvpn's own port layer sets its
  own equivalents.

## What mqvpn does NOT vendor from this fork
`src/ports/` (the hev-socks5-tunnel port layer) is excluded: `src/ports/lib/mem.c`
carries a bare "Copyright (c) 2021 hev" header with no license grant text,
which does not meet mqvpn's BSD-3-clean bar. mqvpn writes its own port layer
(`src/hybrid/lwip_port/` in the mqvpn repo) for NO_SYS + tick-driven timing —
needed anyway for mqvpn's sans-I/O integration, so nothing is lost.

## Sync policy
Same model as `third_party/xquic`: branch `mqvpn-main` tracks what mqvpn
builds against; sync from upstream heiher/lwip via a merge/rebase PR when
picking up fixes (heiher/lwip is actively maintained, so this should be a
routine, low-conflict operation, not a rare one), tag `vendor-base-<sha>`
marks the point this fork diverged from a clean upstream sync. `main` stays
a clean mirror of heiher/lwip main (GitHub "Sync fork"). Note heiher/lwip is
itself a standalone repo (not a GitHub network fork of lwip-tcpip/lwip), so
upstream-lwIP fixes reach it only by heiher's manual sync; for an urgent
lwIP security fix, cherry-pick directly from lwip-tcpip/lwip onto mqvpn-main
without waiting for heiher.

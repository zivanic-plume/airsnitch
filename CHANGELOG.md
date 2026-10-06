# AirSnitch — Plume fork changelog

Changes in the Plume fork (**zivanic-plume/airsnitch**) relative to the upstream
original (**vanhoefm/airsnitch**).

- **Upstream base (fork point):** `a578548` — *"README: Tweaked enterprise clarification"*, 2026-03-13.
  Upstream `airsnitch/research/airsnitch.py` at that point: md5 `31b06f4b`, 1561 lines.
  It already had: `--c2c` (ARP), `--c2c-eth`, `--c2c-ip` (gateway bounce), `--c2c-broadcast`,
  `--c2c-port-steal` (downlink + uplink), `--c2c-gtk-inject` (ICMP payload only),
  `--check-gtk-shared`, `--same-bss`/`--other-bss`, PMK export/import, `--measure`, `--reinject-gtk`.
- **Current working copy:** md5 `dc46ad06`, 2169 lines (+608 vs upstream).
- **Fork HEAD on GitHub:** md5 `59af9f40` (commit `dfed18a`) — **behind** the working copy by the
  2026-10-06 fix below (not yet pushed).

Dates are commit dates from the fork's git history (authoritative); where local development
predated the commit, that is noted.

---

## 2026-10-06 — gtk-inject proof-gap fix  *(local working copy `dc46ad06`; NOT yet committed/pushed to the fork)*

Fixes false-VULNERABLE verdicts for `--c2c-gtk-inject` (re-scoring 144 captured logs showed
96 were false positives — scored on the shared-GTK precondition, not on proof of receipt).

- **`monitor_eth`**: the RA and DHCP-NAK receipt branches were gated on `--c2c-ra-inject` /
  `--c2c-dhcp-nak` (both `None` during a gtk-inject run), so a forged RA / DHCP-NAK that actually
  reached the victim was never detected. Un-gated them to also fire when `--c2c-gtk-inject` is set.
- Emit one uniform `>>> GTK wrapping <payload> is allowed …` receipt line **plus** the existing
  `>>> PROOF (frame received on the VICTIM): …` for every payload (icmp/arp/dhcp-nak/ra) the instant
  the forged frame crosses — so the verdict is driven by proof of crossing, never by a shared GTK alone.
- **Companion (separate repo — the test orchestrator `airsnitch_matrix.py`, not in this repo):**
  `classify()` made attack-aware: a gtk-inject cell is VULNERABLE only with a receipt line, SECURE on
  an explicit block, otherwise NOT-RUN; the shared-GTK rule now applies only to `check-gtk-shared`.

## 2026-10-03 — `dfed18a` *Stop c2c tests on detection; port-steal MAC restore + timeout*
- **`--auto-exit`**: end the test the instant a vulnerable condition is detected (and a shared
  `_finish_c2c_test` tail that prints an explicit `>>> RESULT: … reached / never reached` verdict
  for `--c2c-ip`/`-eth`/`-broadcast`/`-multicast`/`-ra`), instead of flooding until Ctrl-C.
- **Port-steal hardening**: `recover_port_bindings` (victim ARPs the gateway so the AP/switch
  re-learns the correct port after a steal), `--port-steal-timeout`, attacker-MAC restore on exit.

## 2026-10-02 — `8c744f8` *Add plaintext client-isolation tests: DHCP NAK + ARP-poison proof*
- **`--c2c-dhcp-nak`**: spoofed plaintext DHCP NAK (server→client) to force the victim to drop its
  lease; `--dhcp-nak-broadcast`, `--dhcp-nak-count`, `--dhcp-nak-wait`, `--dhcp-nak-xid-auto`
  (reuse the victim's learned XID in self-test). Explicit reached/never-reached RESULT.
- **ARP-poison tangible proof**: read the victim's real kernel ARP cache for the gateway BEFORE and
  AFTER (`_victim_neigh`, `--arp-poison-count`); verdict `POISONED` / `DELIVERED but NOT poisoned` /
  `BLOCKED` instead of just "a frame was seen".

## 2026-10-02 — `47d3c01` *Extend AirSnitch: PMF GTK injection, new payloads/tests, and wpa_supplicant fixes*
*(New tests developed locally from ~2026-09-21; consolidated into this commit. Per project
recollection, `--c2c-multicast` and `--c2c-ra-inject` were added first, lost in an intermediate
deployed checkout, and re-added here — the fork and later deployments carry them from this point.)*

- **wpa_supplicant (C)**: `wpas_glue.c` caches the data GTK and the IGTK separately so `GET_GTK` no
  longer returns the IGTK under PMF (SAE/PFA), excluding BIP keys; `ctrl_iface.c` +
  `wpa_supplicant_i.h` add a `GET_IGTK` control command + `last_igtk*` storage.
- **`--c2c-gtk-inject` PMF fix**: use the data GTK (guard against IGTK key id ≥ 4), report the IGTK,
  auto-tune the injection iface to the victim's frequency, learn the live GTK PN (provoke a group
  frame) and inject just above it with an incrementing-PN burst, persisting the PN per BSSID across
  runs (`learn_group_pn`, `_read/_write_pn_state`, `_inject_burst`, `_provoke`).
- **`--gtk-inject-payload {icmp,arp,dhcp-nak,ra}`** (+ `--gtk-inject-dhcp-xid`,
  `--gtk-inject-pn`, `--gtk-inject-pn-timeout`, `--gtk-inject-repeat`, `--no-gtk-inject-provoke`).
- **`--c2c-multicast`** (+ `--mcast-mac`) — multicast reflection test.
- **`--c2c-ra-inject`** (+ `--ra-inject-dns`, `--ra-inject-count`, `--ra-lifetime`, `--ra-managed`,
  `--ra-other`) — spoofed ICMPv6 RA with a malicious RDNSS (IPv6 DNS hijack).
- Fix default route for off-subnet / captive-portal gateways (`ip route … onlink`).
- Fail cleanly on open networks (no GTK) instead of crashing; strip trailing newline in GTK/IGTK logs.

---

### To bring the GitHub fork up to date
The fork HEAD (`59af9f40` / `dfed18a`) is missing the **2026-10-06 gtk-inject proof-gap fix** that is
in the working copy (`dc46ad06`). Commit that change to publish it.

# RTLPlayground — Homelab Handoff

Working notes for continuing this work in a **local** Claude Code session (the
previous sessions ran in a remote container with no access to the local machine
or its MCP servers).

Last updated: 2026-09-09. Repo state at time of writing: branch
`claude/implement-readme-features-Z6vLB` == `upstream/main` @ `3f65db9`.

---

## 1. What this is

[RTLPlayground](https://github.com/logicog/RTLPlayground) is open replacement
firmware for cheap RTL8372/RTL8373-based 2.5GbE switches. The goal here is to
run it on two XikeStor switches instead of their (buggy) stock firmware.

The firmware is 8051 code — tiny, banked, with ~128 bytes of internal RAM and a
single-connection uIP TCP stack. Expect constraints that feel absurd by modern
standards; most of the gotchas below come from that.

## 2. The two switches

### Switch A — 4×2.5G + 2×SFP+ (in service, do not disturb casually)

| | |
|---|---|
| Machine target | `MACHINE_KP_9000_6XHML_X2` |
| Management IP | `10.0.10.50` |
| Firmware | **Old pre-upstream build** — see §6 |
| Ports | 1–4 RJ45, 5/6 SFP+ (screen numbering; case silkscreen for the two SFP cages is swapped relative to the UI) |

Currently configured: SFP uplink trunk to the core switch tagged for VLANs
10/20/30/35/60, one RJ45 trunk, two RJ45 access ports (VLAN 30 / VLAN 60),
management VLAN 10.

It works, but it runs firmware that predates the upstream rebase, so its
management-VLAN behaviour rests on a local CPU-PVID hack rather than upstream's
proper implementation. Reflashing it to an upstream build is a sensible future
task — but that means redoing its VLAN config.

### Switch B — 8×2.5G + 1×SFP+ (freshly flashed and configured)

| | |
|---|---|
| Model on label | `SKS3200M-8GPY1XF` (upstream documents the same hardware as `SKS3200-8E1X`) |
| PCB silkscreen | `PCB-SWTG118AS-V2.1-17462` |
| Flash chip | Winbond `25Q16JVSIQ`, **2 MB**, SOIC-8, ref `U5` |
| SoC | RTL8373 + RTL8224 PHY (under heatsinks, not visually confirmed — irrelevant, see §5) |
| Machine target | **`MACHINE_SWTGW218AS`** |
| Management IP | `10.0.10.51` |
| Stock firmware defaults (pre-flash) | `192.168.10.12`, admin/admin |

Upstream documents this exact board in `doc/devices/SWTGW218AS.md`, including
the flash part number — so the machine target is confirmed, not guessed.

**Port numbering is identity-mapped**: case label N == UI port N == CLI port N.
The SFP+ cage is labelled `9 (10G)` on the front panel and is CLI port `9`.

## 3. Network context (the homelab it plugs into)

- **Core switch: Brocade ICX6610** — does *all* L3 routing; every VLAN's SVI lives there.
- **VLAN 10 = `10.0.10.0/24`**, gateway `10.0.10.1` — management network.
- VLANs in use: **10, 20, 30, 35, 60**.
- **DHCP: Technitium DNS server at `10.0.60.4`**, reached via UDP helper configured per-VLAN on the Brocade. Works fine; don't debug DHCP here.
- Default route on the Brocade → OPNsense, **internet only**. Inter-VLAN traffic never touches OPNsense.
- Uplinks from both playground switches are **tagged trunks with no native/untagged VLAN** on the Brocade side.

## 4. Repo / branch layout

Fork: `osmanates/RTLPlayground` (remote `origin`)
Upstream: `logicog/RTLPlayground` (add as remote `upstream`)

```bash
git remote add upstream https://github.com/logicog/RTLPlayground
git fetch upstream main
```

| Ref | Commit | What |
|---|---|---|
| `claude/implement-readme-features-Z6vLB` | `3f65db9` | Working branch. **Byte-identical to `upstream/main`.** |
| `archive/pre-upstream-work` | `55049dd` | Old custom work before the rebase. See §6. |
| `firmware-artifact-20260410` | `9d440f3` | Orphan branch used only to ferry `.bin` files out of the remote container. **Delete it** — a local session can just build. |

> **Do not open PRs against `logicog/RTLPlayground`.** All work stays in the fork.

Retrieve the built images if wanted (otherwise just rebuild):
```bash
git show firmware-artifact-20260410:rtlplayground-3f65db9-SWTGW218AS-oem-upgrade.bin > oem.bin
git push origin --delete firmware-artifact-20260410
```

## 5. Building

### Toolchain — sdcc **4.5** is mandatory

Ubuntu 24.04 ships sdcc 4.2, which **fails to compile this codebase**. Get 4.5:

```bash
curl -LO https://downloads.sourceforge.net/project/sdcc/sdcc-linux-amd64/4.5.0/sdcc-4.5.0-amd64-unknown-linux2.5.tar.bz2
tar xjf sdcc-4.5.0-amd64-unknown-linux2.5.tar.bz2
export PATH=$PWD/sdcc-4.5.0/bin:$PATH
sdcc --version   # must say 4.5.0
```

Other deps: `make gcc xxd python-is-python3 libjson-c-dev`
(upstream also ships a `Dockerfile` if you'd rather not install sdcc — see its README §0).

### Build

```bash
make MACHINE=SWTGW218AS          # -> output/SWTGW218AS/rtlplayground-v0.1.0-<sha>-SWTGW218AS.bin
cd installer && make             # -> installer/output/rtlplayground_oem_upgrade.bin
```

`make machine_check` compiles `machine.c` for every machine target — but it uses
`set -o pipefail`, so it needs bash; under dash it errors out. Run the loop
manually if `/bin/sh` isn't bash.

### The two image types — do not mix them up

| File | Size | Upload when |
|---|---|---|
| `output/<MACHINE>/rtlplayground-*.bin` | 524288 | Device already runs **RTLPlayground** (or SOIC clip) |
| `installer/output/rtlplayground_oem_upgrade.bin` | 540710 | Device still runs **stock/OEM** firmware |

Uploading the wrong one gets rejected. A valid raw flash image starts with magic
bytes `00 40` (`xxd file | head -1`).

## 6. What happened to the old custom work

Five commits existed before the rebase; four are dead. Kept on
`archive/pre-upstream-work` (`55049dd`) purely for reference.

| Old commit | Fate |
|---|---|
| SDCC 4.5 build fixes | **Superseded** — upstream builds clean |
| CPU-PVID / management VLAN hack | **Superseded, and upstream's is better** — it inserts a real 802.1Q tag on CPU-originated frames plus a static L2 entry for the switch's own MAC |
| STP + RLDP + "Services" web tab | **Superseded** — upstream has full STP (930 lines) with its own `stp.html`, plus i18n |
| Browser polling removal | Still unique, probably unnecessary now (upstream's httpd is far more robust) |
| Custom XikeStor machine definition | **Redundant** — upstream's `MACHINE_SWTGW218AS` covers this board |
| DHCP **server** (`dhcpd`) | Only genuinely unique piece. **Never tested on hardware.** Porting it into the much-changed upstream tree is real work for a feature that was never used. |

Upstream also fixed **the bug that cost the most time**: multipart POST bodies
split across TCP segments, which made `/config` and `/upload` return 400. That's
why the web-based config save and firmware update now work.

## 7. Gotchas — read before touching a live switch

**Lockout risk — management VLAN.** `vlan <id> mgmt` makes the CPU tag all its
outgoing frames with that VLAN. Any port that isn't a member of it instantly
stops reaching management. This caused a multi-hour lockout once. Always keep
one **untagged access port on the management VLAN** as a lifeline (on switch B
that's port 1) before enabling it.

**Recovery ladder**, cheapest first:
1. Reset button **>10 s** → restores default config → back on `192.168.10.247`.
   (3–10 s = plain reboot, keeps saved config.) Wired to `GPIO54` on
   `MACHINE_SWTGW218AS`, so it should work on switch B. It was never confirmed
   working on switch A.
2. Power-cycle — only helps if the bad config was never saved to flash.
3. **SOIC-8 clip on `U5`** — the real backstop. Always take a full 2 MB backup
   before flashing anything. *Some pads on switch A are damaged; switch B is
   intact.*

**uIP has exactly one TCP connection** (`UIP_CONF_MAX_CONNECTIONS 1`). The web
UI is slow and will time out if hammered. Keep one browser tab, close DevTools'
Network panel, pause a few seconds between actions. `ERR_CONNECTION_TIMED_OUT`
usually means "too many requests in flight", not "broken".

**Web session times out after 200 s** and is only refreshed by *static file*
loads, not AJAX. Sitting on one page thinking will silently log you out.

**Diagnosing "is the switch really answering?"** — ping TTL is the tell.
uIP replies with **TTL 64**. A **TTL 128** reply is Windows answering itself:
some local adapter (Hyper-V / WSL / VPN) holds that IP. This wasted a lot of
time once. Also disable Wi-Fi when testing, or traffic silently routes around
the switch.

**CPU port is auto-added as a tagged member of every VLAN you create**
(`vlan_create()` in `rtl837x_port.c`). Consequences: management-over-trunk works
without extra config, *but* every VLAN you create floods its broadcast traffic
to a slow 8051. Don't create VLANs you aren't using.

**`vlan_create` overwrites the whole membership list.** To remove a port from a
VLAN, re-state the VLAN listing only the ports that should remain. There is no
"remove port" command.

**Ports stay in default VLAN 1** even after being added to another VLAN, and
there's no way to remove them via CLI. Inert as long as every PVID is
reassigned, but worth knowing.

**Flash layout** (512 KiB image on a 2 MB chip):

| Offset | Contents |
|---|---|
| `0x00000`–`0x3FFFF` | code |
| `0x40000`–`0x6EFFF` | web UI files |
| `0x6F000` | default config (baked from `config.txt` at build time) |
| `0x70000` | **runtime/startup config** — a full flash write resets this |
| `0x80000` | firmware upload staging (needs ≥1 MB flash) |
| `0x1FC000` | factory MAC — read via `mac_flash_offset`, needs a 2 MB part |

**SoC type in the machine struct is cosmetic.** `machine.isRTL8373` only drives
an `INCORRECT MACHINE!` console print; everything real uses
`machine_detected.isRTL8373`, read at boot from `RTL837X_REG_CHIP_ID`. No need
to pull heatsinks to identify the chip.

**No serial console available** on these units (would need soldering). All
config goes through the web UI or its `/cmd` endpoint.

## 8. CLI syntax (verified against `cmd_parser.c` @ `3f65db9`)

Commands: `bw dhcp eee flash forget gpio gw history hostname igmp ingress ip
isolate l2 lag laghash mirror mtu netmask off on passwd phyget physet port pvid
regget regset reset rnd sds sdsget sdsset sfp show stat stp syslog time version
vlan`

```
vlan <id> [name] <port>[t] ...   # 't' = tagged; no suffix = untagged
vlan <id> mgmt                   # set management VLAN (0 disables)
vlan <id> d                      # delete
vlan show
pvid <port> <1-4094>
ip <addr> | ip dhcp | ip         # bare 'ip' prints current
gw <addr> / netmask <addr>
hostname <name>
```

Ports are a **single digit**, mapped through `machine.phys_to_log_port`. On
switch B that's identity, so `9` is the SFP+.

Upstream added a **Console tab** under System in the web UI — no more DevTools
`fetch()` gymnastics for one-off commands.

## 9. Switch B's current startup config

Saved via System → Advanced → *Startup configuration* → Save Startup Settings to
Flash.

```
hostname sks3200m
vlan 10 1 9t
vlan 30 2 3 4 5 6 7 8 9t
pvid 1 10
pvid 2 30
pvid 3 30
pvid 4 30
pvid 5 30
pvid 6 30
pvid 7 30
pvid 8 30
pvid 9 30
netmask 255.255.255.0
ip 10.0.10.51
gw 10.0.10.1
vlan 10 mgmt
```

Resulting port roles:

| Port | Role | Membership | PVID |
|---|---|---|---|
| 1 | Management access (**lifeline**) | 10 untagged | 10 |
| 2–8 | Device access (TV, PS5, Chromecast…) | 30 untagged | 30 |
| 9 (SFP+) | Trunk to Brocade | 10 tagged, 30 tagged | 30 |

Devices send untagged → PVID stamps VLAN 30 → egresses port 9 tagged 30 →
Brocade routes. Return traffic gets untagged on the way back out. Devices never
see a VLAN tag.

To move e.g. port 6 to VLAN 20 later:
```
vlan 20 6 9t
pvid 6 20
vlan 30 2 3 4 5 7 8 9t     # re-state VLAN 30 without port 6
```
…and add VLAN 20 to the Brocade trunk.

## 10. Open items / possible next tasks

- **Verify switch B's SFP+ detects a module.** `MACHINE_SWTGW218AS` uses
  `pin_detect = GPIO30`, `pin_los = GPIO37`, characterised on the **V2.0** PCB;
  switch B is **V2.1**. Upstream's other V2.1 board moved to `pin_detect =
  GPIO38` with no LOS wired. If detection fails, that's the one-line fix. All
  other functionality is independent of it.
- **Check switch B's MAC** on the Overview page. Should be the factory MAC from
  the label; a synthesised `06:xx:xx:...` means the `mac_flash_offset` read
  didn't land.
- **Reflash switch A** onto an upstream build (`MACHINE_KP_9000_6XHML_X2`) to
  get the proper management-VLAN implementation and the working web upload.
  Means redoing its VLAN config — plan a maintenance window.
- **Consider contributing back**: upstream's `doc/devices/SWTGW218AS.md` has an
  empty "Chip RTL" column for the XikeStor row, and `doc/supported_devices.md`
  lists the model as `SKS3200-8E1X` while the physical label reads
  `SKS3200M-8GPY1XF`. Both are small, useful PRs — but ask before opening one.
- Decide whether the archived DHCP server is worth porting. Probably not.

## 11. Starting the local session

```bash
git clone https://github.com/osmanates/RTLPlayground
cd RTLPlayground
git checkout claude/implement-readme-features-Z6vLB
# then: "read HANDOFF.md"
```

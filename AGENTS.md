# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this project is

**Router Emulator**: a QEMU machine that emulates MediaTek router boards at
the hardware level, running unmodified firmware through the real boot chain
(BootROM → BL2 → BL31/TF-A → U-Boot → OpenWrt). The emulator is a set of
QEMU patches plus the launchers and tools around them — there is no firmware
fork.

- Machines: `mt7981-router`, `mt7986-router`, `mt7987-router`,
  `mt7988-router` (all in `qemu-system-aarch64`) and `mt7621-router`
  (MIPS, in `qemu-system-mipsel`).
- Base: QEMU **v10.1.0** + `qemu-patches/*.patch`.
- Boards are data, not code: `presets/*.ini` describe concrete routers.
- User-facing docs: [README.md](README.md) (English) and
  [README.ru.md](README.ru.md) (Russian); build docs under
  `README.build.*.md`.

Read [README.md](README.md) and, for QEMU work,
[README.build.linux.md](README.build.linux.md) before changing behavior.

## Ground rules

1. **Never patch or modify firmware.** Firmware (OpenWrt, vendor images,
  bootloaders) runs exactly as shipped. If firmware fails, adapt the QEMU
  device model. Vendor differences belong in board presets, not in
  firmware-specific hacks in the machine.
2. **Never commit, publish, print, or upload real user data**: MAC
  addresses, serial numbers, PINs, admin passwords, Wi-Fi keys, eFuse/UID
  values, or raw dumps of a user's router. `dumps/`, `factory/`, `usb/`
  contents, `firmware/`, vendor bootloader folders and any `nand-*`,
  `nor-*`, `emmc-*` folder are gitignored for this reason. When a flash
  folder must be shipped, every occurrence of the real MAC is replaced
  with the standard `80:af:ca:11:22:33` (see
  `tools/prepare-nand.sh`); Wi-Fi calibration data may be kept only when
  the user explicitly allows it.
3. **Do not push, tag, release, or upload to cloud storage unless the user
  explicitly asks.** Local commits are fine; publishing is not.
4. **Never kill processes by pattern** (`pkill -f …`, `pkill <name>`): the
  pattern matches the calling shell. Keep the PID (`cmd & P=$!`,
  `wait $P`) or use `killall` with an exact process name.
5. **Keep the English and Russian docs in sync.** Changes to
  `README.md`/`README.build.*.md` need the same change in the `.ru.md`
  counterparts; launcher texts live in `languages/en.ini` and
  `languages/ru.ini`. Chat, code comments and commit messages are English.
6. **No compatibility shims for renamed preset/machine keys.** Since 4.1,
  `nand-dir`/`flash=`/`nand=`/`nor=`/`nand-id`/`nor-id`/`nor-nand` were
  replaced by `dump-dir` + `flash-type`/`flash-id`/`flash-mb`/`flash-uid`
  and `flash2-*`; the old names are rejected on purpose. Document
  migrations in the README instead of adding aliases.
7. **Prefer one table over special cases.** Chip type/size data lives in
  `src/qemu/hw/arm/filogic/flash_chips.h`; the flash type is never inferred
  from the chip ID. Add new hardware there (and to the Windows editor list),
  not as one-off branches.
8. **Do not make users supply files they do not have.** A "dump" is normally
  per-partition backups taken under OpenWrt (`*.mtdN.*.bin`). The emulator
  must synthesize what those backups lack (e.g. the MT7621 factory BBT).
   Optional raw-with-OOB inputs stay a documented extra, never a
  requirement.
9. **Keep tests fast.** Abort or avoid anything that runs longer than about
   six minutes; targeted `tests/quick.py` console checks are preferred over
   full boot-and-wait runs.
10. **Do not delete user data.** Deleting build output, temp copies and
    scratch files is routine; never delete `dumps/`, `factory/`, vendor
    dumps, or an uncommitted flash folder without a concrete reason.
11. **Every new board preset gets its LEDs**, generated from the OpenWrt
   device tree with `tools/dts-leds.py` (see below). This is not optional.
12. **Project naming**: the product is "Router Emulator", launchers are
    `emulator.sh` / `emulator.exe`. SoC names belong to machine names and
    `hw/` paths; concrete router models belong only in `presets/`. Do not
    reintroduce a vendor name into the product name or generic docs.

## Repository layout

```
build.sh                    Linux QEMU build (QEMU v10.1.0 + qemu-patches/)
build-windows.sh            cross-build of the Windows package (Docker + mono)
emulator.sh                 Linux launcher (reads presets, builds QEMU args)
qemu-patches/               the QEMU patch series (numbered; the source of truth)
presets/                    board presets, one INI per router
tools/                      prepare-nand.sh, mknand.py, mkemmc.py,
                            dts-leds.py, ib-flash-size.py, host-bridge.sh,
                            pgo-train.py, package-*.sh, gen-lang-en.py,
                            uboot-rutc50/, uboot-mi-router-4a-gigabit-v2/
tests/                      quick.py, powercut.py, cpubench.sh
windows/                    C# launcher, preset editor, terminal, LEDs, USB
languages/                  launcher translations (en.ini, ru.ini)
.github/workflows/build.yml  CI: build, flash images, boot tests, releases
VERSION                     emulator version (release trigger)
src/                        local QEMU/OpenWrt/U-Boot/TF-A trees (gitignored)
work/, logs/, dist/         scratch, console logs, packages (gitignored)
```

`src/qemu` is a **local, gitignored** QEMU clone on branch `mt7981` whose
commits are exported as `qemu-patches/*.patch`. CI and users never see
`src/`; they clone QEMU and apply the patch series. Anything not exported
to `qemu-patches/` effectively does not exist.

## Build

Prerequisites are in [README.build.linux.md](README.build.linux.md)
(compiler, git, meson, ninja, pkg-config, Python 3 venv, glib/pixman/zlib
development files; libslirp/libfdt are downloaded if missing).

```bash
./build.sh                  # clone/update QEMU, apply patches, build both binaries
JOBS=8 ./build.sh           # limit parallelism
CONFIGURE_ARGS="--disable-gtk --disable-sdl --disable-opengl --disable-vnc \
  --disable-spice --disable-curl --audio-drv-list=" ./build.sh   # small CI-like build
PGO_NAND=nand-wr3000p ./build.sh   # profile-guided optimisation (releases; needs GCC)
```

Outputs:

- `src/qemu/build/qemu-system-aarch64` (Filogic machines)
- `src/qemu/build/qemu-system-mipsel` (MT7621)

Sanity check after building:

```bash
src/qemu/build/qemu-system-aarch64 -M help | grep -E 'mt798[1678]-router'
src/qemu/build/qemu-system-aarch64 -M mt7981-router,help
src/qemu/build/qemu-system-mipsel -M help | grep mt7621-router
```

After editing QEMU sources, incremental rebuild:

```bash
ninja -C src/qemu/build qemu-system-aarch64 qemu-system-mipsel
```

The Windows package is built on Linux (Docker image with the MinGW
toolchain, clang, mono-mcs; PGO under Wine by default):

```bash
./build-windows.sh          # dist/Router-Emulator-<version>-win64.zip, -dumps.zip
PGO=0 ./build-windows.sh    # skip PGO
FLASH_PRESETS="cudy-wr3000p-v1" ./build-windows.sh   # fewer flash folders
```

## Run

```bash
./emulator.sh -P list                 # available board presets
./emulator.sh -P cudy-wr3000p-v1      # default preset, console in this terminal
./emulator.sh -P cudy-wr3000p-v1 -o "usb-port=3,ddr=ddr3"
```

Useful options (full list in `emulator.sh -h` and README):

- `-n DIR` flash folder (default: the preset's `dump-dir`)
- `-w bridge|user|offline|none` WAN; `offline` = user-mode with
  `restrict=on` (DHCP works, nothing leaves the PC)
- `-l isolated|nic|user|none` LAN; never put the router LAN on the physical
  network by default (`nic` leaks DHCP/RA onto the real LAN)
- `-S SOCK -L DIR` headless console on a Unix socket, plus console logs
- `-g` log GPIO/LED changes, `-d` log unimplemented register accesses
- `-u DIR` export a folder as a FAT16 USB stick, `-u none` disable
- `-R` boot with reset held 10 s (U-Boot TFTP recovery)

Headless run (no root):

```bash
./emulator.sh -P cudy-wr3000p-v1 -w offline -l user -S work/console.sock &
socat -,raw,echo=0,escape=0x1d UNIX-CONNECT:work/console.sock   # Ctrl-] detaches
echo quit | socat - UNIX-CONNECT:work/monitor.sock               # stop QEMU
```

LuCI is then at `http://127.0.0.1:8080/` (preset `lan-ip`/`lan-forwards`).
Interactive exit from the serial console is `Ctrl-A X`.

## Windows launcher

- Sources: `windows/Launcher.cs` (window, run, log), `Presets.cs` (preset
  parsing and editor, per-SoC option filters), `Terminal.cs`, `Leds.cs`,
  `UsbDevices.cs`, `Lang.cs`; UI texts in `languages/*.ini`.
- Compiled by `build-windows.sh` with mono-mcs against the .NET Framework
  4.8 reference assemblies; the title-bar/file version comes from `VERSION`.
- `windows/Presets.cs` filters machine options per SoC (e.g. `Mt7621Keys`
  for `soc=mt7621`); an option missing from those lists is silently dropped
  by the Windows build even though Linux honours it.
- Editor quirks: show/hide tabs only after the control handle exists
  (Windows ignores `TabPages.Insert` before that, mono does not); for
  screenshots run it under mono + xvfb and use `import -window root`,
  because `DrawToBitmap` renders the wrong tab.

## Flash folders

A flash folder holds per-partition dumps **without OOB**. Every file whose
name contains `mtdN` becomes partition N, concatenated in N order
(`mt7981.mtd0.BL2.bin`, … `mt7981.mtd5.ubi.bin`). The emulator writes
changes back into these files, so treat a folder used for testing as
disposable.

```bash
# official OpenWrt images -> nand-wr3000p/
tools/prepare-nand.sh cudy_wr3000p-v1 25.12.5
tools/prepare-nand.sh cudy_wr3000p-v1 snapshot

# vendor bootloader kept (BL2/FIP dumps in DIR), OpenWrt sysupgrade in that layout
tools/prepare-nand.sh --stock DIR --flash-mb 256 cudy_wr3000u-v1 25.12.5

# own OpenWrt build (the *-ubootmod-* images in DIR)
tools/prepare-nand.sh --local DIR cudy_m3000-v1 25.12.5

# layout variants
tools/prepare-nand.sh --no-bdinfo netis_nx31 25.12.5
tools/prepare-nand.sh --nor --stock DIR cudy_wr3000-v1 25.12.5
tools/prepare-nand.sh --soc mt7621 --stock DIR xiaomi_mi-router-3g 25.12.5
tools/prepare-nand.sh --emmc glinet_gl-mt6000 25.12.5
```

- `PROFILE` is the OpenWrt device profile; the output folder defaults to
  `nand-NAME` (`nor-NAME`, `emmc-NAME`) with `NAME` = profile without the
  vendor prefix and `-v1` suffix. The preset's `dump-dir` must match it.
- Flash chips come from the preset via `--flash-type`, `--flash-id`,
  `--flash-mb`, `--flash2-id`, `--flash2-mb`; sizes are read from
  `flash_chips.h` (or the corresponding patch when `src/qemu` is absent).
- `factory/` supplies `*Factory*.bin` (Wi-Fi calibration) and
  `*bdinfo*.bin` (MAC); without them defaults/random data are used.
- `tools/mknand.py`, `tools/mkemmc.py` create/edit images directly;
  `tools/ib-flash-size.py` builds an OpenWrt image for a larger SPI-NOR.

## Tests

The workhorse is `tests/quick.py`: it boots QEMU, puts the console on a
socket, matches patterns in order, and fails on timeout. Exit code 0 means
all steps matched.

```bash
python3 tests/quick.py -P cudy-wr3000p-v1 --limit 600 \
  --qemu="-netdev user,id=wan,restrict=on" \
  'BL2: @60' 'U-Boot 20@120' 'Starting kernel@240' \
  'Linux version@60' 'Please press Enter@480'
```

Step syntax: `PATTERN` (regex, 120 s), `PATTERN@SEC` (own timeout),
`>TEXT` (send TEXT + CR), `!SEC` (sleep). Other flags: `-n` flash folder,
`-o k=v,...` override machine options, `-M` full machine option string,
`--win` run the Windows build under Wine, `--log` output log.

Other checks:

```bash
python3 tests/powercut.py nand-wr3000p 'gmac0=mt7531,ports=lan4:lan3:lan2:lan1:-,gmac1=rtl8221b,ddr=ddr3' 6
tests/cpubench.sh [QEMU_BINARY] [PRESET]     # guest loop/md5 timings
```

CI (`.github/workflows/build.yml`) does what a change must survive:
builds `linux`, `linux-aarch64` and `win64`; builds flash folders for
`TEST_PRESETS` from the presets; boots every test preset from the Linux
package and from the Windows package on real Windows, checking the first
boot stage, `U-Boot 20`, console and the WAN link. Presets added to
`TEST_PRESETS` also need an entry in the `test-linux` and `windows-test`
matrices with the WAN interface name that board's kernel prints.

## Changing QEMU device models

Device models live in the local QEMU tree (after the patches are applied):

- `src/qemu/hw/arm/filogic/` — `filogic_router.c` (machine, BootROM),
  `mtk_sysctl.c`, `mtk_eth.c`, `spinand.c`, `spinor.c`, `mtk_spim.c`,
  `mtk_msdc.c`, `mtk_pcie.c`, `mtk_wmac.c`, `mt7992_wifi.c`,
  `mtk_pinctrl.c`, `filogic_leds.c`, `ws2812b.c`, `hc595.c`,
  `flash_chips.h`
- `src/qemu/hw/mips/mt7621*.c` — MT7621 machine, NFI NAND, PCIe
- `src/qemu/target/mips/`, `src/qemu/hw/intc/mips_gic.c` — MT7621 CPU/GIC
- `src/qemu/net/pcap.c`, `src/qemu/chardev/` — launcher-facing extras

Workflow:

```bash
# 1. edit, build, test
ninja -C src/qemu/build qemu-system-aarch64 qemu-system-mipsel

# 2. commit in the QEMU tree (branch mt7981, based on v10.1.0)
cd src/qemu
git add -A && git commit -m "hw/arm/filogic: ..."

# 3. export the patch series (this is what CI/users consume)
git format-patch --no-numbered --start-number N -o ../../qemu-patches -1
# N = one more than the last patch number in qemu-patches/ (0064 today);
# or regenerate the whole series:
git format-patch --no-numbered -o ../../qemu-patches v10.1.0..mt7981
```

- New patches continue the numbering; never renumber an existing patch
  without regenerating the whole series.
- `build.sh` applies only the patches that are missing from an existing
  checkout (`git am` on the remainder), so a rewritten existing patch needs
  a fresh branch/reset to take effect in a clean clone.
- The patch files are the deliverable; a local `src/qemu` commit that is not
  exported will be lost for CI, packages and users.
- To find what new firmware polls, run with `-d` (`-d unimp` into
  `work/qemu.log`).
- New machine options must be documented in the README option table and
  added to the Windows launcher lists in `windows/Presets.cs`; MT7621
  options additionally need to be in `Mt7621Keys`, or the Windows build
  silently drops them.
- Keep firmware untouched: workaround in the device model, not in the guest.
- Avoid `-O3`/LTO experiments without measurements; the project already
  found TCG translation-block lookup dominates, and the Linux/Windows
  packages are PGO-optimised.

## Adding or changing a board preset

1. Copy the closest preset in `presets/`, rename it, and set `name=`,
   `description=`, `soc=` (omit for MT7981), `ram=`, `dump-dir=` and the
   machine options (Ethernet/switch/PHY, `flash-type`/`flash-id`/…, DDR,
   USB, buttons, LEDs). Build-only keys `openwrt=`, `openwrt-no-bdinfo=`,
   `openwrt-parts=`, `openwrt-emmc=`, `openwrt-uboot=`, `openwrt-ubi-fip=`,
   `openwrt-nor-nand=` are consumed by `prepare-nand.sh`/CI, not by QEMU;
   launcher-only keys are `lan-ip=` and `lan-forwards=`.
2. Generate the LED lines from the OpenWrt DTS (never write them by hand if
   the DTS is available):

   ```bash
   python3 tools/dts-leds.py --openwrt src/openwrt PRESET      # print
   python3 tools/dts-leds.py --write --all                     # update presets
   ```

3. Build the flash folder with `tools/prepare-nand.sh` using the same
   `--flash-type/--flash-id` as the preset, then boot-test it:

   ```bash
   python3 tests/quick.py -P PRESET --limit 600 \
     --qemu="-netdev user,id=wan,restrict=on" \
     'BL2: @120' 'U-Boot 20@240' 'Please enter@600'
   ```

4. Update the board table in `README.md` **and** `README.ru.md`.
5. Add the preset to `TEST_PRESETS` and the test matrices if official
   OpenWrt images exist for it. Do not add CI jobs that build custom
   U-Boot or modified OpenWrt images — those are built in the separate
   `stavultras/openwrt-patched-builds` repository.
6. A change that touches only `presets/` is a presets-only release; see
   below.

Notes:

- The Windows preset editor drops `;` comments when saving. Keep presets
  readable anyway; comments are useful in git.
- A preset's `dump-dir` is relative to the repository root and must match
  the folder `prepare-nand.sh` produced.
- Flash key names changed in 4.1 with no aliases; if you add/rename a key,
  update the README migration table, `windows/Presets.cs`, and any
  generators (`tools/dts-leds.py`, `tools/prepare-nand.sh`).

## Documentation and translations

- English is the source of truth for docs; Russian files mirror them:
  `README.md` ↔ `README.ru.md`, `README.build.linux.md` ↔
  `README.build.linux.ru.md`, same for Windows, and
  `languages/en.ini` ↔ `languages/ru.ini`.
- After changing launcher UI strings: `tools/gen-lang-en.py` regenerates
  `en.ini`; `tools/gen-lang-en.py --check` lists keys missing from the
  other language files.
- Keep the machine-option table, preset table and flash-option migration
  table in sync with the code.
- Markdown-only pushes do not trigger CI (`paths-ignore: ['**.md']`).

## Releases

The release kind is encoded in `VERSION`:

- **`X.Y`** — binary release: packages plus the presets archive. Use it
  whenever anything outside `presets/` changes (QEMU patches, launchers,
  tools, docs that ship).
- **`X.Y.Z`** — presets-only release: only `presets/` changed; CI builds no
  packages and runs no boot tests, and publishes only
  `Router-Emulator-<version>-presets.zip`, to be unpacked over the `X.Y`
  packages.

To release: bump `VERSION`, commit, and push to `main` (only on an explicit
request). When tag `v<VERSION>` does not exist, CI creates it and the
GitHub Release. For a binary (`X.Y`) release it first builds with PGO and
runs the boot tests; a presets-only (`X.Y.Z`) release skips those and
publishes only the presets archive. A binary release contains:

- `Router-Emulator-<version>-linux-x86_64.tar.gz`
- `Router-Emulator-<version>-linux-aarch64.tar.gz`
- `Router-Emulator-<version>-win64.zip`
- `Router-Emulator-<version>-presets.zip`

No flash folders and no presets are inside the packages. Release notes are
prepared in `work/notes-<version>.md` and inserted into the GitHub release
body before `**Full Changelog**`. Locally, `build-windows.sh` also produces
`Router-Emulator-<version>-dumps.zip` containing only the flash folders.
Release commit messages are in English.

## Conventions and pitfalls

- **Flash folders are mutable.** The guest writes into them (settings,
  sysupgrade, U-Boot environment). Copy a folder before destructive tests;
  never point a test at a folder you cannot recreate.
- **Default LAN must stay isolated.** Bridging the router LAN to `br0`
  exposes its DHCP and IPv6 RA on the real network; `-l isolated` is the
  default for a reason. Connecting several LAN ports to the same host
  bridge creates a loop.
- **MT7621 is a different binary** (`qemu-system-mipsel`) and a different
  code path (`hw/mips/`); the Filogic launcher logic still applies. Its
  presets use `soc=mt7621`.
- **SPI-NOR vs SPI-NAND vs eMMC is never guessed from the chip ID**;
  `flash-type` decides. Chips must be added to `flash_chips.h` (and to the
  Windows editor's list) to get a size automatically.
- **`src/qemu` is not in git.** `git status` in the repository root will not
  show QEMU edits; check `src/qemu` and export patches.
- **`tests/quick.py` picks `src/qemu/build/...`, then `qemu/...`** (release
  layout); set `QEMU_BIN` to test another binary, `--win`/`QEXE` for the
  Windows build under Wine.
- **PGO is sticky.** `PGO_NAND=… ./build.sh` leaves the profile in
  `src/qemu/pgo-prof` and later plain builds reuse it; `rm -rf
  src/qemu/build` (and the PGO directories) returns to a plain build.
- **Do not reboot OpenWrt too early in tests.** It marks the overlay
  "ready" only at the end of boot; a reboot before that makes fstools wipe
  the overlay (the same happens on hardware).
- **Do not run bare `git clean`/`rm -rf` over the root**: gitignored folders
  contain multi-GB flash folders and user dumps.
- **Long builds/tests**: use background jobs and wait on their PID or log;
  do not poll with `pgrep -f`.
- Shell style: `#!/bin/bash` + `set -e` for scripts, `cd` to the repo root
  via `readlink -f`, quote paths, and make comments explain *why*.
- Python tools use the standard library only, `argparse`, and a module
  docstring with usage examples.
- C changes follow QEMU style (checkpatch); keep diffs minimal and focused.
- Commit messages: short area prefix, imperative, English
  (e.g. `hw/arm/filogic: …`, `hw/mips/mt7621: …`, `CI: …`,
  `Presets: …`, `README: …`).

## Where to look first

| Task | Start at |
|---|---|
| Board boots / DDR / boot chain | `src/qemu/hw/arm/filogic/filogic_router.c`, `mtk_sysctl.c` |
| Ethernet, switch, PHYs | `src/qemu/hw/arm/filogic/mtk_eth.c` |
| NAND / NOR / eMMC | `spinand.c`, `spinor.c`, `mtk_msdc.c`, `flash_chips.h`, `tools/mknand.py`, `tools/mkemmc.py` |
| Wi-Fi / WED / WO | `mtk_wmac.c`, `mt7992_wifi.c`, `mtk_wo.c` |
| LEDs / buttons | `mtk_pinctrl.c`, `filogic_leds.c`, `ws2812b.c`, `hc595.c`, `tools/dts-leds.py` |
| Board definition | `presets/*.ini`, README option table |
| Flash image creation | `tools/prepare-nand.sh`, `tools/mknand.py`, `tools/mkemmc.py` |
| Launcher (Linux) | `emulator.sh` |
| Launcher (Windows) | `windows/*.cs`, `build-windows.sh` |
| CI / releases | `.github/workflows/build.yml`, `VERSION`, `tools/package-*.sh` |
| Boot-chain regression tests | `tests/quick.py`, `tests/powercut.py` |

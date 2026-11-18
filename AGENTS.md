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
  `README.build.*.md`; the Windows package ships `windows/README.txt`.

Read [README.md](README.md) and, for QEMU work,
[README.build.linux.md](README.build.linux.md) before changing behavior.

## First steps in a new session

```bash
git status --short                      # repo root: presets, tools, docs, CI
cat VERSION                             # release trigger; bump only on request
ls src/qemu/build/qemu-system-*         # built? if not: ./build.sh
git -C src/qemu rev-list --count v10.1.0..mt7981   # QEMU commits since base
ls qemu-patches/*.patch | wc -l         # must equal the count above
./emulator.sh -P list                   # the boards
```

- QEMU edits live in the **gitignored** `src/qemu`; the deliverable is
  `qemu-patches/`. A local QEMU commit that is not exported does not exist
  for CI, packages or users.
- Fast smoke test (no root, nothing leaves the PC; needs the preset's flash
  folder, built by `tools/prepare-nand.sh`):

  ```bash
  python3 tests/quick.py -P cudy-wr3000p-v1 --limit 600 \
    --qemu="-netdev user,id=wan,restrict=on" \
    'BL2: @60' 'U-Boot 20@120' 'Please press Enter@480'
  ```

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
   the user explicitly allows it. Vendor kernels may print the real MAC/UID
   on the console — redact logs before sharing them.
3. **Do not push, tag, release, or upload to cloud storage unless the user
   explicitly asks.** Local commits are fine; publishing is not.
4. **Never kill or wait on processes by pattern.** `pkill -f …` /
   `pkill <name>` matches the calling shell; an `until ! pgrep -f …` wait
   loop never ends because its own command line contains the pattern. Keep
   the PID (`cmd & P=$!`, `wait $P`), wait on a done-file the job writes, or
   use `killall` with an exact process name. Check for leftover loops before
   ending a long task.
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
   from the chip ID (IDs may collide or be user-modified). Add new hardware
   there (and to the Windows editor list), not as one-off branches.
8. **Do not make users supply files they do not have.** A "dump" is normally
   per-partition backups taken under OpenWrt (`*.mtdN.*.bin`). The emulator
   must synthesize what those backups lack (e.g. the MT7621 factory BBT).
   Optional raw-with-OOB inputs stay a documented extra, never a
   requirement.
9. **Keep tests fast.** Abort or avoid anything that runs longer than about
   six minutes; targeted `tests/quick.py` console checks are preferred over
   full boot-and-wait runs.
10. **Do not delete user data.** Deleting build output, temp copies and
    scratch files is routine and needs no confirmation; never delete
    `dumps/`, `factory/`, vendor dumps, or an uncommitted flash folder
    without a concrete reason. User-provided dumps often live in `~/NAME`
    or in gitignored folders at the repo root — copy them, never move.
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
                            dts-leds.py, kernel-dts.sh, ib-flash-size.py,
                            host-bridge.sh, pgo-train.py, pgo-windows.sh,
                            gen_dramc_table.py, package-*.sh, gen-lang-en.py,
                            uboot-rutc50/, uboot-mi-router-4a-gigabit-v2/
tests/                      quick.py, powercut.py, cpubench.sh
windows/                    C# launcher, preset editor, terminal, LEDs, USB;
                            README.txt (the package's user doc)
languages/                  launcher translations (en.ini, ru.ini)
.github/workflows/build.yml  CI: build, flash images, boot tests, releases
VERSION                     emulator version (release trigger)
src/                        local reference/build trees (gitignored): qemu,
                            openwrt, linux, u-boot, mtk-atf, mt76, coreboot
work/, logs/, dist/         scratch, console logs, packages (gitignored)
```

`src/qemu` is a **local, gitignored** QEMU clone on branch `mt7981` whose
commits are exported as `qemu-patches/*.patch`. CI and users never see
`src/`; they clone QEMU and apply the patch series. The other `src/` trees
are reference sources for modelling (OpenWrt's Linux, U-Boot, TF-A, mt76,
coreboot's MediaTek DRAMC code) and for `tools/dts-leds.py`.

`tools/uboot-rutc50/` and `tools/uboot-mi-router-4a-gigabit-v2/` hold the
U-Boot sources, defconfig, DTS and the **committed built binaries** for
boards that have no OpenWrt U-Boot (Teltonika RUTC50; Xiaomi 4A Gigabit v2
with 16/32/64 MB NOR). Their `build.sh` rebuilds locally; CI never builds
them ("users already have it"). Modified OpenWrt images (e.g. the 32/64 MB
4A builds from `tools/ib-flash-size.py`) are built in the separate
`stavultras/openwrt-patched-builds` repository, not here.

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

PGO: `PGO_NAND=DIR` builds an instrumented binary in `build-pgo-gen`, boots
the WR3000P flash folder `DIR` through `tools/pgo-train.py`, then rebuilds
`build/` with the profile (`src/qemu/pgo-prof`). **PGO is sticky**: later
plain builds reuse the profile; `rm -rf src/qemu/build` returns to a plain
build. `PGO_NAND` is not `PGO=1` (that is `build-windows.sh`'s own step).

The Windows package is built on Linux (Docker image with the MinGW
toolchain, clang, mono-mcs; PGO under Wine by default, `tools/pgo-windows.sh`):

```bash
./build-windows.sh          # dist/Router-Emulator-<version>-win64.zip, -dumps.zip
PGO=0 ./build-windows.sh    # skip PGO
FLASH_PRESETS="cudy-wr3000p-v1" ./build-windows.sh   # fewer flash folders
```

The unpacked package lands in `work/winpkg/Router-Emulator/` (what
`tests/quick.py --win` runs). clang is used deliberately: MinGW GCC's
emulated TLS made guest execution ~30 % slower.

## Run

```bash
./emulator.sh -P list                 # available board presets
./emulator.sh -P cudy-wr3000p-v1      # default preset, console in this terminal
./emulator.sh -P cudy-wr3000p-v1 -o "usb-port=3,ddr=ddr3"
```

`-P` accepts a preset file name, a path to an `.ini`, or the preset's
`name=` value. `QEMU_BIN` selects another binary (own build or a package).
Useful options (full list in `emulator.sh -h` and README):

- `-n DIR` flash folder (default: the preset's `dump-dir`)
- `-w bridge|user|offline|none` WAN; `offline` = user-mode with
  `restrict=on` (DHCP works, nothing leaves the PC). The default `bridge`
  creates taps via `tools/host-bridge.sh` and needs root/sudo once per host
  boot — agents should prefer `-w offline -l user` (no root, no leaks).
- `-l isolated|nic|user|none` LAN; never put the router LAN on the physical
  network by default (`nic` leaks DHCP/RA onto the real LAN)
- `-p PORTS` LAN ports to connect (default `1`; several ports on one host
  bridge create a loop)
- `-S SOCK -L DIR` headless console on a Unix socket, plus console logs
- `-m SOCK` QEMU monitor socket (default `work/monitor.sock`)
- `-g` log GPIO/LED changes, `-d` log unimplemented register accesses
- `-u DIR` export a folder as a FAT16 USB stick, `-u none` disable;
  `-o usb-host=VID:PID[;…]` passes real USB devices through
- `-R` boot with reset held 10 s (U-Boot TFTP recovery)
- `-o ram=…`, `-o lan-ip=…`, `-o lan-forwards=…` override preset values
- `WAN_EXTRA` / `LAN_EXTRA` (env) append to the user-mode netdev, e.g. a
  mock server: `WAN_EXTRA=",guestfwd=tcp:10.0.2.100:80-cmd:nc 127.0.0.1 18555"`

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
  (Windows ignores `TabPages.Insert` before that, mono does not); hide
  fields that the chosen SoC/devices do not have, rather than disabling
  them; for screenshots run it under mono + xvfb and use
  `import -window root`, because `DrawToBitmap` renders the wrong tab.
- `windows/README.txt` is the user doc shipped in the package; update it
  when launcher behavior changes.

## Flash folders

A flash folder holds per-partition dumps **without OOB**. Every file whose
name contains `mtdN` becomes partition N, concatenated in N order
(`mt7981.mtd0.BL2.bin`, … `mt7981.mtd5.ubi.bin`). The emulator writes
changes back into these files, so treat a folder used for testing as
disposable. Folders are named `nand-NAME` / `nor-NAME` / `emmc-NAME`.

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

### Vendor firmware and chip identity

Stock (MediaTek SDK) firmware checks chip identity, so the preset must
match the real board:

- Cudy stock compares `/proc/sys/dev/flash_uuid` ("C801-" + NAND UID) with
  the bdinfo `fuuid`; the user's own `flash-uid` is private (never
  committed). MT7621 NOR stock uses the NOR unique ID (16 hex digits, the
  part after "-" in `/proc/sys/dev/flash_uuid`); the machine reverses the
  bytes (the stock kernel prints them last-first).
- Cudy bdinfo is DES-encrypted with a known key (see the public
  `blocktrron/cudy-bdinfo-decrypt`); after editing it must be re-encrypted.
- Sanitizing a user's dump means overwriting **every** copy of the real
  MAC: factory `0xe000`/`0xe006`, the Wi-Fi EEPROM MAC fields
  (`0x4`/`0x8004`), and any occurrence in Bdata/Config (search the bytes,
  replace, do not print them). Wi-Fi calibration may ship only with the
  user's explicit permission.

## Tests

The workhorse is `tests/quick.py`: it boots QEMU, puts the console on a
socket, matches patterns in order, and fails on timeout. Exit code 0 means
all steps matched. Default hard limit 300 s; console log `work/quick.log`,
QEMU stderr `work/quick.log.err` — read both on failure.

```bash
python3 tests/quick.py -P cudy-wr3000p-v1 --limit 600 \
  --qemu="-netdev user,id=wan,restrict=on" \
  'BL2: @60' 'U-Boot 20@120' 'Starting kernel@240' \
  'Linux version@60' 'Please press Enter@480'
```

Step syntax: `PATTERN` (regex, 120 s), `PATTERN@SEC` (own timeout),
`>TEXT` (send TEXT + CR), `!SEC` (sleep). Other flags: `-n` flash folder,
`-o k=v,...` override machine options, `-M` full machine option string,
`--win` run the Windows build under Wine (`QEXE` overrides the exe,
default `work/winpkg/Router-Emulator/qemu/<binary>.exe`), `--log` output
log. `QEMU_BIN` selects another Linux binary.

Other checks:

```bash
python3 tests/powercut.py nand-wr3000p 'gmac0=mt7531,ports=lan4:lan3:lan2:lan1:-,gmac1=rtl8221b,ddr=ddr3' 6
tests/cpubench.sh [QEMU_BINARY] [PRESET]     # guest loop/md5 timings
```

CI (`.github/workflows/build.yml`) does what a change must survive:
builds `linux`, `linux-aarch64` and `win64`; the `flash` job builds flash
folders for `TEST_PRESETS` from the presets; `test-linux` and
`windows-test` boot every test preset from the Linux package and from the
Windows package on real Windows, checking the first boot stage,
`U-Boot 20`, console and the WAN link.

`TEST_PRESETS` is **one preset per distinct hardware combination** that
official OpenWrt images exist for — not every preset. Add a preset to
`TEST_PRESETS` and both matrices only when it brings a new combination
(SoC, DDR type, switch/PHY layout, flash type/layout, USB…). Matrix
entries carry `wan:` (the WAN link line that board's kernel prints) and
optionally `first:` (first boot stage, default `BL2: `) and `qemu:`
(binary; MT7621 uses `first: 'U-Boot SPL'`, `qemu: qemu-system-mipsel`).

## Debugging

- **Console logs**: every `emulator.sh` start writes
  `logs/console_YYYY-MM-DD_HH-MM-SS.log` (escape sequences stripped);
  `quick.py` writes `work/quick.log` + `.err`. Start here on any boot
  failure.
- **Unimplemented registers**: `-d` logs unimp/guest_errors to
  `work/qemu.log` — the fastest way to find what new firmware polls.
  `-g` prints GPIO/LED changes.
- **Monitor / buttons**: `socat - UNIX-CONNECT:work/monitor.sock`; press
  buttons with `qom-set /machine/pinctrl reset-button true` / `false`
  (also `wps-button`); `echo quit | socat - …` stops QEMU. `-R` holds
  reset 10 s at power-on (TFTP recovery).
- **Mock servers**: `WAN_EXTRA`/`LAN_EXTRA` guestfwd (see Run) lets a
  host-side script answer the router's cloud/update requests without
  touching the real network.
- **gdb on MT7621** (`qemu-system-mipsel -s -S`): `set endian little`,
  `set osabi none`, `set tdesc filename work/dbg/mips32.xml`. Attach at
  `-S` and set breakpoints **before** the secondary cores run — inserting
  one later segfaults QEMU. When that is not enough, use `fprintf` traces
  in the access/store hooks instead of runtime single-step.
- **MT7621 facts that cost weeks** (do not regress them): the GIC timer
  must run at the CPU clock (880 MHz, `MT7621_CPU_HZ`), not QEMU's default
  100 MHz; Breed needs the write-back D-cache model (`dcache=on`); reset
  must leave secondary cores powered off (`start_powered_off`) or a warm
  reboot runs the bootloader on two cores and corrupts the SPL in locked
  L2. The stock NAND driver needs instant-op `INTR_STA`, a cumulative ECC
  sector counter and FDM auto-load (all in `mt7621_nand.c`).
- **Performance**: TCG translation-block lookup dominates; device models
  are negligible. QEMU already runs each guest CPU in its own host thread
  (MTTCG); one guest thread cannot spread over more cores. `-O3`/LTO gave
  nothing — do not retry without measurements. Linux/Windows packages are
  PGO-optimised (see Build).

## Changing QEMU device models

Device models live in the local QEMU tree (after the patches are applied):

- `src/qemu/hw/arm/filogic/` — `filogic_router.c` (machine, BootROM),
  `mtk_sysctl.c` (clocks/DRAMC), `mtk_dramc_tbl.h` (generated DRAMC status
  defaults), `mtk_eth.c`, `spinand.c`, `spinor.c`, `mtk_spim.c`,
  `mtk_msdc.c`, `mtk_pcie.c`, `mtk_wmac.c`, `mt7992_wifi.c`, `mtk_wo.c`,
  `mtk_pinctrl.c`, `mtk_toprgu.c` (reset), `filogic_leds.c`, `ws2812b.c`,
  `hc595.c`, `flash_chips.h`
- `src/qemu/hw/mips/mt7621.c`, `mt7621_nand.c`, `mt7621_pcie.c` — MT7621
  machine, NFI NAND, PCIe (reuse the Filogic spinor/eth/pinctrl/LED models)
- `src/qemu/target/mips/`, `src/qemu/hw/intc/mips_gic.c` — MT7621 CPU/GIC
- `src/qemu/net/pcap.c`, `src/qemu/chardev/` — launcher-facing extras

`mtk_dramc_tbl.h` is generated by `tools/gen_dramc_table.py` from coreboot's
MT8195 DRAMC/DDRPHY headers (`src/coreboot`); do not hand-edit it.

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
- **Keep the branch equal to the exported series.** `build.sh` applies
  patches by count (`git rev-list --count v10.1.0..mt7981` vs the number of
  patch files); extra unexported local commits break that assumption, and
  a rewritten existing patch needs a fresh branch/reset to take effect in
  a clean clone. Verify the two counts match before committing.
- The patch files are the deliverable; a local `src/qemu` commit that is not
  exported will be lost for CI, packages and users.
- New machine options must be documented in the README option table and
  added to the Windows launcher lists in `windows/Presets.cs`; MT7621
  options additionally need to be in `Mt7621Keys`, or the Windows build
  silently drops them.
- Keep firmware untouched: workaround in the device model, not in the guest.

## Adding or changing a board preset

1. Copy the closest preset in `presets/`, rename it, and set `name=`,
   `description=`, `soc=` (omit for MT7981), `ram=`, `dump-dir=` and the
   machine options (Ethernet/switch/PHY, `flash-type`/`flash-id`/…, DDR,
   USB, buttons, LEDs). Build-only keys consumed by
   `prepare-nand.sh`/CI, not by QEMU: `openwrt=`, `openwrt-version=`,
   `openwrt-no-bdinfo=`, `openwrt-parts=`, `openwrt-emmc=`,
   `openwrt-emmc-layout=`, `openwrt-uboot=`, `openwrt-ubi-fip=`,
   `openwrt-ubi-factory=`, `openwrt-nor-nand=`. Launcher-only keys are
   `lan-ip=` and `lan-forwards=`.
2. Generate the LED lines from the OpenWrt DTS (never write them by hand if
   the DTS is available):

   ```bash
   python3 tools/dts-leds.py --openwrt src/openwrt PRESET      # print
   python3 tools/dts-leds.py --write --all                     # update presets
   ```

   Boards whose DTS lives in the kernel tree (e.g. Bananapi BPi-R4) need
   `tools/kernel-dts.sh` first: it applies OpenWrt's DTS patches to
   `src/linux` and writes `work/kdts`, which `dts-leds.py` reads.
3. Build the flash folder with `tools/prepare-nand.sh` using the same
   `--flash-type/--flash-id` as the preset, then boot-test it:

   ```bash
   python3 tests/quick.py -P PRESET --limit 600 \
     --qemu="-netdev user,id=wan,restrict=on" \
     'BL2: @120' 'U-Boot 20@240' 'Please enter@600'
   ```
4. Update the board table in `README.md` **and** `README.ru.md`.
5. Add the preset to `TEST_PRESETS` and both test matrices **only if it
   brings a new hardware combination** (see Tests). Do not add CI jobs that
   build custom U-Boot or modified OpenWrt images — those live in
   `stavultras/openwrt-patched-builds`.
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
  table in sync with the code; `windows/README.txt` ships in the package.
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

No flash folders and no presets are inside the packages. Locally,
`build-windows.sh` also produces `Router-Emulator-<version>-dumps.zip`
containing only the flash folders. Release commit messages are in English.

Release workflow habits:

- **Release notes**: every release gets `work/notes-<version>.md`
  ("## What's new in <v>", English), inserted into the GitHub release body
  before `**Full Changelog**` right after CI creates it; say whether the
  flash-folder archives changed.
- **Test build first**: for risky changes the maintainer often wants a
  local Windows test build (`build-windows.sh`) to check personally before
  anything is committed or released. Offer it; do not publish unasked.
- **Dumps archive**: do not rebuild `-dumps.zip` unless flash folders were
  added or changed; keep its old version number so users know not to
  re-download.

### Cloud mirrors (maintainer's machine, only on explicit request)

The release set is mirrored to Google Drive and Yandex Disk (folder
"Router Emulator", links in the README). On the maintainer's machine:

```bash
gh release download v<VERSION> -D work/rel<VERSION>   # the exact GitHub assets
~/bin/gdrive-release.sh FILE...      # Google Drive
~/bin/yadisk-release.sh FILE...      # Yandex Disk
```

- Both scripts **delete files** in the target folder that were not uploaded
  and are not listed in `GDRIVE_KEEP` / `YADISK_KEEP` (they never delete
  folders). `GDRIVE_DIR` / `YADISK_DIR` select the folder (default
  "Router Emulator"). Uploads are slow — run them in background jobs.
- Flash folders live in the `dumps/` subfolder as **one archive per
  folder**, `Router-Emulator-<folder>.zip` (e.g.
  `Router-Emulator-nand-wr3000p.zip`). Re-upload only new/changed folders
  and pass all the others in `*_KEEP` so the cleanup keeps them.
- Upload the packages + the presets archive with every release; only the
  newest `-presets.zip` may exist in the clouds. For a presets-only
  release, keep the current packages and all flash-folder zips in `*_KEEP`.
- Credentials/tokens live outside the repo (`~/.config/…`); never commit
  them. If rclone auth breaks, the maintainer must re-authorize.

## Before you commit

- QEMU change: committed in `src/qemu` **and** exported to `qemu-patches/`;
  the commit count equals the patch count; rebuilt and boot-tested with
  `tests/quick.py` on the affected machine(s).
- Preset change: LEDs generated, flash folder builds, boot test passes,
  README board tables updated in both languages.
- Launcher/UI change: `languages/en.ini` + `ru.ini` updated
  (`gen-lang-en.py --check` clean), Windows option lists updated.
- The diff contains no real MAC/serial/UID/password and no flash folders,
  dumps or factory data.
- `VERSION` untouched unless a release was explicitly requested.
- Commit message: short area prefix, imperative, English.
- Docs-only change: no build/test needed; CI will not run (`**.md` ignored).

## Conventions and pitfalls

- **Flash folders are mutable.** The guest writes into them (settings,
  sysupgrade, U-Boot environment). Copy a folder before destructive tests;
  never point a test at a folder you cannot recreate.
- **Default LAN must stay isolated.** Bridging the router LAN to `br0`
  exposes its DHCP and IPv6 RA on the real network (this leaked once on
  the maintainer's VM); `-l isolated` is the default for a reason.
  Connecting several LAN ports to the same host bridge creates a loop.
- **MT7621 is a different binary** (`qemu-system-mipsel`) and a different
  code path (`hw/mips/`); the Filogic launcher logic still applies. Its
  presets use `soc=mt7621`. See Debugging for its hard-won facts.
- **SPI-NOR vs SPI-NAND vs eMMC is never guessed from the chip ID**;
  `flash-type` decides. Chips must be added to `flash_chips.h` (and to the
  Windows editor's list) to get a size automatically.
- **`src/qemu` is not in git.** `git status` in the repository root will not
  show QEMU edits; check `src/qemu` and export patches.
- **`tests/quick.py` picks `src/qemu/build/...`, then `qemu/...`** (release
  layout); set `QEMU_BIN` to test another binary, `--win`/`QEXE` for the
  Windows build under Wine.
- **Do not reboot OpenWrt too early in tests.** It marks the overlay
  "ready" only at the end of boot; a reboot before that makes fstools wipe
  the overlay (the same happens on hardware).
- **Do not run bare `git clean`/`rm -rf` over the root**: gitignored folders
  contain multi-GB flash folders and user dumps.
- **Long builds/tests**: use background jobs and wait on their PID or a
  done-file; never poll with `pgrep -f` (see rule 4).
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
| Board boots / DDR / boot chain | `src/qemu/hw/arm/filogic/filogic_router.c`, `mtk_sysctl.c`, `mtk_dramc_tbl.h` |
| Ethernet, switch, PHYs | `src/qemu/hw/arm/filogic/mtk_eth.c` |
| NAND / NOR / eMMC | `spinand.c`, `spinor.c`, `mtk_msdc.c`, `flash_chips.h`, `tools/mknand.py`, `tools/mkemmc.py` |
| Wi-Fi / WED / WO | `mtk_wmac.c`, `mt7992_wifi.c`, `mtk_wo.c` |
| LEDs / buttons / reset | `mtk_pinctrl.c`, `filogic_leds.c`, `ws2812b.c`, `hc595.c`, `mtk_toprgu.c`, `tools/dts-leds.py` |
| MT7621 (MIPS) | `src/qemu/hw/mips/mt7621*.c`, `hw/intc/mips_gic.c`, Debugging section |
| Board definition | `presets/*.ini`, README option table |
| Flash image creation | `tools/prepare-nand.sh`, `tools/mknand.py`, `tools/mkemmc.py` |
| Launcher (Linux) | `emulator.sh` |
| Launcher (Windows) | `windows/*.cs`, `windows/README.txt`, `build-windows.sh` |
| CI / releases | `.github/workflows/build.yml`, `VERSION`, `tools/package-*.sh` |
| Boot-chain regression tests | `tests/quick.py`, `tests/powercut.py` |
| Boot failure diagnosis | `logs/console_*.log`, `work/quick.log(.err)`, `-d` → `work/qemu.log` |

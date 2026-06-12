# omen-dkms

DKMS package providing an enhanced `hp-wmi` kernel module for HP OMEN and Victus
gaming laptops on Linux. This driver extends the upstream Linux `hp-wmi` module
with board support, sysfs interfaces, and fan-control reliability fixes needed
for recent OMEN Slim and related models.

The module replaces the in-tree `hp_wmi` driver at load time via DKMS and exposes
standard interfaces used by tools such as [OmenCore](https://github.com/theantipopau/omencore).

## Supported hardware

This fork is developed and tested primarily on:

| Board ID | Example model | Notes |
|----------|---------------|-------|
| `8D40` | OMEN Slim 16-an0xxx / 16-an0015tx | Victus-S thermal path, four-zone RGB, improved fan AUTO handoff |
| `8E35` | HP OMEN 16 | Four-zone RGB, GPU thermal mode probing |

Other boards already supported by the imported upstream driver may work unchanged.
Adding a board ID requires confirming the correct thermal profile table and
firmware behavior for that SKU.

## Changes from upstream

This repository is based on the Linux kernel `hp-wmi` driver (imported at
`cef99c0`) with the following additions and fixes (`789dfa1` and later):

### Board support

- Added DMI board `8D40` to `victus_s_thermal_profile_boards` with
  `omen_v1_no_ec_thermal_params`, enabling the Victus-S platform profile and
  hwmon path on OMEN Slim 16 hardware.
- Added `8E35` to `omen_thermal_profile_boards` and
  `omen_timed_thermal_profile_boards` where missing from the imported base.

### Four-zone keyboard backlight (sysfs)

New WMI command definitions and sysfs attributes under
`/sys/devices/platform/hp-wmi/`:

| Attribute | Access | Description |
|-----------|--------|-------------|
| `fourzone_color` | read/write | 24-character hex string (4 zones x 6 hex digits RGB) |
| `fourzone_brightness` | read/write | Global brightness (0-255) |
| `fourzone_animation` | read/write | Firmware animation mode index |

Positive WMI return codes from the EC are treated as errors and propagated to
userspace as I/O errors.

**Physical zone layout** (left to right on a full-size OMEN keyboard):

| Zone | Location |
|------|----------|
| Zone 3 | Left edge (Esc / Tab / modifier column) |
| Zone 4 | WASD cluster |
| Zone 2 | Middle section (G row through center) |
| Zone 1 | Right section and numpad (including middle Enter) |

The sysfs string order is Zone 1, Zone 2, Zone 3, Zone 4 as defined by the
firmware protocol.

Example: set all zones to OMEN blue (`00BFFF`):

```bash
echo "00BFFF00BFFF00BFFF00BFFF" | sudo tee /sys/devices/platform/hp-wmi/fourzone_color
```

### Fan control reliability

- Added `hp_wmi_fan_speed_max_get()` and `hp_wmi_fan_speed_max_set_verify()` with
  retry logic so max-mode transitions are confirmed by the EC.
- Added `hp_wmi_omen_exit_userdefined_mode()` to reliably leave experimental
  fan-stop / user-defined PWM mode on boards that do not clear state on a
  single `max_set(0)`.
- Added `hp_wmi_omen_fan_speed_reset()` for Victus-S manual fan baseline.
- `hp_wmi_apply_fan_settings()` now accepts `prev_mode` and only runs the full
  exit-pulse sequence when transitioning from `PWM_MODE_MANUAL` to
  `PWM_MODE_AUTO`, avoiding unnecessary fan bursts on other transitions.
- Replaced `cancel_delayed_work_sync()` with `cancel_delayed_work()` when
  holding `priv->lock`, preventing a deadlock with the keep-alive worker.

### GPU thermal modes

- Replaced a hardcoded board check with `omen_has_gpu_thermal_modes()`, which
  probes Victus-S CTGP/PPAB WMI support at runtime so compatible firmware is
  used without maintaining a board-name allowlist.

### Build dependencies

- Added `#include <linux/delay.h>` and `#include <linux/hex.h>` for the above.

For a line-level view of fork changes against the import baseline:

```bash
git diff cef99c0..HEAD -- hp-wmi.c
```

## Requirements

- Linux with kernel headers installed (`/lib/modules/$(uname -r)/build`)
- DKMS (`dkms` package)
- Build tools: `gcc`, `make`
- Root access to load kernel modules

Supported distributions: any distro with DKMS (Arch, CachyOS, Fedora, Ubuntu,
Debian, etc.).

## Installation

Clone this repository and install with DKMS:

```bash
git clone https://github.com/saikiran2001-v2/hp-omen-dkms.git omen-dkms
cd omen-dkms

sudo dkms add .
sudo dkms build hp-wmi/1.0.0
sudo dkms install hp-wmi/1.0.0
```

Reload the module:

```bash
sudo modprobe -r hp_wmi
sudo modprobe hp_wmi
```

Verify platform profile registration:

```bash
cat /sys/class/platform-profile/*/choices
cat /sys/class/platform-profile/*/profile
```

Verify keyboard sysfs nodes:

```bash
ls /sys/devices/platform/hp-wmi/fourzone_*
```

Verify hwmon fan interface:

```bash
ls /sys/devices/platform/hp-wmi/hwmon/hwmon*/pwm1_enable
```

## Updating

After pulling new changes:

```bash
sudo dkms remove hp-wmi/1.0.0 --all
sudo dkms add .
sudo dkms install hp-wmi/1.0.0
sudo modprobe -r hp_wmi && sudo modprobe hp_wmi
```

## Uninstallation

```bash
sudo modprobe -r hp_wmi
sudo dkms remove hp-wmi/1.0.0 --all
sudo modprobe hp_wmi   # loads the stock in-tree module, if present
```

## Platform profile values

On Victus-S boards such as `8D40`, available profiles typically include:

- `low-power`
- `balanced`
- `performance`

Set via:

```bash
echo balanced | sudo tee /sys/class/platform-profile/*/profile
```

## Fan control (hwmon)

Fan mode is exposed through the hwmon `pwm1_enable` attribute:

| Value | Mode |
|-------|------|
| `0` | Max (immediate full speed) |
| `1` | Manual / user-defined (fan-stop capable on supported boards) |
| `2` | Auto (BIOS/EC managed) |

Paths vary by kernel version; locate with:

```bash
find /sys/devices/platform/hp-wmi -name pwm1_enable
```

## License and attribution

This project is licensed under the **GNU General Public License v2.0** (or
later). See [LICENSE](LICENSE) for the full text.

Upstream `hp-wmi` is Copyright (C) Matthew Garrett, Anssi Hannula, and other
Linux kernel contributors. Modifications in this repository are Copyright (C)
2026 saikiran.

You may modify and redistribute this code under the GPL. If you redistribute
it, you must preserve copyright notices and document your changes. Do not
present this work or substantial portions of it as your own without attribution.
See [NOTICE](NOTICE) for details.

## Contributing

Contributions are welcome. Please:

1. Describe the board ID and hardware tested.
2. Keep changes focused and documented in commit messages.
3. Preserve existing copyright headers in `hp-wmi.c`.
4. Do not remove attribution to upstream or fork authors.

## Disclaimer

This is an unofficial community driver. It is not affiliated with or endorsed
by HP. Loading an out-of-tree kernel module may affect system stability,
thermal behavior, and warranty coverage depending on your jurisdiction and
hardware. Test on your own system and keep backups of working configurations.

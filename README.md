# asus-zenbook-a14-ec

Out-of-tree Linux kernel drivers for the **ASUS Zenbook A14**.
Provides fan/temperature sensors, power profiles, keyboard backlight, and Fn
hotkeys on Linux.

Forked from [Sombre-Osmoze/asus-zenbook-a14-ec](https://github.com/Sombre-Osmoze/asus-zenbook-a14-ec)
(RA / X1E80100). This fork is tuned for **UX3407QA** (Snapdragon X1P, x1p42100)
and adds the changes needed for QA hardware. RA users can use it too — the QA
path falls back to the RA fan-mode dressup automatically when WEBC NACKs.

## Hardware compatibility

| Variant   | SoC      | Status                                       |
|-----------|----------|----------------------------------------------|
| UX3407QA  | X1P      | Tested on QA hardware. WEBC profiles work.   |
| UX3407RA  | X1E      | Untested in this fork. Should fall back to fan-mode dressup as in upstream. |

## Modules

| Module                  | Function                                         |
|-------------------------|--------------------------------------------------|
| `asus_zenbook_a14_ec`   | EC hwmon (fan/PWM/temp) + platform_profile       |
| `hid_asus_ec`           | Keyboard backlight LED class + Fn hotkeys        |

## Status

### EC driver (`asus_zenbook_a14_ec`)

- **hwmon**: `fan1_input`, `pwm1`, `pwm1_enable`, `temp1_input`
  - `temp2` removed (sub-register `(0x05, 0x01)` returns a constant 103°C on QA;
    not a real second thermistor).
- **platform_profile**: `quiet` / `balanced` / `performance` / `max-power`
  - QA: WEBC(0x11) accepted by the EC; profile change is a single block-write,
    fan stays in EC AUTO afterwards.
  - `max-power` uses the firmware profile byte `0x10` on QA.
  - If WEBC NACKs (RA path), driver falls back to fan-mode dressup
    (quiet/balanced = AUTO, performance = MANUAL + PWM 180,
    max-power = MANUAL + PWM 69).
- **Manual PWM**: works; A14 has no watchdog.
- **Suspend/resume**: clean entry/exit on the driver side. Note: deep sleep
  bails after ~2 s on this kernel for unrelated PSCI reasons (use `s2idle`).
- DT bind via `compatible = "asus,zenbook-a14-ec"` at I2C address 0x5b.

### HID driver (`hid_asus_ec`)

- **Keyboard backlight**: LED class `asus::kbd_backlight`, 4 levels (0–3).
- **Fn+F4 cycle key**: advances backlight 0→1→2→3→0. Driver tracks state
  internally; the EC report's payload byte is 0 every press (it's the
  key-state byte, not current brightness — fixing this was a one-line patch
  versus the original RA driver).
- **Other Fn hotkeys**: F5/F6 (brightness), F8 (emoji), F9 (micmute),
  F10 (camera), F11 (touchpad), F12 (PROG1), F (KEY_PERFORMANCE).
- Target device: `0B05:0220` (I2C-HID keyboard).

## Kernel patches

`patches/0001-platform_profile-allow-non-ACPI-systems.patch`
Drops the `acpi_disabled` early-return in `drivers/acpi/platform_profile.c`
and guards the firmware/acpi sysfs path with `if (acpi_kobj)`. The class
device (`/sys/class/platform-profile/platform-profile-0/`) registers on
DT-only systems.

```sh
cd /path/to/linux
git apply /path/to/patches/0001-platform_profile-allow-non-ACPI-systems.patch
```

You also need `embedded-controller@5b { compatible = "asus,zenbook-a14-ec"; reg = <0x5b>; };`
inside the `&i2c5` node of your DTS, plus `CONFIG_HWMON=y` and the kernel's
`platform_profile` class enabled.

## power-profiles-daemon (optional)

Stock `power-profiles-daemon` 0.30 only reads `/sys/firmware/acpi/platform_profile`,
which doesn't exist on DT-only kernels. Two options:

1. **Patch PPD** (recommended) —
   `patches/0002-ppd-class-platform-profile-fallback.patch` adds a fallback
   to `/sys/class/platform-profile/*` when the firmware path is absent.
   ```sh
   git clone --branch 0.30 https://gitlab.freedesktop.org/upower/power-profiles-daemon.git
   cd power-profiles-daemon
   git apply /path/to/patches/0002-ppd-class-platform-profile-fallback.patch
   meson setup build --prefix=/usr --libexecdir=/usr/libexec --buildtype=release \
       -Dgtk_doc=false -Dtests=false -Dpylint=disabled
   ninja -C build && sudo ninja -C build install
   sudo systemctl restart power-profiles-daemon
   ```

2. **Userspace bridge** — `scripts/ppd-bridge.py` exposes the kernel class
   device on the `net.hadess.PowerProfiles` D-Bus name, for environments
   where you can't patch PPD.

## Build & install

```sh
make                       # both .ko against running kernel
make KDIR=/path/to/linux   # against a specific tree
sudo modprobe platform_profile
sudo insmod ./asus_zenbook_a14_ec.ko
sudo insmod ./hid_asus_ec.ko
```

For an in-tree kernel package build, enable:

```text
CONFIG_EC_ASUS_ZENBOOK_A14=m
CONFIG_HID_ASUS_EC=m
```

and add the EC node under the Zenbook A14 `&i2c5` device-tree node.

## Validation

After booting the target kernel, the ready state should look like this:

```sh
uname -r
lsmod | grep -E 'asus_zenbook_a14_ec|hid_asus_ec|platform_profile'
cat /sys/class/platform-profile/platform-profile-0/name
cat /sys/class/platform-profile/platform-profile-0/choices
cat /sys/class/platform-profile/platform-profile-0/profile
ls /sys/class/leds | grep 'asus::kbd_backlight'
sensors | grep -A4 asus_zenbook_a14_ec
```

Expected profile choices for the validated path:

```text
quiet balanced performance max-power
```

First-write testing should be conservative: switch one profile at a time, wait
for fan/temperature readings to settle, and return to `balanced` before
continuing.

## Exposed interfaces

| sysfs                                                  | semantics                          |
|--------------------------------------------------------|------------------------------------|
| `hwmon/hwmonN/fan1_input`                              | RPM (tach × 88)                    |
| `hwmon/hwmonN/pwm1`                                    | 0–255 (RW; takes effect in manual) |
| `hwmon/hwmonN/pwm1_enable`                             | 1 = manual, 2 = auto (RW)          |
| `hwmon/hwmonN/temp1_input`                             | EC thermistor, m°C                 |
| `leds/asus::kbd_backlight/brightness`                  | 0–3                                |
| `class/platform-profile/platform-profile-0/profile`    | quiet / balanced / performance / max-power |

## Scripts

| Script                          | Purpose                                  |
|---------------------------------|------------------------------------------|
| `scripts/ppd-bridge.py`         | Userspace PPD shim (alternative to patching PPD). |
| `scripts/capture-fkeys.py`      | HID feature-report enumeration / Fn capture. |
| `scripts/test-4543.py`          | Probe the second ASUS HID device (`0B05:4543`). |

## Hardware safety

The driver enforces:

- Major-allowlist on EC writes (`0xC4` mailbox, `0xC6` sensors, `0xC9` block window).
- Profile bytes are enum-only (`0x01/0x02/0x04/0x10`); no raw-byte path.
- WEBC kick is fire-and-forget; the per-device mutex is released within
  microseconds (the post-kick poll deadlock from earlier RE attempts is gone).
- Probe-time sanity read gates hwmon registration; on garbage, probe aborts
  before any write.

A14 has **no fan watchdog** (verified: 3 + min manual mode = no reboot), so
manual PWM is safe without a temperature-feed loop. **Vivobook S15 and
similar Snapdragon laptops do** — porting needs a watchdog kthread.

Do not run raw userspace I2C tools against `0x5b` or `0x76` while
`asus_zenbook_a14_ec` is bound. The kernel driver owns the EC mailbox and fan
controller; concurrent raw writes can race the driver.

If anything misbehaves: hard power-cycle and pick a working kernel from the
bootloader.

## Credits

- **Sombre-Osmoze** &lt;sombre@osmoze.xyz&gt; — original RA driver,
  EC reverse-engineering, platform_profile patch, PPD bridge.
- **Alexandru Marc Serdeliuc** &lt;serdeliuk@yahoo.com&gt; — original
  `hid-asus-ec` keyboard backlight driver, QA backlight protocol confirmation.
- **Ramshouriesh** &lt;rshouriesh@gmail.com&gt; — QA fork: WEBC profile
  path, DT i2c_client bind, Fn+F4 backlight cycle fix, PPD class-device
  fallback patch, end-to-end QA hardware verification.
- **icecream95** — udev-hid-bpf work on Vivobook S15 / Zenbook A14, early
  EC protocol documentation.

## License

GPL-2.0-only (EC driver, kernel patch, PPD patch).
GPL-2.0-or-later (HID driver, per upstream).

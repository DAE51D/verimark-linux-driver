# Ubuntu / Kubuntu 26.04: build notes and test report

Second-unit, second-distro report for the `driver/` (clean-room) libfprint driver, plus the
Debian-family translation of the dependency list and the KDE/PAM pitfalls we hit. Everything
below was done on real hardware; where something was *not* tested it says so.

## Environment

| | |
|---|---|
| Distro | Kubuntu 26.04.1 LTS ("Resolute Raccoon") |
| Kernel | 7.0.0-34 / 7.0.0-38 generic |
| Desktop | Plasma 6.6.6, SDDM 0.21 |
| `fprintd` / system `libfprint` | 1.94.5-4 / 1.95.1+tod1-0ubuntu2 (left untouched) |
| libfprint built against | upstream `main`, commit `6f9479c3d55f` (2026-09-02), reports `1.94.100` |
| This repo | `1239ab8` |
| Device | Kensington VeriMark Desktop K62330WW, `047d:00f2`, factory-fresh, product string "VeriMark DT Fingerprint Key" |

## Result

| Operation | State |
|---|---|
| Build against libfprint `main` (1.94.100) | Works, no source changes needed |
| First enroll on a factory-fresh unit (this is the pairing step) | Works, 8 stages, two fingers enrolled |
| `fprintd-verify` | Works, single tap |
| Plasma lock-screen unlock | Works |
| SDDM login | Works, with the caveat below |
| `sudo` | Not tested (passwordless sudo on this machine) |
| "Touch now" LED (white, front of the key) | Works (lights while waiting for a touch) |
| Delete / clear storage | Not tested |

`fprintd-list` showed "No devices available" before the driver was installed, and
`fido2-token -I` on the key's HID interface reports `caps: nocbor, msg` (U2F only), so the
fingerprint sensor really is only reachable through the vendor interface this driver uses.

## Dependencies (apt)

```
sudo apt install meson ninja-build cmake gcc g++ pkgconf git rsync \
  libglib2.0-dev libgusb-dev libssl-dev libjson-glib-dev libpixman-1-dev \
  libnss3-dev libgirepository1.0-dev libsystemd-dev libcairo2-dev \
  libudev-dev libgudev-1.0-dev systemd-dev fprintd libpam-fprintd
```

`systemd-dev` is the non-obvious one. libfprint's `meson.build` asks for the `udev`
pkg-config module, and on Debian/Ubuntu `udev.pc` ships in `systemd-dev`, not in
`libudev-dev`. Without it `meson setup` stops with
`Dependency "udev" not found, tried pkgconfig and cmake`.

## Build and install

`setup-libfprint-build.sh` calls `dnf`, so skip its dependency step. There is also no
`v1.95.1` tag upstream (Ubuntu's 1.95.1 is a development snapshot), so use the default
clone of `main`, or pass a checkout:

```sh
git clone --depth 1 https://gitlab.freedesktop.org/libfprint/libfprint.git ~/verimark-build/upstream-libfprint

./driver/setup-libfprint-build.sh --skip-deps --skip-udev --drivers=default \
  --work-dir ~/verimark-build --libfprint-src ~/verimark-build/upstream-libfprint

./driver/install-verimark.sh --build-dir ~/verimark-build/libfprint/build
```

`--drivers=default` builds every stock driver plus `verimark`, so other readers keep
working. The SELinux relabel step is skipped automatically (Ubuntu uses AppArmor; no
AppArmor changes were needed). Offline tests (`meson test`) pass.

Check it took effect:

```sh
systemctl show fprintd -p Environment   # LD_LIBRARY_PATH=/usr/local/lib64/verimark-libfprint
fprintd-list "$USER"                    # should now list the Kensington device
```

## Pitfalls

**Other `fprintd` drop-ins can silently win.** systemd merges
`/etc/systemd/system/fprintd.service.d/*.conf` alphabetically and a later file's
`Environment=LD_LIBRARY_PATH=` replaces an earlier one. A leftover `override.conf` from
another experimental driver sorted after `10-verimark-libfprint.conf`, so `fprintd` kept
loading the wrong library and reported "No devices available" with no error. If
`systemctl show fprintd -p Environment` does not show the verimark path, look for other
drop-ins.

**`install-verimark.sh` has a hard-coded default `--build-dir`** (`/home/sshadows/...`).
Pass `--build-dir` explicitly unless your path happens to match.

## KDE / PAM

Do **not** enable the distro's global fprintd PAM profile
(`pam-auth-update --enable fprintd`). It puts `pam_fprintd` in `common-auth`, so every
password check (lock screen, polkit, login) first waits up to 10 s for a finger. Plasma
does not need it: the lock screen uses its own fingerprint-only PAM service,
`/usr/lib/pam.d/kde-fingerprint`, which already worked with no PAM changes.

**SDDM is the exception and has an inherent limit.** The greeter collects username and
password and only starts PAM on Enter, so it can never show a fingerprint prompt or light
the reader on its own. To use the finger: type the username, leave the password empty, press
Enter, then touch the sensor. To keep that without slowing down typed passwords, and to keep
the `pam_kwallet5` / `pam_gnome_keyring` lines after `common-auth` running (a `success=done`
would skip them), put this before `@include common-auth` in `/etc/pam.d/sddm`:

```
auth    [success=3 default=ignore]      pam_unix.so try_first_pass
auth    [success=2 default=ignore]      pam_fprintd.so max-tries=1 timeout=10
```

The jump counts assume `common-auth` is `pam_unix`, `pam_deny`, `pam_permit`, `pam_cap`;
re-check them after any `pam-auth-update`. Keep a text console (Ctrl+Alt+F3) handy while
editing PAM, and a backup of `/etc/pam.d`.

**KWallet.** A fingerprint login never supplies the login password, so KWallet cannot be
unlocked by it and prompts afterwards. KWallet is not only for explicit passwords:
Chromium-family browsers and Electron apps keep their "Safe Storage" key there, so disabling
the wallet subsystem makes their saved data unreadable. Setting an empty wallet password
removes the prompt without losing anything (at the cost of those keys being stored without a
wallet password).

## Notes

- The reader's white LED lights while the driver waits for a touch and turns off when a frame
  arrives. It uses the `0x39` LED_EX2 command (frames from the Windows capture, findings/25),
  is purely cosmetic, and any LED error is ignored so it can never fail a login. Set
  `VERIMARK_LED=0` in `fprintd`'s environment to disable all LED traffic. The wait that
  matters is the frame wait (`CAP_FRAME_ACQ`/`CAP_WAIT_FRAME`): `CAP_WAIT_PRESS` passes
  immediately with no touch, so lighting the LED around it is invisible.
- Pairing was a non-event on a factory-fresh unit, but it is one-way, as the READMEs say. We
  did not test what happens when a unit that was previously paired elsewhere is used.

---
name: omarchy-dual-boot
description: >
  Apply the Omarchy dual-boot Limine menu on a fresh Omarchy install: Omarchy
  boots by default after 5 seconds, Windows 11 is the other entry, snapshot
  UKIs stay off the ESP, and Secure Boot is enrolled with sbctl plus Microsoft
  keys. Use when the user runs /omarchy-dual-boot, or asks for dual boot, a
  Limine timeout, default Omarchy boot, or the same boot menu on a new Omarchy.
metadata:
  short-description: "Dual boot: Omarchy after 5s, Windows beside it"
---

# Dual boot Omarchy

On a fresh Omarchy install with Windows on a separate disk, bring the menu to this state:

- Two top-level entries: `/Omarchy` (default, boots the UKI directly) and `/Windows 11` (chainload, no hash).
- `timeout: 5` and `quiet: no`. Waiting boots Omarchy.
- Snapshots stay on the btrfs disk. They are not in the menu and their UKIs are not on the ESP.
- Secure Boot uses sbctl's own keys plus the Microsoft keys, so Windows still boots.

Show what the disks contain before changing anything. Write privileged files with `pkexec` when the terminal cannot prompt for a password. Use `sudo` when the user is at an interactive prompt.

## Leave these alone

- The BitLocker partition (the Windows system volume). You may read the small Windows EFI partition read-only, then unmount it.
- `/usr/share/omarchy/` and `/etc/limine-entry-tool.d/omarchy-defaults.conf`. The drop-in belongs to the package. Overrides go in `/etc/default/limine`, which Limine reads last.
- `sbctl create-keys` when keys already exist.
- `sbctl enroll-keys` unless the firmware is in Setup Mode. Always pass `--microsoft`.
- A hash on the Windows path. Firmware checks the Microsoft signature. A hash breaks the chainload after a Windows update.

## Read first

Before any change: `lsblk -o NAME,SIZE,FSTYPE,LABEL,PARTUUID,MOUNTPOINT`, `findmnt /boot`, `sbctl status`, `efibootmgr`, `/etc/default/limine`, and `$ESP/limine.conf`. The ESP is the vfat filesystem that contains `EFI/limine`, usually mounted at `/boot`. Windows EFI is the other vfat partition that contains `EFI/Microsoft/Boot/bootmgfw.efi`.

If there is no Windows EFI partition, stop and say so. Do not guess the path.

## Configuration

In `/etc/default/limine`, keep `ESP_PATH` and every `KERNEL_CMDLINE` line. Set these three keys, replacing an existing value when the key is already present:

```
ENABLE_ENROLL_LIMINE_CONFIG=yes
ENABLE_LIMINE_FALLBACK=no
MAX_SNAPSHOT_ENTRIES=0
```

`ENABLE_ENROLL_LIMINE_CONFIG` takes effect only from this file. Limine clears that variable before it reads `/etc/default/limine`.

`MAX_SNAPSHOT_ENTRIES=0` and `ENABLE_LIMINE_FALLBACK=no` override `omarchy-defaults.conf` (`6` and `yes`). The fallback file may remain on the ESP, but an update must not replace a signed file with an unsigned one, and the menu has no fallback entry.

In the `$ESP/limine.conf` header, set these without duplicates:

```
timeout: 5
quiet: no
default_entry: Omarchy
```

An Omarchy migration writes `timeout: 1` and `quiet: yes` only when no timeout is set yet. An uncommented `timeout: 5` is left as it is.

## Hook

The script is beside this file: `scripts/80-flat-boot-menu`. Install it as `/etc/boot/hooks/post.d/80-flat-boot-menu` with mode `0755`.

On `limine-update` the order is: `10-limine-reset-enroll` clears the checksum in the Limine binary, `limine-entry-tool` and `limine-snapper-sync` write the tree, this hook flattens it to two entries, and `90-limine-enroll-config` enrolls the `limine.conf` checksum into Limine and signs the binary when sbctl has keys.

The hook leaves the file unchanged when Omarchy or Windows is missing, or when Omarchy has no kernel path. It recognizes Windows by the word "Windows" in the title or by `bootmgfw.efi` in the path, and it strips a `#hash` suffix from that path.

A snapper warning that more snapshots exist than `MAX_SNAPSHOT_ENTRIES=0` is expected. So is the warning from `89-warn-missing-file-hashes`. Do not add `ENABLE_VERIFICATION`.

## Apply

1. Copy `limine.conf` to `limine.conf.bak-before-flat` beside it, only when that copy does not already exist.
2. Write the three keys, the header, and the hook.
3. Run `limine-update`.
4. Check the menu before touching Secure Boot. If the header is missing, write it again and run `limine-update` again. The checksum inside the Limine binary must match the file, so after enrollment do not edit `limine.conf` without another `limine-update`.

If Windows is missing after `limine-update`, mount the Windows EFI partition read-only, run `limine-update` again, and unmount it. The hook has to see that entry.

## Secure Boot

Do this only when the user wants Secure Boot on.

1. `pacman -S --needed sbctl`
2. `sbctl create-keys` only when no keys exist.
3. `sbctl sign -s` the Omarchy UKI (`$ESP/EFI/Linux/*.efi`) and `$ESP/EFI/BOOT/BOOTX64.EFI` when that file exists. Enrollment signs the Limine binary during `limine-update`. After signing, run `limine-update` once more.
4. Reboot to firmware and enter Setup Mode. Do not clear the keys when enrollment already succeeded and only the menu needs a fix.
5. In Setup Mode: `sbctl enroll-keys --microsoft`.
6. The user turns Secure Boot on in firmware and reboots.
7. `sbctl verify` must pass for `limine_x64.efi`, the Omarchy UKI, and `BOOTX64.EFI` when it exists. `sbctl status` must show Vendor Keys `microsoft` and Secure Boot enabled.

When Secure Boot is already enabled and Setup Mode is disabled: do not run `create-keys` or `enroll-keys`. Sign files with the existing keys and rebuild the menu. If sbctl has no keys and the firmware has someone else's, stop.

Before the first Windows boot, say that BitLocker may ask for the recovery key from the Microsoft account.

## Check

Show the resulting menu. It must have:

- `timeout: 5`, `quiet: no`, `default_entry: Omarchy`
- Omarchy: `protocol: efi` and `path: boot():/EFI/Linux/….efi#…`
- Windows: `path: uuid(<EFI PARTUUID>):/EFI/Microsoft/Boot/bootmgfw.efi` with no `#`
- no Snapshots, no kernel submenu, and no fallback entry

In `efibootmgr`, Limine is first in `BootOrder`. Do not remove the Windows Boot Manager entry.

## If Omarchy will not boot

Turn Secure Boot off in firmware and boot Limine. Do not clear the keys unless enrollment has to be repeated from Setup Mode.

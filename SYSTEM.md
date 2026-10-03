# Windows 11 + EndeavorOS Dual-Boot Guide

A from-scratch guide for adding EndeavorOS alongside an existing Windows install, across two separate SSDs so neither OS's disk ever needs resizing. Covers the Windows 11 upgrade, Btrfs subvolumes with Snapper/grub-btrfs for restorable snapshots, a unified GRUB boot menu, and Secure Boot with self-managed keys via sbctl.

**Setup:**
- **Samsung** — Samsung SSD 970 EVO Plus 1TB (931 GB), currently Windows, drive `C:`
- **WD** — WD Black SN750 1TB, model WDS100T3X0C-00SJG0 (931 GB), currently drive `D:` — target for EndeavorOS

Both drives are healthy (Samsung: 4% lifetime used, 60TB written; WD: 0% used, 7TB written) — wear isn't a factor in any decision below.

**Goal:** Upgrade Windows to 11, then add EndeavorOS on the WD drive — no disk encryption, keep Windows fully intact.
**Out of scope (handled by chezmoi later):** Hyprland, apps, dotfiles/configs. This plan stops at "you have a working, networked EndeavorOS base install."

## Key decision: keep the two disks separate

One SSD per OS avoids the partition-resizing work most dual-boot guides spend their effort on (shrinking one Windows partition to make room for Linux).

- **Samsung** keeps 100% of the current Windows install, untouched, no resizing.
- **WD** gets wiped and dedicated entirely to EndeavorOS, with its own EFI partition.
- GRUB (installed on the WD drive) becomes the unified boot menu — EndeavorOS as the default entry, Windows as a selectable second option — see Phase 6.

**Note:** the WD drive already has a drive letter (`D:`) in Windows, which means it's currently holding data — that gets fully erased in Phase 4, so back it up in Phase 1.

## On encryption

Skipping LUKS on the Linux side is a reasonable call for a single-user home PC with no sensitive data on it — it also minimizes data-loss risk, since there's no passphrase or key file that can lock anyone out of their own data if something goes wrong. The trade-off is that anyone with physical access to the WD drive could read it directly. One thing worth knowing: adding full-disk encryption *later* generally means reinstalling, it's not something you bolt on afterward — so if there's any chance it'll be wanted, now is the only cheap time to do it. Nothing else in this guide changes either way.

---

## Phase 0 — Pre-flight checks (do first)

- [ ] Firmware boot mode: **UEFI**, confirmed via `msinfo32`
- [ ] Samsung disk partition style: **GPT**, confirmed via Disk Management
- [ ] TPM 2.0: **currently disabled in BIOS.** This will block the normal Windows 11 upgrade path in Phase 2 (Windows Update / PC Health Check require TPM 2.0 enabled, not just present) — enable it in BIOS before starting Phase 2.
- [ ] Secure Boot state: **ON** — Phase 3 has you disable it temporarily for the EndeavorOS install; Phase 7 re-enables it.
- [ ] BitLocker/Device Encryption: **none on either volume** (`manage-bde -status` shows Protection Off, no key protectors, on both `C:` and `D:`) — Phase 1's recovery-key step is moot, nothing to save.

## Phase 1 — Backup (before any partitioning)

- [ ] Back up personal files off the Windows (Samsung) disk regardless of the encryption decision above
- [ ] Back up everything on the WD drive (`D:`) — it will be fully erased in Phase 4
- [ ] License keys checked — nothing needed saving

## Phase 2 — Upgrade Windows 10 → Windows 11

Do this **before** touching partitions. Keeping the Windows disk completely stable during this phase, and only introducing the second OS afterward, is the single biggest risk-reducer in this guide.

1. Reboot into BIOS and enable TPM — it's currently disabled, and Windows Update/PC Health Check won't offer the Windows 11 upgrade without it. On this Intel CPU (i7-10700K) that's **Advanced → PCH-FW Configuration** → set **PTT** to **Enabled**. Two more worth setting while in there: **Advanced → USB → Legacy USB Support: Off** (probably already effectively off, since Secure Boot being ON per Phase 0 implies CSM is already disabled — harmless to confirm explicitly) and **Boot → Secure Boot → OS Type: Windows UEFI Mode**. Exact wording varies by motherboard vendor.
2. Settings → Windows Update → check for updates. If Windows 11 is offered, take the in-place upgrade (keeps files, apps, settings).
3. If it's not offered, run the PC Health Check app to see why, then use the Windows 11 Installation Assistant (in-place) rather than the Media Creation Tool's clean-install route — you don't want to touch partitions here at all.
4. Reboot, then run Windows Update repeatedly until fully current — driver updates matter for the steps that follow.
5. Re-check Secure Boot state and BitLocker/Device Encryption status with [`manage-bde -status`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde) — Windows 11 sometimes auto-enables Device Encryption on qualifying hardware during setup, even though neither volume has it now. Save a recovery key if that happens.
6. Check that Windows has applied Microsoft's 2023 Secure Boot certificate (needed for Phase 7). In an admin PowerShell, with Secure Boot still on: [`[System.Text.Encoding]::ASCII.GetString((Get-SecureBootUEFI db).bytes) -match 'Windows UEFI CA 2023'`](https://learn.microsoft.com/en-us/powershell/module/secureboot/get-securebootuefi) — `True` is what you want. If `False`, run Windows Update again and check for a BIOS update.

## Phase 3 — Prepare install media (Ventoy, one USB for both ISOs)

1. Download the EndeavorOS ISO and checksum from endeavouros.com; verify the checksum. Have a Windows ISO on hand too — handy as recovery media if a repair install is ever needed.
2. [Install Ventoy](https://www.ventoy.net/en/doc_start.html) to a USB drive (use a current release — recent versions fixed [Secure Boot shim/certificate issues](https://www.ventoy.net/en/doc_secure.html)), then copy both ISOs onto it. No reformatting needed to add/swap images later.
3. In firmware setup, temporarily **disable Secure Boot** for the EndeavorOS install — Arch-based kernels aren't signed for it out of the box. On this board, that's the same **Boot → Secure Boot → OS Type** setting from Phase 2: switch it to **Other OS**. Leave it there through Phase 6, since you're running an unsigned install that whole time. Phase 7 switches it back.

## Phase 4 — Partition & install EndeavorOS

1. Boot the USB; confirm the firmware's boot menu shows it as a **UEFI** entry, not a Legacy one.
2. Connect to the network (needed for this step) and start the installer, choosing **[Online install](https://discovery.endeavouros.com/installation/installation-intro/2021/03/)**.
3. When you reach partitioning, choose **manual partitioning** and target only the WD drive:

   | Partition | Size | Type | Notes |
   |---|---|---|---|
   | EFI system partition | ~512 MB | FAT32 | mount at `/boot/efi`, flag `boot` (`esp`) — a **new** one on the WD drive, don't try to reuse Windows' ESP on the Samsung |
   | Root | remaining space | Btrfs | mount at `/`, format it. Calamares' Btrfs install creates `@` (`/`), `@home`, `@cache` and `@log` by itself |

   No separate swap partition — zram covers swap, since hibernation isn't being set up here (checked in Phase 5). Double-check the target disk by size/model before confirming (Samsung vs WD) — this step only needs to be destructive to the WD drive.
4. In the desktop-selection screen, pick **Base** (top of the list) — a real no-desktop option, not Xfce. It skips desktop packages, theming, and config entirely, so there's nothing to strip out later once chezmoi sets up Hyprland. (The offline installer doesn't offer Base — only Xfce/KDE — which is why online install is needed here.)
5. When prompted for a bootloader, choose **GRUB** and confirm it targets the WD drive's new ESP.
6. Finish and reboot into the new system (a console login, since Base has no desktop).

## Phase 5 — Add your extra Btrfs subvolumes

Calamares already creates `@`, `@home`, `@cache` and `@log` automatically when Btrfs is selected as the root filesystem. What's missing is `@home_cache`, `@home_containers`, and `@snapshots`. Do this right after first boot, before the home directory fills up.

1. Confirm what Calamares made: `sudo btrfs subvolume list -t /`, `findmnt -t btrfs`, `cat /etc/fstab`. The mount options used throughout are just `noatime,compress=zstd` — nothing else needs to be written explicitly ([`ssd`, `discard=async`, `space_cache=v2`](https://wiki.archlinux.org/title/Btrfs#Mount_options) show up on their own via kernel/device detection). If any line has `subvolid=`, drop that option: it pins an ID that changes when `@` is restored.
2. [Create the subvolumes](https://wiki.archlinux.org/title/Btrfs#Subvolumes) at the top level (`@snapshots` gets mounted later, in step 7, after Snapper's own setup is out of the way):
```
ROOT=$(findmnt -no SOURCE / | cut -d'[' -f1)
sudo mount -o subvolid=5 "$ROOT" /mnt
sudo btrfs subvolume create /mnt/@home_cache
sudo btrfs subvolume create /mnt/@home_containers
sudo btrfs subvolume create /mnt/@snapshots
sudo umount /mnt
```
3. Create the mountpoints (as your normal user, so the parent directories are yours). Both can start empty on a fresh install; clear the old cache so it isn't left hidden under the mount:
```
rm -rf ~/.cache/*
mkdir -p ~/.cache ~/.local/share/containers
```
4. [Add the fstab entries](https://wiki.archlinux.org/title/Fstab) — `sudo cp /etc/fstab /etc/fstab.bak`, then `sudoedit /etc/fstab`, copying your `/home` line's UUID and matching its exact option string:
```
UUID=<same>  /home/<user>/.cache                   btrfs  subvol=/@home_cache,noatime,compress=zstd       0 0
UUID=<same>  /home/<user>/.local/share/containers  btrfs  subvol=/@home_containers,noatime,compress=zstd  0 0
```
5. Test before rebooting — a bad fstab line can drop the next boot into emergency mode:
```
sudo findmnt --verify
sudo systemctl daemon-reload
sudo mount -a
sudo chown "$USER:$USER" ~/.cache ~/.local/share/containers   # new subvolume roots are root-owned
findmnt -t btrfs
```
6. Install the snapshot toolchain ([`sudo pacman -S yay`](https://wiki.archlinux.org/title/AUR_helpers) first if Base didn't include it):
```
yay -S snapper btrfs-assistant inotify-tools snap-pac grub-btrfs snap-pac-grub
```
7. [Create the snapper config](https://wiki.archlinux.org/title/Snapper#Creating_a_new_configuration) for `/`, then swap its default nested `.snapshots` for your separate top-level `@snapshots` subvolume — this is the [Arch Wiki's suggested layout](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout), specifically because it makes restoring `@` from a live USB later much simpler (no detaching/reattaching snapshot history):
```
sudo snapper -c root create-config /
sudo btrfs subvolume delete /.snapshots
sudo mkdir /.snapshots
```
   Add the fstab entry (same UUID and options as the rest):
```
UUID=<same>  /.snapshots  btrfs  subvol=/@snapshots,noatime,compress=zstd  0 0
```
```
sudo mount -a
sudo chmod 750 /.snapshots
sudo systemctl enable --now snapper-cleanup.timer
sudo systemctl enable --now grub-btrfsd
```
   Worth editing in `/etc/snapper/configs/root`: set `ALLOW_USERS="<your username>"` so `snapper` can run without `sudo`. `NUMBER_LIMIT`/`TIMELINE_*` default to fairly generous retention — tune down if 50+ pacman snapshots is more than you want kept.
8. [Set up zram](https://wiki.archlinux.org/title/Zram#Using_zram-generator) (no swap partition — zram covers it):
```
sudo pacman -S zram-generator
```
   Create `/etc/systemd/zram-generator.conf`:
```
[zram0]
zram-size = min(ram / 2, 8192)
compression-algorithm = zstd
```
```
sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service
zramctl   # confirm it's active
```

**Why these three extra subvolumes:**
- `@home_cache` → `/home/<user>/.cache` — a nested subvolume is invisible to a snapshot of its parent, so disposable cache data stays out of `@home` snapshots.
- `@home_containers` → `/home/<user>/.local/share/containers` — same nested-subvolume treatment, for rootless Podman's image/layer storage. Quadlet unit files (`~/.config/containers/systemd/*.container`) are plain config, not data — chezmoi manages those directly, no subvolume needed. On Docker instead, this would be a top-level `@docker` → `/var/lib/docker`, with Docker's own btrfs storage driver creating one subvolume per layer automatically underneath — not something to manage by hand.
- `@snapshots` → `/.snapshots` — kept as a sibling of `@` rather than nested inside it (Snapper's own default), so a snapshot swap during a restore never has to touch the snapshot history itself. See Troubleshooting for the actual restore procedure.

## Phase 6 — GRUB as the unified boot menu (EndeavorOS default, Windows second)

1. GRUB was installed as the primary bootloader in Phase 4 and should register itself as the default boot entry in firmware (NVRAM) automatically — verify by rebooting with no key pressed and confirming GRUB shows up, not Windows.
2. [Enable Windows detection](https://wiki.archlinux.org/title/GRUB#Detecting_other_operating_systems): `sudo pacman -S os-prober` (Arch's GRUB ships with it disabled), then edit `/etc/default/grub`, set `GRUB_DISABLE_OS_PROBER=false` (uncomment/change the line), and run `sudo grub-mkconfig -o /boot/grub/grub.cfg`. The output should include a "Found Windows Boot Manager on ..." line. EndeavorOS's own kernel entries are listed first (so they stay the default/highlighted choice), with "Windows Boot Manager" appended below as a selectable second option.

## Phase 7 — Re-enable Secure Boot with sbctl

Not required for dual-boot to work on its own, but this guide carries it through as a required step, on the reasoning that Secure Boot being on already (Phase 0) is worth keeping rather than turning off. Worth weighing the trade-offs first:

1. **Pros:** blocks unsigned/malicious code from hijacking the boot chain even if something gets root on either OS; it's already on per Phase 0, so keeping it is the no-extra-cost default; keeps both OSes symmetric on this setting.
2. **Cons:** every new kernel needs signing (sbctl's pacman hook usually handles it, but a missed sign means unbootable until fixed from a live USB); without LUKS its benefit here is narrow — it stops boot-chain attacks, not someone with physical disk access, which is already an accepted risk; adds one more thing that can go wrong in a plan otherwise built around minimizing failure points. GRUB without shim is also the fiddliest bootloader/Secure Boot combination (step 6), and once you own the keys, Microsoft's future Secure Boot certificate updates may no longer apply automatically through Windows Update.
3. **Live USB boot behavior after this:** Windows install/repair media and most mainstream Linux live USBs (Ubuntu, Fedora, etc.) still boot fine, since `enroll-keys -m` trusts Microsoft's signing keys too. Your own EndeavorOS Ventoy image won't — Arch-based live media generally isn't signed for Secure Boot — so it'd need Secure Boot temporarily disabled in firmware to use, then re-enabled after (see Troubleshooting).
4. `sudo pacman -S sbctl`, then `sbctl status` (expect Setup Mode disabled for now).
5. Only continue if Phase 2's Windows UEFI CA 2023 check returned `True`. Windows is moving its boot manager to that certificate during 2026; if your new key set lacks it, Windows may stop booting under Secure Boot after that update.
6. [Reinstall GRUB for Secure Boot without shim](https://wiki.archlinux.org/title/GRUB#Secure_Boot) (the "CA Keys" method), then regenerate the config:
```
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=EndeavourOS --modules="tpm" --disable-shim-lock
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
   Under Secure Boot GRUB can't load extra modules at runtime, so everything it needs must be built into that image. If the menu, the Btrfs root or the Windows chainload entry fails, add modules to `--modules` (a reasonable first try: `tpm chain fat part_gpt btrfs search search_fs_uuid`) and reinstall. If nothing works, turn Secure Boot off in firmware — Windows stays bootable via the firmware boot menu key either way.
7. In firmware, clear the Secure Boot keys / enter Setup Mode — a separate action from the OS Type switch, usually under a "Key Management" section (menu wording varies by vendor). Boot Linux with OS Type still on Other OS and confirm `sbctl status` now shows Setup Mode enabled.
8. [`sudo sbctl create-keys && sudo sbctl enroll-keys -m -f`](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#sbctl) — `-m` also enrolls Microsoft's keys so Windows keeps booting; `-f` keeps your firmware's built-in vendor keys (GPU option ROMs, and any 2023 Microsoft certificates your BIOS ships).
9. [Sign GRUB and the kernel](https://github.com/Foxboron/sbctl#usage):
```
sudo sbctl sign -s /boot/efi/EFI/EndeavourOS/grubx64.efi
sudo sbctl sign -s /boot/vmlinuz-linux
sudo sbctl verify
```
   If anything else shows up unsigned (an old fallback `EFI/BOOT/BOOTX64.EFI`, `vmlinuz-linux-lts` if you install it, etc.), catch it in one shot rather than hunting paths by hand: `sudo sbctl verify | sed 's/✗ /sbctl sign -s /e'`.
10. Reboot and check `sbctl status` first — on many boards, `enroll-keys` already exits Setup Mode and enables Secure Boot enforcement from the OS side by itself, no firmware toggle needed (this varies enough by board that some laptops don't even expose a manual "enable" option because of it). Only if it still shows disabled, switch **Boot → Secure Boot → OS Type** back to **Windows UEFI mode**. Either way, boot both OSes to confirm, and `journalctl -kg 'secure boot'` after booting EndeavorOS is a second way to confirm the kernel itself sees it as active.

## Phase 8 — Hand off to chezmoi

- The only prerequisites chezmoi needs are a base EndeavorOS install and a network connection — everything else (Hyprland, apps, dotfiles) is intentionally out of scope here.
- `sudo pacman -S git chezmoi`, then run your usual [chezmoi init/apply](https://www.chezmoi.io/install/).

## Troubleshooting (post-install reference)

- **Boots straight to Windows, GRUB menu skipped:** a major Windows feature update can reset the firmware's boot order back to "Windows Boot Manager" first. Boot into EndeavorOS via the firmware's one-time boot menu key (F8/F11/F12/Esc), then fix the order:
```
sudo efibootmgr
sudo efibootmgr -o <grub-entry-number>,<other-entries...>
```
  ([efibootmgr usage / boot order](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface#efibootmgr))
- **Windows entry missing from the GRUB menu:** re-run `sudo grub-mkconfig -o /boot/grub/grub.cfg` — can happen after Windows recreates its own ESP files during an update.
- **GRUB itself won't boot (especially right after Phase 7):** disable Secure Boot in firmware first; the firmware's one-time boot menu key also still works to pick either OS directly as a fallback. Keep the EndeavorOS install USB around too — it doubles as a rescue/chroot disk if GRUB needs reinstalling.
- **Installing/repairing Windows from the Ventoy stick:** setup reboots partway through. If you don't remove the USB (or manually pick the disk's own Windows Boot Manager entry) at that reboot, it can loop back into the Ventoy menu instead of continuing.
- **New kernel won't boot under Secure Boot:** new kernels need signing — [EndeavorOS's sbctl pacman hook](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Signing_the_kernel_with_a_pacman_hook) usually handles this automatically, but after a major kernel update check with `sudo sbctl verify` and sign anything unsigned with `sudo sbctl sign -s <path>`.
- **Rolling back after a bad update/install:** booting a `grub-btrfs` snapshot entry isn't a separate live environment — it's your real kernel/OS with an old `@` as root, while `@home`/`@cache`/`@log`/`@home_containers`/etc. keep mounting normally, untouched (and thanks to `@snapshots` being a sibling of `@`, so does your snapshot history). That's a way to *check* the old state works, and to grab a single old file (or just read it from `/.snapshots/<N>/snapshot/...` while running normally). To make it permanent: plain `snapper rollback` won't take effect on its own here — it works by changing btrfs's anonymous default subvolume, which has no effect since fstab pins `subvol=/@` explicitly. Instead use the [`snapper-rollback`](https://aur.archlinux.org/packages/snapper-rollback/) AUR tool (configure `/etc/snapper-rollback.conf` for your layout), or do it [by hand per the Arch Wiki](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout): boot a live USB, mount the top-level subvolume (no `subvol=`/`subvolid=` options), move the broken `@` aside (or delete it), create a writable snapshot of the good one from `@snapshots/<N>/snapshot` named `@`, then mount `@` and the ESP and `arch-chroot` in just long enough to regenerate the initramfs before rebooting. Since your fstab has no `subvolid=`, none of this needs fstab edits afterward.
- **EndeavorOS rescue USB won't boot:** `sbctl enroll-keys -m` trusts Windows and any Microsoft-shim-signed Linux (Ubuntu, Fedora, etc.), but not unsigned Arch-based live media — including your own EndeavorOS Ventoy image. Temporarily disable Secure Boot in firmware to use it, then re-enable after.

## Safety notes

- Because the Samsung drive is never repartitioned, the worst-case Linux mistake is limited to the WD drive — Windows stays bootable throughout the whole process.
- Leave Windows' own ESP on the Samsung drive alone entirely — don't mount, edit, or delete anything on it.

## References

- [r/archlinux secure boot thread](https://www.reddit.com/r/archlinux/comments/13d7rec/setting_up_secure_boot_while_dual_booting_windows/) → Phase 7
- [ZachHilman Btrfs + LUKS2 + Secure Boot guide](https://wiki.archlinux.org/title/User:ZachHilman/Installation_-_Btrfs_%2B_LUKS2_%2B_Secure_Boot) / [vortexavalanche gist](https://gist.github.com/vortexavalanche/64b3a7b97b3f163e252c49d6f82e6151) (2022) → background for Phase 5's subvolume work; skip their LUKS sections since we're going without encryption
- [mihirchanduka gist](https://gist.github.com/mihirchanduka/a9ba1c6edbfa068d2fbc2acb614c80e8) (2023) → subvolume reference for Phase 5

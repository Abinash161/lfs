# LFS 13.0 (Systemd) Build Notes
> Personal build log — LFS 13.0 stable systemd version  
> Kernel 7.0 | Intel integrated GPU | XFS filesystem | Hyprland on Wayland

---

## System Specs
- **CPU/GPU:** Intel (integrated graphics)
- **Kernel:** Linux 7.0
- **Init system:** Systemd
- **Filesystem:** XFS (root + separate home) + EFI FAT32
- **Display server:** Wayland (Hyprland)
- **Login:** TTY (no display manager)

---

## Partition Layout
```
/boot/efi   → FAT32 (EFI)
/           → XFS
/home       → XFS (separate partition)
```

### Why XFS
- Better performance than ext4 for this use case
- Kernel 7.0 introduced **online autorepair** for XFS — filesystem can repair itself without unmounting
- Additional I/O improvements in 7.0 kernel made it worthwhile over btrfs

---

## Build Order Overview

### Phase 1 — Base LFS
Follow LFS 13.0 book strictly. No deviations here.

### Phase 2 — BLFS / Network First
After base LFS, first priority is getting network up since everything else depends on it.

**Initial network setup (in host kernel environment):**
```bash
# Used wpa_supplicant temporarily to get network going
# during the BLFS compilation phase
```
> **Note:** wpa_supplicant was removed later and replaced with NetworkManager once the full system was stable.

### Phase 3 — Core BLFS packages
In rough order:
1. PAM
2. Shadow (with PAM support — critical, see gotcha below)
3. Systemd recompile (critical, see gotcha below)
4. Wayland + dependencies
5. Wayland protocols
6. Xwayland
7. Pipewire
8. Fonts (basics)
9. NetworkManager (replaced wpa_supplicant)

### Phase 4 — Wayland Desktop
Following **SLFS (Supplemental Linux From Scratch)** book for Hyprland:
- Hyprland compiled from source per SLFS guide
- Waybar
- Wofi
- Foot terminal
- Swaync
- Xwayland (worked without issues)

---

## Critical Gotchas

### ⚠️ PAM → Shadow → Systemd Recompile Order
**This is the most important thing in this entire document.**
Your compiled Shadow without PAM support in LFS will not work since Systemd will not pick it up correctly and authentication will silently fail or behave unexpectedly.

**Correct order — do not deviate:**
```
1. Compile PAM
2. Recompile Shadow WITH PAM support enabled
3. Recompile Systemd after both are in place
```

If Hyprland or your login is failing for no obvious reason after a complete build, this is almost certainly why. Recompile in this order and it will fix it.

---

### ⚠️ iwlwifi Network Firmware (Intel WiFi)
**Problem:** iwlwifi firmware not present in new system on first boot.

**Step 1 — Identify which ucode your card needs:**
```bash
dmesg | grep iwl
```
This tells you the exact firmware file being requested. Example output:
```
iwlwifi-7265D-29.ucode
```
Do not guess — the filename must match exactly.

**Step 2 — Get the firmware:**
Copy from your host system:
```bash
ls /lib/firmware/iwlwifi-*
# copy the relevant .ucode file to $LFS/lib/firmware/
```
Or download from [linux-firmware](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git)

**Step 3 — Build firmware into kernel via menuconfig:**
```
Device Drivers
  → Generic Driver Options
    → Firmware loader base path: /lib/firmware
    → Build named firmware blobs into the kernel binary:
       iwlwifi-7265D-29.ucode
```
Space-separate multiple files if needed. Then reconfigure and recompile kernel.

**Step 4 — Kernel config flags:**
```
CONFIG_IWLWIFI=y
CONFIG_IWLDVM=y   # or CONFIG_IWLMVM depending on your card
```

> **Why builtin over module:** As a module it requires initramfs to load firmware early enough. As builtin (`=y`) the driver is always available without initramfs complexity. Simpler and more reliable on LFS.

---

## Kernel Configuration

**Approach:** Started with a hardware-specific base config, then used `make menuconfig` for manual adjustments.

**Key areas configured manually:**
- iwlwifi builtin (see above)
- Intel integrated graphics (i915)
- XFS filesystem support + online autorepair (kernel 7.0 feature)
- Wayland/DRM requirements for Hyprland
- Binder/binderfs (for future Waydroid)

**XFS autorepair kernel flag (7.0+):**
```
CONFIG_XFS_ONLINE_SCRUB=y
CONFIG_XFS_ONLINE_REPAIR=y
```

**Intel GPU:**
```
CONFIG_DRM_I915=y
CONFIG_DRM=y
CONFIG_FB=y
```

**Wayland/Hyprland minimum requirements:**
```
CONFIG_DRM=y
CONFIG_DRM_KMS_HELPER=y
CONFIG_PACKET=y
CONFIG_INPUT_EVDEV=y
CONFIG_INPUT_UINPUT=y
```

---

## Why the First Build Was Scrapped

First build accumulated too many fixes over a week without documentation. System worked but the mental model of what was installed and why was lost. Rather than maintain a system that couldn't be reasoned about, it was rebuilt from scratch.

**Lesson:** On LFS, an undocumented working system is already broken — it just hasn't failed yet.

Second build was faster, cleaner, and every fix was understood.

---

## Current Userspace Stack

| Purpose | Tool |
|---|---|
| Compositor | Hyprland |
| Bar | Waybar |
| Launcher | Wofi |
| Notifications | Swaync |
| Terminal | Foot |
| Network | NetworkManager |
| Audio | Pipewire |
| Login | TTY (no DM) |
| Filesystem | XFS |

---

## Things Still TODO
- [ ] Waydroid setup (kernel binder config, ARM translation via libndk)
- [ ] Document exact Hyprland SLFS build steps
- [ ] Kernel config file export
- [ ] Exact BLFS package list with versions

---

## Notes
- This document is incomplete by nature — built from memory after the fact
- Gaps will be filled as things are encountered again
- SLFS book was used for Hyprland specific compilation steps

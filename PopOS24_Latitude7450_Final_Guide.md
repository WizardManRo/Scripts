# Dell Latitude 7450 — Pop!_OS 24.04 LTS
## Complete Setup Guide — Slim, Optimized, Production Ready

> **Hardware:** Intel Core Ultra 7 165H · 32GB RAM · 1TB SK Hynix BC901 NVMe  
> **GPU:** Intel Arc Graphics (Meteor Lake-P) · **Wi-Fi:** Intel BE20 (Wi-Fi 7)  
> **Serial:** H3LC044

---

## ⚠️ Critical Notes Before You Start

| Topic | Reality |
|---|---|
| Secure Boot | **Must be OFF** — Pop!_OS 24.04 does not support Secure Boot |
| TPM Auto-Unlock | Works, but uses PCR 0+1 instead of PCR 7 (no Secure Boot = no PCR7) |
| Hardware Encryption | **Do not use OPAL** on SK Hynix BC901 — known bypass vulnerabilities |
| Fedora alternative | Fedora 43 has Secure Boot + PCR7 TPM but Citrix is painful |

---

## Quick Reference — Execution Order

| Priority | Step | When |
|---|---|---|
| 🔴 | BIOS configuration | Before installing |
| 🔴 | Install Pop!_OS with LUKS2 encryption | During install |
| 🔴 | System update + kernel verification | First boot |
| 🔴 | Dell firmware updates via fwupdmgr | First boot |
| 🔴 | TPM2 auto-unlock (systemd-cryptenroll) | First boot |
| 🔴 | Webcam fix — IPU6 PPA | First boot |
| 🟠 | Intel Arc iGPU tuning | Same day |
| 🟠 | Battery optimization — TLP + s2idle | Same day |
| 🟠 | SOF audio firmware | Same day |
| 🟡 | Desktop — COSMIC or KDE | Settling in |
| 🟡 | Slim install — remove bloat | Settling in |
| 🟡 | OneDrive sync | Settling in |
| 🟡 | Dev and sysadmin tools | Settling in |
| 🟢 | Citrix Workspace | When needed |
| 🟢 | Fingerprint enrollment | After fwupdmgr |
| 🟢 | Thunderbolt dock | When using dock |

---

## Phase 0 — BIOS Configuration

Boot into Dell BIOS with **F2** at startup.

| Setting | Value | Why |
|---|---|---|
| Security → TPM | TPM 2.0 — **Enabled** | Required for TPM2 auto-unlock |
| Secure Boot | **Disabled** | Pop!_OS 24.04 does not support Secure Boot |
| Storage → NVMe Mode | AHCI/NVMe — **not RAID** | RAID mode hides the drive from the installer |
| Block Sleep | **ON** | Forces Modern Standby — Meteor Lake does not support S3 |
| Fast Boot | **Disabled** | Interferes with Linux boot |
| Intel VT-x / VT-d | **Enabled** | Required for KVM/virt-manager |
| Boot Sequence | USB first (temporary) | Revert to NVMe after install |

> **Block Sleep ON is critical.** It disables S3 legacy suspend and forces Modern Standby (s2idle). Without it, the laptop will drain battery even with the lid closed and suspend/resume will be unreliable on Meteor Lake.

---

## Phase 1 — Installation

### 1.1 — Download

Get the **Intel/AMD ISO** — not the NVIDIA one:

```
https://pop.system76.com
```

Write to USB:

```bash
sudo dd if=pop-os_24.04_amd64_intel_*.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

### 1.2 — Installer Steps

1. Boot from USB → **Clean Install**
2. On the partitioning screen → select **Encrypt Drive**
3. Set a **strong recovery passphrase** — store it in a password manager
4. Complete installation → reboot

> Your recovery passphrase is irreplaceable. If you lose it and TPM unlock fails, the data is unrecoverable. Store it somewhere safe.

### 1.3 — What the Installer Creates

| Partition | Size | Format | Notes |
|---|---|---|---|
| `/boot/efi` | 512 MB | FAT32 | Unencrypted — UEFI required |
| `/` root | Remaining | ext4 inside LUKS2 | Full disk encrypted |
| Recovery | ~4 GB | FAT32 | Pop!_OS recovery partition |

---

## Phase 2 — First Boot Setup

### 2.1 — System Update

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

### 2.2 — Verify Kernel

Pop!_OS 24.04 ships kernel 6.17. System76 pushes newer kernels through standard apt updates:

```bash
uname -r
# Expected: 6.17.x or newer

# Check available kernel versions
apt list --installed 2>/dev/null | grep linux-image
```

Always keep the system fully updated to get the latest kernel:

```bash
sudo apt full-upgrade -y
```

### 2.3 — Dell Firmware Updates

> Run this before any driver work. Dell regularly releases BIOS, NVMe, fingerprint, and peripheral firmware via LVFS that directly affects Linux compatibility.

```bash
sudo fwupdmgr refresh --force
sudo fwupdmgr get-updates
sudo fwupdmgr update
sudo reboot
```

### 2.4 — TPM2 Auto-Unlock

Binds the LUKS2 encrypted disk to the TPM2 chip. The disk unlocks automatically on boot. Because Secure Boot is off, PCR7 is unavailable — use PCR 0+1 instead, which ties unlock to the firmware and boot configuration.

**Verify TPM2 is visible:**

```bash
systemd-cryptenroll --tpm2-device=list
# Should show /dev/tpmrm0
```

**Find your LUKS partition:**

```bash
lsblk -o NAME,FSTYPE,UUID | grep crypto
# Note the device — typically nvme0n1p3
```

**Enroll TPM2:**

```bash
# PCR 0 = Core firmware executable code
# PCR 1 = Core firmware data/config
sudo systemd-cryptenroll \
  --tpm2-device=auto \
  --tpm2-pcrs=0+1 \
  /dev/nvme0n1p3
# Enter your LUKS passphrase when prompted
```

**Update crypttab:**

```bash
# View current crypttab
cat /etc/crypttab
# Example: dm_crypt-0  UUID=xxxx  none  luks,discard

# Edit — add tpm2-device=auto to options
sudo nano /etc/crypttab
# Result: dm_crypt-0  UUID=xxxx  none  luks,discard,tpm2-device=auto
```

**Rebuild initramfs:**

```bash
sudo update-initramfs -u -k all
sudo reboot
# Disk should unlock automatically — no passphrase prompt
```

**Verify enrollment after reboot:**

```bash
sudo systemd-cryptenroll /dev/nvme0n1p3
# Should show both passphrase slot and tpm2 slot
```

> Your original LUKS passphrase remains as an automatic fallback. If TPM unlock ever fails — after a BIOS update for example — just type the passphrase manually, then re-enroll the TPM.

---

## Phase 3 — Hardware Drivers

### 3.1 — Webcam (IPU6) — Most Critical Fix

> The Latitude 7450 webcam routes through Intel's IPU6 Image Processing Unit. It is not a standard UVC device and will not work out of the box on any distro without this fix.

```bash
# Add OEM Solutions PPA for IPU6
sudo add-apt-repository ppa:oem-solutions-group/intel-ipu6 -y
sudo apt update

sudo apt install -y \
  intel-ipu6-dkms \
  intel-ipu6-dkms-fbpt \
  gstreamer1.0-icamera \
  v4l2loopback-dkms

sudo reboot

# Verify after reboot
ls /dev/video*
v4l2-ctl --list-devices
```

### 3.2 — Intel Arc iGPU

```bash
# VA-API for hardware video acceleration
sudo apt install -y \
  intel-media-va-driver \
  libva-intel-driver \
  vainfo \
  intel-gpu-tools

# Enable GuC/HuC — required for proper GPU power management
echo 'options i915 enable_guc=3' | sudo tee /etc/modprobe.d/i915.conf

sudo update-initramfs -u -k all
sudo reboot

# Verify VA-API
vainfo
# Should list VAProfileH264, VAProfileHEVC etc
```

### 3.3 — Audio (SOF Firmware)

```bash
sudo apt install -y firmware-sof-signed alsa-ucm-conf

# Verify SOF is loading
dmesg | grep -i sof
aplay -l
arecord -l
```

> The internal microphone may be flaky on first boot. This typically resolves after a kernel update. If it persists, run `alsamixer` and check for muted channels.

### 3.4 — Wi-Fi 7 (Intel BE20)

Works out of the box on kernel 6.17+. If you see firmware errors in dmesg:

```bash
sudo apt install -y linux-firmware
```

Verify:

```bash
sudo dmesg | grep iwlwifi
iw dev wlp114s0f0 info
```

### 3.5 — Thunderbolt 4

```bash
sudo apt install -y bolt
sudo systemctl enable bolt --now

# List TB devices
boltctl list

# Authorize a dock permanently
boltctl enroll <uuid>
```

> Authorize docks once with `boltctl enroll` — subsequent connections auto-authorize. First hot-plug after suspend can be unreliable until authorized.

### 3.6 — Fingerprint Reader

```bash
# Firmware update is essential for Dell fingerprint sensors
sudo fwupdmgr update

sudo apt install -y fprintd libpam-fprintd

# Verify sensor is detected
fprintd-list $USER
lsusb | grep -i finger

# Enroll
fprintd-enroll

# Enable for sudo (optional)
sudo pam-auth-update
```

---

## Phase 4 — Battery Optimization

> This is the most impactful phase for daily usability. Untuned Linux on Meteor Lake can drain 2–3x faster than Windows. With these steps applied you should reach near-Windows battery life.

### 4.1 — s2idle Suspend Mode

With Block Sleep ON in BIOS, the system is already forced into Modern Standby. Match the kernel:

```bash
sudo kernelstub -a "mem_sleep_default=s2idle"

# Verify the parameter was added
sudo kernelstub -p

# Confirm after reboot
cat /sys/power/mem_sleep
# Expected: [s2idle] deep
```

### 4.2 — NVMe Latency Tweak (SK Hynix BC901)

```bash
# Check current power states
sudo apt install -y nvme-cli
sudo nvme get-feature /dev/nvme0 -f 0x0c -H

# Apply conservative latency value to allow deeper NVMe sleep
sudo kernelstub -a "nvme_core.default_ps_max_latency_us=5500"
```

### 4.3 — PCIe ASPM

```bash
sudo kernelstub -a "pcie_aspm=force"
```

### 4.4 — TLP with Meteor Lake Profile

```bash
sudo apt install -y tlp tlp-rdw

# Mask conflicting rfkill service
sudo systemctl mask systemd-rfkill.service systemd-rfkill.socket
sudo systemctl enable tlp --now
```

Create a dedicated config for the Latitude 7450:

```bash
sudo tee /etc/tlp.d/01-latitude7450.conf << 'EOF'
# CPU
CPU_SCALING_GOVERNOR_ON_AC=performance
CPU_SCALING_GOVERNOR_ON_BAT=powersave
CPU_ENERGY_PERF_POLICY_ON_AC=performance
CPU_ENERGY_PERF_POLICY_ON_BAT=power
CPU_MIN_PERF_ON_AC=0
CPU_MAX_PERF_ON_AC=100
CPU_MIN_PERF_ON_BAT=0
CPU_MAX_PERF_ON_BAT=30

# PCIe ASPM — most impactful setting for Meteor Lake idle drain
PCIE_ASPM_ON_AC=default
PCIE_ASPM_ON_BAT=powersupersave

# Runtime power management
RUNTIME_PM_ON_AC=auto
RUNTIME_PM_ON_BAT=auto

# Wi-Fi power saving
WIFI_PWR_ON_AC=off
WIFI_PWR_ON_BAT=on

# Wake on LAN
WOL_DISABLE=Y

# USB autosuspend
USB_AUTOSUSPEND=1
EOF
```

```bash
sudo tlp start

# Verify TLP is active and config loaded
sudo tlp-stat -s
```

### 4.5 — Powertop Persistent Auto-Tune

```bash
sudo apt install -y powertop

# Create systemd service for persistent auto-tune on boot
sudo tee /etc/systemd/system/powertop.service << 'EOF'
[Unit]
Description=PowerTop auto-tune
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/powertop --auto-tune
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable powertop.service --now
```

### 4.6 — Reboot and Verify Power Draw

```bash
sudo reboot

# After reboot, measure idle power consumption
# Let it settle for 2 minutes first
sudo powertop --time=60

# Target: under 5W at idle with screen on, under 2W with screen off
```

### Expected Battery Life After Tuning

| Scenario | Windows | Untuned Linux | Tuned Linux |
|---|---|---|---|
| Idle / light browsing | 7–9h | 2–3h | 6–8h |
| Lid closed / suspend | Near zero | High drain | Near zero |
| Video playback | 6–8h | 3–4h | 5–7h |
| Active workload | 3–4h | 2–3h | 3–4h |

---

## Phase 5 — Slim Install (Remove Bloat)

Pop!_OS 24.04 ships with COSMIC and various apps you may not need. Remove what doesn't serve you:

### 5.1 — Remove Unused COSMIC Apps (if switching to KDE)

```bash
sudo apt remove --purge -y \
  cosmic-files \
  cosmic-store \
  cosmic-screenshot \
  cosmic-media-player \
  cosmic-text-editor \
  gedit \
  totem \
  cheese \
  gnome-calculator \
  gnome-characters \
  simple-scan

sudo apt autoremove --purge -y
```

### 5.2 — Disable Unused Services

```bash
# Disable cups (printing) if not needed
sudo systemctl disable cups --now

# Disable avahi (network discovery) if not needed
sudo systemctl disable avahi-daemon --now

# Check what's running and consuming resources
systemctl list-units --type=service --state=running
```

### 5.3 — Swap (zram is better than swap partition)

Pop!_OS 24.04 uses zram by default. Verify it's active:

```bash
zramctl
# Should show /dev/zram0 with a size

# Check swap
swapon --show
```

If you want to tune zram compression for performance:

```bash
# lz4 is faster, zstd is better compression
echo 'ZRAM_ALGORITHM=lz4' | sudo tee /etc/default/zram-config
```

---

## Phase 6 — Desktop Setup

### Option A — Keep COSMIC (Default)

COSMIC is Wayland-native, fast, and well-integrated with Pop!_OS 24.04. Recommended if you don't have a strong preference. Rough edges exist in Epoch 1 but it's usable for daily work.

**Useful COSMIC settings to configure:**

```
System Settings → Power → set screen dim and sleep timeouts
System Settings → Display → set scaling (125% for FHD, 150% for QHD)
System Settings → Touchpad → configure gestures
System Settings → Keyboard → set shortcuts
```

### Option B — Install KDE Plasma 6

```bash
sudo apt install -y \
  kde-standard \
  sddm \
  plasma-systemmonitor \
  yakuake \
  kdeconnect

# Set SDDM as default display manager
sudo dpkg-reconfigure sddm

sudo reboot
# Select Plasma (Wayland) at login screen
```

**KDE Wayland fix for Intel Arc:**

```bash
echo 'KWIN_DRM_USE_EGL_STREAMS=0' | sudo tee /etc/environment
```

**KDE Connect firewall:**

```bash
sudo ufw allow 1714:1764/tcp
sudo ufw allow 1714:1764/udp
sudo ufw reload
```

**Remove COSMIC after switching to KDE (optional):**

```bash
sudo apt remove --purge -y \
  cosmic-session \
  cosmic-applets \
  cosmic-comp \
  cosmic-launcher \
  cosmic-osd \
  cosmic-panel \
  cosmic-settings \
  cosmic-bg \
  cosmic-workspaces-epoch

sudo apt autoremove --purge -y
```

---

## Phase 7 — Applications & Tools

### 7.1 — Flatpak & Flathub

Flathub is enabled by default in COSMIC Store. For CLI:

```bash
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

### 7.2 — OneDrive Sync

Same abraunegg client as before — config files transfer directly from any previous install:

```bash
sudo apt install -y onedrive

# Migrate config from previous install if needed
# scp user@oldmachine:~/.config/onedrive/config ~/.config/onedrive/config
# scp user@oldmachine:~/.config/onedrive/sync_list ~/.config/onedrive/sync_list

# Initial authorization
onedrive

# Enable as user service
systemctl --user enable onedrive --now
systemctl --user status onedrive
```

### 7.3 — Developer & Sysadmin Tools

```bash
sudo apt install -y \
  htop btop \
  git curl wget \
  neovim \
  python3-pip python3-venv \
  tmux \
  nvme-cli smartmontools \
  nmap iperf3 \
  smbclient \
  openssh-server \
  virt-manager qemu-kvm libvirt-daemon-system \
  bridge-utils

# Enable SSH
sudo systemctl enable ssh --now

# Enable libvirt for VMs
sudo systemctl enable libvirtd --now
sudo usermod -aG libvirt $USER

# Verify KVM
egrep -c '(vmx|svm)' /proc/cpuinfo   # must be > 0
sudo virt-host-validate
```

---

## Phase 8 — Citrix Workspace

Pop!_OS 24.04 is Citrix's primary Linux target. Installation is straightforward on the Ubuntu base.

### 8.1 — Dependencies

```bash
sudo apt install -y \
  libwebkit2gtk-4.0 \
  libssl3 \
  libpng16-16 \
  libnss3 \
  libgtk2.0-0 \
  libxmu6 libxtst6 libxaw7 \
  libjpeg-turbo8 \
  libglib2.0-0 \
  x11-utils
```

### 8.2 — Install

```bash
# Download .deb from:
# https://www.citrix.com/downloads/workspace-app/linux/

sudo dpkg -i icaclient_*.deb
sudo apt install -f -y
```

### 8.3 — Fix SSL Certificates

The most common post-install issue — Citrix uses its own cert store:

```bash
sudo ln -s /etc/ssl/certs/ca-certificates.crt \
  /opt/Citrix/ICAClient/keystore/cacerts/ca-certificates.crt

sudo c_rehash /opt/Citrix/ICAClient/keystore/cacerts/
```

### 8.4 — Audio (PipeWire)

If sessions have no audio:

```bash
sudo apt install -y pipewire-alsa pipewire-audio-client-libraries
```

In Citrix session preferences → set audio quality to **Medium** (High often breaks under Linux).

### 8.5 — Wayland Note

Citrix runs via XWayland. Clipboard sync between Citrix session and local desktop can be unreliable. For Citrix-heavy workdays, log into an **X11 session** from the login screen for rock-solid clipboard behavior.

---

## Phase 9 — Hardware Status Summary

| Component | Status | Action |
|---|---|---|
| Webcam (IPU6) | ❌ Broken out of box | PPA `oem-solutions-group/intel-ipu6` |
| Audio / Mic (SOF) | ⚠️ Partial | `firmware-sof-signed` — may need tuning |
| Intel Arc iGPU | ✅ Works | `enable_guc=3` + `intel-media-va-driver` |
| Wi-Fi 7 (BE20) | ✅ Works | `linux-firmware` update if errors |
| Thunderbolt 4 | ⚠️ Partial | `bolt` — authorize devices once |
| Fingerprint | ⚠️ Needs FW | `fwupdmgr update` first |
| NVMe SK Hynix | ✅ Works | Latency tweak for power saving |
| Touchpad | ✅ Works | Nothing needed |
| Bluetooth | ✅ Works | Nothing needed |
| Suspend / Resume | ✅ With fix | Block Sleep ON + s2idle |
| Battery life | ✅ With fix | TLP + s2idle + enable_guc=3 + ASPM |
| TPM auto-unlock | ✅ Works | PCR 0+1 (no PCR7 — Secure Boot off) |
| Citrix Workspace | ✅ Works | SSL cert symlink after install |
| Secure Boot | ❌ Not supported | Pop!_OS limitation |
| NPU (Meteor Lake) | ❌ Not usable | No driver yet |

---

## Final Notes

**On Secure Boot and TPM security:**
Pop!_OS does not support Secure Boot. TPM auto-unlock works via PCR 0+1 which ties unlock to firmware configuration — it protects against moving the drive to another machine but does not protect against bootloader tampering the way PCR7 does on Fedora. This is a known and accepted limitation of Pop!_OS. If Secure Boot + PCR7 is a hard requirement for your security policy, use Fedora 43 instead and solve the Citrix problem with Distrobox.

**On kernel updates:**
System76 validates and pushes kernel updates through the standard `apt full-upgrade` path. No manual intervention needed. Run `sudo apt full-upgrade -y` weekly to stay current.

**On TPM re-enrollment:**
A Dell BIOS update can change PCR 0+1 measurements, causing TPM unlock to fail. If this happens after a firmware update, boot with your passphrase and re-run `systemd-cryptenroll` to re-bind.

**On battery life:**
The three steps with the most impact are, in order: `s2idle`, `enable_guc=3`, and `PCIE_ASPM_ON_BAT=powersupersave`. The others are incremental. If you only do three things from Phase 4, do those three.

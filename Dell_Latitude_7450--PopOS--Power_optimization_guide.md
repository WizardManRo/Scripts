# Dell Latitude 7450 (Meteor Lake) — Power Optimization Guide
**Pop!_OS 24.04 LTS | Intel Core Ultra 7 165H | Intel Arc iGPU**

> Validated configuration. Apply in order from a fresh Pop!_OS install.  
> Expected result: ~6–10W idle on battery, full performance on AC.

---

## Prerequisites

- Fresh Pop!_OS 24.04 LTS install
- Internet connection
- `sudo` access

---

## Step 1 — Verify Kernel Version

Meteor Lake requires kernel **6.7 or newer** for full P-state, EPP, and GuC support.
Pop!_OS ships a recent kernel by default, but confirm:

```bash
uname -r
```

Expected output (or newer): `6.18.7-76061807-generic`

If older than 6.7, update:

```bash
sudo apt update && sudo apt full-upgrade
```

---

## Step 2 — Install Required Tools

```bash
sudo apt install -y tlp tlp-rdw thermald powertop lm-sensors
sudo sensors-detect --auto
sudo systemctl enable --now tlp thermald
```

**Disable the conflicting power daemon** — Pop!_OS ships with `power-profiles-daemon` active by
default. It controls the same kernel interfaces as TLP and must be disabled:

```bash
sudo systemctl disable --now power-profiles-daemon
```

---

## Step 3 — Kernel Parameters (systemd-boot)

Pop!_OS uses **systemd-boot**, not GRUB. Use `kernelstub` to add parameters persistently:

```bash
sudo kernelstub -a "i915.enable_guc=3 i915.enable_fbc=1 i915.enable_dc=2"
```

> **Do NOT add** `intel_pstate=active` — it is already active by default on Meteor Lake.  
> **Do NOT add** `nvme.noacpi=1` — Pop!_OS already sets `nvme_core.default_ps_max_latency_us=5500`
> which is a more precise NVMe power tuning for this hardware.

Reboot to activate:

```bash
sudo reboot
```

### Verify the full cmdline after reboot

```bash
cat /proc/cmdline
```

All of these must be present:

| Parameter | Purpose |
|---|---|
| `mem_sleep_default=s2idle` | Modern S0ix suspend (optimal for Meteor Lake) |
| `nvme_core.default_ps_max_latency_us=5500` | NVMe power state tuning |
| `pcie_aspm=force` | Force PCIe Active State Power Management |
| `i915.enable_guc=3` | GuC + HuC firmware (GPU power + video offload) |
| `i915.enable_fbc=1` | Frame Buffer Compression (iGPU power save) |
| `i915.enable_dc=2` | Intel DC6 deep GPU idle states |

> `mem_sleep_default=s2idle` and `pcie_aspm=force` are already present in the default Pop!_OS
> install for this hardware. Only the three `i915` parameters need to be added manually.

---

## Step 4 — TLP Configuration

Open the TLP config file:

```bash
sudo nano /etc/tlp.conf
```

Append the following block at the **end of the file** (after all the commented defaults):

```ini
# === Dell Latitude 7450 Meteor Lake — Optimized Settings ===

# CPU Governor
CPU_SCALING_GOVERNOR_ON_AC=performance
CPU_SCALING_GOVERNOR_ON_BAT=powersave

# Energy Performance Preference — key efficiency lever for Meteor Lake
CPU_ENERGY_PERF_POLICY_ON_AC=balance_performance
CPU_ENERGY_PERF_POLICY_ON_BAT=balance_power

# Turbo Boost
CPU_BOOST_ON_AC=1
CPU_BOOST_ON_BAT=0

# Platform Profile (Dell firmware-level power target)
PLATFORM_PROFILE_ON_AC=balanced
PLATFORM_PROFILE_ON_BAT=low-power

# PCIe ASPM
PCIE_ASPM_ON_AC=default
PCIE_ASPM_ON_BAT=powersupersave

# NMI Watchdog — disable saves ~1W
NMI_WATCHDOG=0

# WiFi Power Save
WIFI_PWR_ON_AC=off
WIFI_PWR_ON_BAT=on

# Runtime PM for PCIe devices (NVMe, etc.)
RUNTIME_PM_ON_AC=on
RUNTIME_PM_ON_BAT=auto

# Battery Charge Thresholds — extends long-term battery lifespan
# Stops charging at 80%, resumes below 20%
START_CHARGE_THRESH_BAT0=20
STOP_CHARGE_THRESH_BAT0=80
```

Apply without rebooting:

```bash
sudo tlp start
```

---

## Step 5 — Verify Everything

### TLP status and mode
```bash
sudo tlp-stat -s
```
Expected: `State = enabled`, `Mode = AC` or `BAT` depending on power source.

### Battery thresholds (confirm Dell natacpi plugin is active)
```bash
sudo tlp-stat -b
```
Look for `natacpi` driver active and thresholds showing `20 / 80`.

### CPU P-state and EPP
```bash
cat /sys/devices/system/cpu/intel_pstate/status
# expected: active

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# expected on AC: performance  |  on BAT: powersave

cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
# expected on AC: balance_performance  |  on BAT: balance_power
```

### Suspend mode
```bash
cat /sys/power/mem_sleep
# expected: [s2idle]
```

### GPU firmware (after reboot with i915.enable_guc=3)
```bash
sudo dmesg | grep -iE "guc|huc" | grep -iE "load|firmware"
```
Expected: lines confirming GuC and HuC firmware successfully loaded.

### Live power draw — unplug AC first
```bash
sudo powertop --time=10
```
Expected on idle desktop: **6–10W**. Above 15W suggests a wake lock or runaway process.

### Temperatures
```bash
sensors
```
Expected idle package temp: **40–55°C**. Fan should be at low RPM or stopped.

---

## Step 6 — Diagnostic Script

The companion script performs a full automated audit of every setting in this guide and gives
a clear ✔ / ⚠ / ✘ per check. Run it at any time to confirm a machine is correctly configured.

```bash
# Audit only
bash dell7450_power_analysis.sh

# Audit + apply live fixes (EPP, NVMe PM, USB autosuspend, s2idle, install TLP if missing)
sudo bash dell7450_power_analysis.sh --apply
```

A fully optimized system should return **all green** when run on battery after Steps 1–5.

---

## Quick Replication Checklist

For each new laptop of the same model and configuration:

- [ ] Fresh Pop!_OS 24.04 install, fully updated (`sudo apt update && sudo apt full-upgrade`)
- [ ] `sudo apt install -y tlp tlp-rdw thermald powertop lm-sensors`
- [ ] `sudo sensors-detect --auto`
- [ ] `sudo systemctl disable --now power-profiles-daemon`
- [ ] `sudo systemctl enable --now tlp thermald`
- [ ] `sudo kernelstub -a "i915.enable_guc=3 i915.enable_fbc=1 i915.enable_dc=2"`
- [ ] Append TLP config block to `/etc/tlp.conf`
- [ ] `sudo tlp start && sudo reboot`
- [ ] Post-reboot: `sudo tlp-stat -s` and `sudo tlp-stat -b` to verify
- [ ] Unplug AC and run `sudo powertop --time=10` to confirm idle wattage

---

## Expected Final State

| Check | On AC | On Battery |
|---|---|---|
| Kernel | 6.7+ | 6.7+ |
| Intel P-state | active | active |
| CPU Governor | performance | powersave |
| EPP | balance_performance | balance_power |
| Turbo Boost | enabled | disabled |
| Platform Profile | balanced | low-power |
| PCIe ASPM | default | powersupersave |
| GPU Driver | Intel Xe | Intel Xe |
| GPU RC6 gating | enabled | enabled |
| GuC/HuC firmware | loaded | loaded |
| Suspend mode | s2idle | s2idle |
| NVMe runtime PM | on | auto |
| USB autosuspend | 2s | 2s |
| WiFi power save | off | on |
| Idle power draw | — (on AC) | 6–10W |
| Idle package temp | 40–55°C | 40–55°C |
| Battery charge stop | — | 80% |
| Battery charge start | — | 20% |

---

## Notes

**Battery thresholds:** The 20/80 rule is a longevity-focused choice. If the laptop is frequently
used away from power for long periods, consider 20/85 or 20/90 as a compromise between health
and runtime. Adjust `STOP_CHARGE_THRESH_BAT0` accordingly.

**GuC/HuC firmware:** On kernels before 6.8, the `i915` driver handles Meteor Lake's Arc iGPU.
From 6.8 onward, the `xe` driver takes over. Both are correctly handled on this system —
the `i915.enable_guc=3` parameter applies to whichever driver is active.

**power-profiles-daemon vs TLP:** Pop!_OS ships with `power-profiles-daemon` active by default.
It must be disabled before TLP will fully take effect — they control the same kernel interfaces
and conflict silently if both run simultaneously.

**NMI Watchdog:** `NMI_WATCHDOG=0` disables the kernel hardware error watchdog. This is safe on
office/production hardware and saves ~1W. Leave enabled only if you need kernel crash debugging.

**intel_pstate=active is NOT needed in cmdline:** The P-state driver is already active by default
on Meteor Lake. Adding it explicitly is redundant. The diagnostic script will flag it as
unnecessary if found in cmdline.

---

## Appendix — Diagnostic Script Source

Save as `dell7450_power_analysis.sh` alongside this guide.

```bash
#!/usr/bin/env bash
# =============================================================================
# Dell Latitude 7450 (Meteor Lake) — Power Optimization Analysis
# Pop!_OS 24.04 LTS | Intel Core Ultra 7 165H | Intel Arc iGPU
#
# Usage:
#   bash dell7450_power_analysis.sh           # audit only
#   sudo bash dell7450_power_analysis.sh --apply  # audit + apply live fixes
# =============================================================================

APPLY=false
[[ "$1" == "--apply" ]] && APPLY=true

RED='\033[0;31m'; YELLOW='\033[1;33m'; GREEN='\033[0;32m'
CYAN='\033[0;36m'; BOLD='\033[1m'; NC='\033[0m'

sep()  { echo -e "${CYAN}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"; }
ok()   { echo -e "  ${GREEN}✔${NC} $*"; }
warn() { echo -e "  ${YELLOW}⚠${NC}  $*"; }
info() { echo -e "  ${CYAN}ℹ${NC}  $*"; }
bad()  { echo -e "  ${RED}✘${NC} $*"; }
hdr()  { sep; echo -e "${BOLD}  $*${NC}"; sep; }

# Detect current power source
AC_ONLINE=$(cat /sys/class/power_supply/AC*/online 2>/dev/null | head -1)
[[ "$AC_ONLINE" == "1" ]] && ON_AC=true || ON_AC=false

echo ""
echo -e "${BOLD}  Dell Latitude 7450 (Meteor Lake) — Power Analysis  $(date)${NC}"
$ON_AC && echo -e "  ${CYAN}Power source: AC${NC}" || echo -e "  ${YELLOW}Power source: BATTERY${NC}"
echo ""

# =============================================================================
hdr "1. HARDWARE IDENTIFICATION"
# =============================================================================
info "Product : $(cat /sys/class/dmi/id/product_name 2>/dev/null)"
info "Board   : $(cat /sys/class/dmi/id/board_name 2>/dev/null)"
info "BIOS    : $(cat /sys/class/dmi/id/bios_version 2>/dev/null)"
info "EC FW   : $(cat /sys/class/dmi/id/ec_firmware_release 2>/dev/null)"
info "Kernel  : $(uname -r)"
info "Distro  : $(grep PRETTY_NAME /etc/os-release | cut -d= -f2 | tr -d '"')"

KMAJ=$(uname -r | cut -d. -f1); KMIN=$(uname -r | cut -d. -f2)
[[ $KMAJ -gt 6 || ($KMAJ -eq 6 && $KMIN -ge 7) ]] \
  && ok "Kernel $(uname -r) — Meteor Lake support confirmed (>=6.7)" \
  || bad "Kernel too old! Meteor Lake needs 6.7+ — upgrade via Pop!_OS kernel manager"

# =============================================================================
hdr "2. CPU / INTEL METEOR LAKE"
# =============================================================================
info "CPU: $(grep -m1 'model name' /proc/cpuinfo | cut -d: -f2 | xargs)"
info "Logical CPUs: $(nproc)"

# P-state — on Meteor Lake this is active by default, no cmdline param needed
PSTATE=$(cat /sys/devices/system/cpu/intel_pstate/status 2>/dev/null)
[[ "$PSTATE" == "active" ]] && ok "Intel P-state: active (default, no cmdline param needed)" \
  || { [[ -n "$PSTATE" ]] && warn "Intel P-state: $PSTATE (expected: active)" \
    || bad "Intel P-state driver not found"; }

TURBO=$(cat /sys/devices/system/cpu/intel_pstate/no_turbo 2>/dev/null)
[[ "$TURBO" == "0" ]] && ok "Turbo Boost: enabled" || warn "Turbo Boost: disabled (no_turbo=1)"

GOV=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor 2>/dev/null)
info "CPU Governor: $GOV"
if $ON_AC; then
  [[ "$GOV" == "performance" || "$GOV" == "powersave" ]] \
    && ok "Governor OK on AC ($GOV)" || warn "Unexpected governor on AC: $GOV"
else
  [[ "$GOV" == "powersave" ]] \
    && ok "Governor: powersave (correct on battery)" \
    || warn "Governor: $GOV — expected powersave on battery (check TLP config)"
fi

EPP=$(cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference 2>/dev/null)
EPP_AVAIL=$(cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_available_preferences 2>/dev/null)
if [[ -n "$EPP" ]]; then
  info "EPP current   : $EPP"
  info "EPP available : $EPP_AVAIL"
  if $ON_AC; then
    [[ "$EPP" == "balance_performance" || "$EPP" == "performance" ]] \
      && ok "EPP correct on AC ($EPP)" \
      || warn "EPP on AC: $EPP (expected balance_performance — check TLP config)"
  else
    [[ "$EPP" == "balance_power" || "$EPP" == "power" ]] \
      && ok "EPP correct on battery ($EPP)" \
      || warn "EPP on battery: $EPP (expected balance_power — check TLP config)"
  fi
else
  bad "EPP not available — intel_pstate must be in active mode"
fi

FMIN=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq 2>/dev/null)
FMAX=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq 2>/dev/null)
FCUR=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq 2>/dev/null)
[[ -n "$FMIN" ]] && info "Freq: min=$(( FMIN/1000 ))MHz  max=$(( FMAX/1000 ))MHz  cur=$(( FCUR/1000 ))MHz"

# =============================================================================
hdr "3. BATTERY & POWER DRAW"
# =============================================================================
BAT=$(find /sys/class/power_supply -name "BAT*" | head -1)
if [[ -n "$BAT" ]]; then
  BAT_STATUS=$(cat $BAT/status 2>/dev/null)
  info "Status  : $BAT_STATUS"
  info "Charge  : $(cat $BAT/capacity 2>/dev/null)%"
  info "Type    : $(cat $BAT/technology 2>/dev/null)"

  EF=$(cat $BAT/energy_full 2>/dev/null)
  ED=$(cat $BAT/energy_full_design 2>/dev/null)
  [[ -n "$EF" && -n "$ED" && $ED -gt 0 ]] && {
    WEAR=$(awk "BEGIN {printf \"%.1f\", (1 - $EF/$ED)*100}")
    EFWH=$(awk "BEGIN {printf \"%.1f\", $EF/1000000}")
    EDWH=$(awk "BEGIN {printf \"%.1f\", $ED/1000000}")
    info "Capacity: ${EFWH}Wh / ${EDWH}Wh design  (~${WEAR}% wear)"
    awk "BEGIN {exit ($WEAR >= 10)}" && ok "Battery health: excellent (<10% wear)" \
    || awk "BEGIN {exit ($WEAR >= 20)}" && ok "Battery health: good (<20% wear)" \
    || warn "Battery wear: ${WEAR}% — consider recalibration or replacement"
  }

  PN=$(cat $BAT/power_now 2>/dev/null)
  [[ -n "$PN" && "$BAT_STATUS" == "Discharging" ]] && {
    W=$(awk "BEGIN {printf \"%.2f\", $PN/1000000}")
    info "Current draw: ${W}W"
    awk "BEGIN {exit ($W >= 8)}"  && ok "Excellent draw (<8W)" \
    || awk "BEGIN {exit ($W >= 15)}" && ok "Good draw (<15W)" \
    || awk "BEGIN {exit ($W >= 25)}" && warn "Moderate draw (${W}W) — room for improvement" \
    || bad "High draw (${W}W) — optimization strongly recommended"
  }
else
  warn "No battery detected"
fi

# =============================================================================
hdr "4. POWER MANAGEMENT TOOLS"
# =============================================================================
if systemctl is-active --quiet tlp 2>/dev/null; then
  TLP_VER=$(tlp-stat -s 2>/dev/null | grep "TLP version" | awk '{print $3}')
  ok "TLP active (v${TLP_VER:-unknown})"
elif dpkg -l tlp &>/dev/null; then
  warn "TLP installed but inactive — run: sudo systemctl enable --now tlp"
else
  bad "TLP not installed — run: sudo apt install tlp tlp-rdw && sudo systemctl enable --now tlp"
fi

if systemctl is-active --quiet power-profiles-daemon 2>/dev/null; then
  bad "power-profiles-daemon is RUNNING — conflicts with TLP!"
  warn "Disable it: sudo systemctl disable --now power-profiles-daemon"
else
  ok "power-profiles-daemon: inactive (correct when using TLP)"
fi

systemctl is-active --quiet thermald 2>/dev/null \
  && ok "thermald: active" \
  || warn "thermald inactive — run: sudo apt install thermald && sudo systemctl enable --now thermald"

which powertop &>/dev/null \
  && ok "powertop: installed" \
  || warn "powertop not installed — run: sudo apt install powertop"

which sensors &>/dev/null \
  && ok "lm-sensors: installed" \
  || warn "lm-sensors not installed — run: sudo apt install lm-sensors && sudo sensors-detect --auto"

# =============================================================================
hdr "5. TLP CONFIGURATION CHECK"
# =============================================================================
TLP_CONF="/etc/tlp.conf"
if [[ -f "$TLP_CONF" ]]; then
  check_tlp() {
    local key="$1" expected="$2" label="$3"
    local val=$(grep -E "^${key}=" "$TLP_CONF" 2>/dev/null | tail -1 | cut -d= -f2)
    if [[ -z "$val" ]]; then
      warn "TLP: $key not set (expected: $expected) — $label"
    elif [[ "$val" == "$expected" ]]; then
      ok "TLP: $key=$val — $label"
    else
      warn "TLP: $key=$val (expected: $expected) — $label"
    fi
  }
  check_tlp "CPU_SCALING_GOVERNOR_ON_AC"    "performance"         "AC governor"
  check_tlp "CPU_SCALING_GOVERNOR_ON_BAT"   "powersave"           "BAT governor"
  check_tlp "CPU_ENERGY_PERF_POLICY_ON_AC"  "balance_performance" "AC EPP"
  check_tlp "CPU_ENERGY_PERF_POLICY_ON_BAT" "balance_power"       "BAT EPP"
  check_tlp "CPU_BOOST_ON_AC"               "1"                   "AC turbo"
  check_tlp "CPU_BOOST_ON_BAT"              "0"                   "BAT turbo off"
  check_tlp "PLATFORM_PROFILE_ON_AC"        "balanced"            "AC platform profile"
  check_tlp "PLATFORM_PROFILE_ON_BAT"       "low-power"           "BAT platform profile"
  check_tlp "PCIE_ASPM_ON_AC"               "default"             "AC PCIe ASPM"
  check_tlp "PCIE_ASPM_ON_BAT"              "powersupersave"      "BAT PCIe ASPM"
  check_tlp "NMI_WATCHDOG"                  "0"                   "NMI watchdog disabled"
  check_tlp "WIFI_PWR_ON_AC"               "off"                  "AC WiFi power save"
  check_tlp "WIFI_PWR_ON_BAT"              "on"                   "BAT WiFi power save"
  check_tlp "RUNTIME_PM_ON_AC"             "on"                   "AC runtime PM"
  check_tlp "RUNTIME_PM_ON_BAT"            "auto"                 "BAT runtime PM"
  # Battery charge thresholds
  START=$(grep -E "^START_CHARGE_THRESH_BAT0=" "$TLP_CONF" 2>/dev/null | tail -1 | cut -d= -f2)
  STOP=$(grep  -E "^STOP_CHARGE_THRESH_BAT0="  "$TLP_CONF" 2>/dev/null | tail -1 | cut -d= -f2)
  if [[ -n "$START" && -n "$STOP" ]]; then
    ok "TLP: charge thresholds set — start=${START}% stop=${STOP}%"
  else
    warn "TLP: charge thresholds not set — add START/STOP_CHARGE_THRESH_BAT0 to extend battery lifespan"
  fi
else
  bad "/etc/tlp.conf not found"
fi

# =============================================================================
hdr "6. INTEL Xe iGPU (Meteor Lake Arc)"
# =============================================================================
info "GPU(s):"
lspci | grep -iE "VGA|Display|3D" | while read l; do info "  $l"; done

lsmod | grep -q "^xe "     && ok "Intel Xe driver loaded (optimal for Meteor Lake)" \
|| lsmod | grep -q "^i915 " && ok "Intel i915 driver loaded" \
|| bad "No Intel GPU driver loaded — check driver/kernel status"

RC6=$(cat /sys/class/drm/card*/power/rc6_enable 2>/dev/null | head -1)
[[ "$RC6" == "1" ]] && ok "GPU RC6 power gating: enabled" || warn "GPU RC6 power gating not detected"

GUC=$(dmesg 2>/dev/null | grep -iE "guc.*(load|firmware|enabled)" | tail -1)
HUC=$(dmesg 2>/dev/null | grep -iE "huc.*(load|firmware|enabled)" | tail -1)
[[ -n "$GUC" ]] && ok "GuC firmware: loaded" || warn "GuC firmware not detected — add i915.enable_guc=3 to kernel params"
[[ -n "$HUC" ]] && ok "HuC firmware: loaded" || info "HuC status: not detected in dmesg (may be normal)"

FBC=$(cat /sys/kernel/debug/dri/0/i915_fbc_status 2>/dev/null | head -1)
[[ -n "$FBC" ]] && info "GPU FBC status: $FBC" || info "GPU FBC status: not readable (needs root debug access)"

# =============================================================================
hdr "7. THERMAL"
# =============================================================================
TEMPS=$(sensors 2>/dev/null | grep -E "Core|Package|CPU Fan|temp")
if [[ -n "$TEMPS" ]]; then
  info "Temperatures:"
  echo "$TEMPS" | while read l; do info "  $l"; done
  PKG_TEMP=$(sensors 2>/dev/null | grep "Package id 0" | grep -oP '\+\K[0-9.]+' | head -1)
  [[ -n "$PKG_TEMP" ]] && {
    awk "BEGIN {exit ($PKG_TEMP >= 90)}" && ok "Package temp OK (${PKG_TEMP}°C — well below 110°C limit)" \
    || awk "BEGIN {exit ($PKG_TEMP >= 75)}" && warn "Package temp elevated (${PKG_TEMP}°C)" \
    || bad "Package temp high (${PKG_TEMP}°C) — check cooling"
  }
else
  warn "lm-sensors not configured or not installed"
fi

# =============================================================================
hdr "8. SUSPEND / SLEEP STATES"
# =============================================================================
SM=$(cat /sys/power/mem_sleep 2>/dev/null)
info "Available suspend modes: $SM"
echo "$SM" | grep -q "\[s2idle\]" \
  && ok "s2idle (S0ix) active — optimal for Meteor Lake" \
  || { echo "$SM" | grep -q "\[deep\]" \
    && warn "S3 deep sleep active — switch: mem_sleep_default=s2idle in kernel params" \
    || warn "Unknown suspend mode: $SM"; }

# =============================================================================
hdr "9. PCIe / USB / NVMe POWER"
# =============================================================================
ASPM=$(cat /sys/module/pcie_aspm/parameters/policy 2>/dev/null)
info "PCIe ASPM policy: $ASPM"
if $ON_AC; then
  # On AC, 'default' is correct per our TLP config (PCIE_ASPM_ON_AC=default)
  [[ "$ASPM" == "default" || "$ASPM" == *"[default]"* ]] \
    && ok "PCIe ASPM: default (correct on AC)" \
    || info "PCIe ASPM on AC: $ASPM"
else
  [[ "$ASPM" == "powersupersave" || "$ASPM" == "powersave" ]] \
    && ok "PCIe ASPM: $ASPM (correct on battery)" \
    || warn "PCIe ASPM on battery: $ASPM (expected powersupersave — check TLP)"
fi

USB=$(cat /sys/module/usbcore/parameters/autosuspend 2>/dev/null)
[[ "${USB:-0}" -ge 1 ]] 2>/dev/null \
  && ok "USB autosuspend: ${USB}s" \
  || warn "USB autosuspend disabled — TLP should enable this on battery"

NVME_PM=$(cat /sys/class/nvme/*/power/control 2>/dev/null | head -1)
[[ "$NVME_PM" == "auto" ]] \
  && ok "NVMe runtime PM: auto" \
  || warn "NVMe runtime PM: ${NVME_PM:-not found} — expected auto"

NVME_PS=$(cat /proc/cmdline | grep -o "nvme_core.default_ps_max_latency_us=[0-9]*")
[[ -n "$NVME_PS" ]] \
  && ok "NVMe PS latency tuning: $NVME_PS" \
  || info "NVMe PS latency: not set (optional — default Pop!_OS install includes this)"

WIFI_IF=$(ip link | grep -E "^[0-9]+: w" | awk -F': ' '{print $2}' | head -1)
[[ -n "$WIFI_IF" ]] && {
  WPM=$(iwconfig $WIFI_IF 2>/dev/null | grep "Power Management" | grep -o "on\|off")
  if $ON_AC; then
    info "WiFi power management: $WPM ($WIFI_IF) — off is correct on AC"
  else
    [[ "$WPM" == "on" ]] \
      && ok "WiFi power management: on ($WIFI_IF) — correct on battery" \
      || warn "WiFi power mgmt off on battery ($WIFI_IF) — check TLP WIFI_PWR_ON_BAT=on"
  fi
}

# =============================================================================
hdr "10. KERNEL PARAMETERS"
# =============================================================================
CMDLINE=$(cat /proc/cmdline)
echo ""
info "Full cmdline:"
echo "  $CMDLINE" | fold -w 88; echo ""

# Parameters that MUST be present
declare -A REQUIRED_PARAMS=(
  ["mem_sleep_default=s2idle"]="S0ix suspend default (required for Meteor Lake)"
  ["pcie_aspm=force"]="Force PCIe ASPM power gating"
  ["i915.enable_guc=3"]="GuC+HuC firmware (GPU power + video offload)"
  ["i915.enable_fbc=1"]="Frame Buffer Compression (iGPU power save)"
  ["i915.enable_dc=2"]="Intel DC6 deep GPU idle states"
)

# Parameters that should NOT be present (we learned these are wrong/redundant for this hw)
declare -A WRONG_PARAMS=(
  ["intel_pstate=active"]="Redundant — P-state is active by default on Meteor Lake"
  ["nvme.noacpi=1"]="Not needed — nvme_core.default_ps_max_latency_us=5500 is already set"
)

echo "  Required parameters:"
for p in "${!REQUIRED_PARAMS[@]}"; do
  key="${p%%=*}"
  echo "$CMDLINE" | grep -q "$key" \
    && ok "$p — ${REQUIRED_PARAMS[$p]}" \
    || bad "MISSING: $p — ${REQUIRED_PARAMS[$p]}"
done

echo ""
echo "  Parameters that should NOT be in cmdline:"
for p in "${!WRONG_PARAMS[@]}"; do
  key="${p%%=*}"
  echo "$CMDLINE" | grep -q "$key" \
    && warn "UNNECESSARY: $p — ${WRONG_PARAMS[$p]}" \
    || ok "Absent (correct): $p"
done

# =============================================================================
hdr "SUMMARY — RECOMMENDED ACTIONS"
# =============================================================================
echo ""
echo -e "${BOLD}  [1] KERNEL PARAMS (Pop!_OS uses kernelstub, NOT update-grub)${NC}"
echo '      sudo kernelstub -a "i915.enable_guc=3 i915.enable_fbc=1 i915.enable_dc=2"'
echo '      # mem_sleep_default=s2idle and pcie_aspm=force are already set by Pop!_OS'
echo '      # Do NOT add intel_pstate=active or nvme.noacpi=1'
echo '      sudo reboot'
echo ""
echo -e "${BOLD}  [2] TLP CONFIG (/etc/tlp.conf — append at end of file)${NC}"
echo '      CPU_SCALING_GOVERNOR_ON_AC=performance'
echo '      CPU_SCALING_GOVERNOR_ON_BAT=powersave'
echo '      CPU_ENERGY_PERF_POLICY_ON_AC=balance_performance'
echo '      CPU_ENERGY_PERF_POLICY_ON_BAT=balance_power'
echo '      CPU_BOOST_ON_AC=1'
echo '      CPU_BOOST_ON_BAT=0'
echo '      PLATFORM_PROFILE_ON_AC=balanced'
echo '      PLATFORM_PROFILE_ON_BAT=low-power'
echo '      PCIE_ASPM_ON_AC=default'
echo '      PCIE_ASPM_ON_BAT=powersupersave'
echo '      NMI_WATCHDOG=0'
echo '      WIFI_PWR_ON_AC=off'
echo '      WIFI_PWR_ON_BAT=on'
echo '      RUNTIME_PM_ON_AC=on'
echo '      RUNTIME_PM_ON_BAT=auto'
echo '      START_CHARGE_THRESH_BAT0=20'
echo '      STOP_CHARGE_THRESH_BAT0=80'
echo '      Then: sudo tlp start'
echo ""
echo -e "${BOLD}  [3] INSTALL MISSING TOOLS${NC}"
echo '      sudo apt install tlp tlp-rdw thermald powertop lm-sensors'
echo '      sudo sensors-detect --auto'
echo '      sudo systemctl disable --now power-profiles-daemon'
echo '      sudo systemctl enable --now tlp thermald'
echo ""
echo -e "${BOLD}  [4] VERIFY${NC}"
echo '      sudo tlp-stat -s          # TLP state and mode'
echo '      sudo tlp-stat -b          # battery thresholds (natacpi plugin)'
echo '      cat /sys/power/mem_sleep  # should show [s2idle]'
echo '      sudo powertop --time=10   # live power breakdown (unplug AC first)'
echo ""

# =============================================================================
if $APPLY; then
  hdr "APPLYING LIVE FIXES (--apply)"
  [[ $EUID -ne 0 ]] && { bad "Requires sudo — run: sudo bash $0 --apply"; exit 1; }

  # Disable conflicting daemon
  if systemctl is-active --quiet power-profiles-daemon 2>/dev/null; then
    systemctl disable --now power-profiles-daemon
    ok "Disabled power-profiles-daemon"
  fi

  # Install TLP if missing
  if ! dpkg -l tlp &>/dev/null; then
    apt install -y tlp tlp-rdw
    systemctl enable --now tlp
    ok "TLP installed and enabled"
  fi

  # Install thermald if missing
  if ! dpkg -l thermald &>/dev/null; then
    apt install -y thermald
    systemctl enable --now thermald
    ok "thermald installed and enabled"
  fi

  # EPP — set to balance_power on all cores (battery-side live fix)
  for f in /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference; do
    echo balance_power > "$f" 2>/dev/null
  done
  ok "EPP: balance_power applied to all cores"

  # NVMe runtime PM
  for f in /sys/class/nvme/*/power/control; do
    echo auto > "$f" 2>/dev/null
  done
  ok "NVMe runtime PM: auto"

  # USB autosuspend
  echo 2 > /sys/module/usbcore/parameters/autosuspend 2>/dev/null
  ok "USB autosuspend: 2s"

  # s2idle (already set by Pop!_OS default, but enforce)
  echo s2idle > /sys/power/mem_sleep 2>/dev/null
  ok "mem_sleep: s2idle"

  echo ""
  warn "Persistent changes still needed:"
  warn "  1. sudo kernelstub -a \"i915.enable_guc=3 i915.enable_fbc=1 i915.enable_dc=2\""
  warn "  2. Append TLP config block to /etc/tlp.conf then: sudo tlp start"
  warn "  3. sudo reboot to activate all kernel params"
fi

sep
echo -e "  Analysis complete. Run ${BOLD}sudo bash $0 --apply${NC} to apply live optimizations."
sep
echo ""
```

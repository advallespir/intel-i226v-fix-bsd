# Intel I226-V NIC Fix for FreeBSD / OPNsense / pfSense

> Fix random disconnections, link flapping, and EEE-related crashes on Intel I226-V 2.5GbE controllers running FreeBSD-based firewalls — **without Windows**.

## TL;DR

```bash
# 1. Add tunables in OPNsense GUI (System → Settings → Tunables):
#    dev.igc.X.eee_control = 0  (for each NIC)
#    dev.igc.X.fc = 0           (for each NIC)
#    hw.pci.do_power_nodriver = 3
#    Reboot.

# 2. Update NVM firmware via SSH:
scp nvmupdate64e FXVL_125C_V_2MB_2.32.bin nvmupdate_2mb.cfg root@<router-ip>:/tmp/
ssh root@<router-ip>
# Shell (option 8)
cd /tmp && chmod +x nvmupdate64e
./nvmupdate64e -u -l -o update.xml -b -c nvmupdate_2mb.cfg
shutdown -p now
# Unplug power 10 seconds, plug back in.
```

---

## The Problem

The Intel I225-V and I226-V 2.5GbE Ethernet controllers have a widely reported firmware/hardware bug. Under light traffic, the NIC enters a low-power state via Energy Efficient Ethernet (EEE). When it tries to wake up, the link renegotiation fails, causing:

- Complete loss of traffic (NIC appears up but passes no data)
- Rapid link flapping (carrier lost/gained every few seconds)
- Devices behind the router lose internet until power-cycled
- Random disconnections lasting seconds to minutes

This is especially common on:
- Mini-PCs from CWWK, Topton, Protectli, and similar brands
- OPNsense, pfSense, or any FreeBSD-based firewall
- Firmware versions below 2.22

Intel fixed this at the firmware level starting with NVM version 2.22, but most devices ship with V2.13 or V2.14 and **Intel does not provide the update tool publicly for i226-V**.

## Why This Repo Exists

All existing guides focus on Windows. This repo provides everything needed to fix the issue directly from **FreeBSD/OPNsense via SSH**, without booting Windows or FreeDOS:

- Pre-configured `.cfg` files for the Intel NVM Update Tool
- Firmware binary links
- Complete step-by-step for FreeBSD/OPNsense
- Documented gotchas (1MB vs 2MB flash, config format, etc.)

---

## Part 1: Software Fix (Tunables)

This provides immediate relief. No firmware flashing needed.

### OPNsense GUI: System → Settings → Tunables

Add these tunables and **reboot** (do NOT apply in hot mode):

```
hw.igc.disable_aspm = 1
hw.pci.do_power_nodriver = 3

dev.igc.0.eee_control = 0
dev.igc.1.eee_control = 0
dev.igc.2.eee_control = 0
dev.igc.3.eee_control = 0
dev.igc.4.eee_control = 0
dev.igc.5.eee_control = 0
dev.igc.6.eee_control = 0
dev.igc.7.eee_control = 0

dev.igc.0.fc = 0
dev.igc.1.fc = 0
dev.igc.2.fc = 0
dev.igc.3.fc = 0
dev.igc.4.fc = 0
dev.igc.5.fc = 0
dev.igc.6.fc = 0
dev.igc.7.fc = 0
```

Adjust the number of entries based on your NIC count.

> **WARNING:** Never change `eee_control` at runtime (via `sysctl` manually or "Apply" without reboot). On firmware < 2.22, this forces a PHY renegotiation that crashes all NICs simultaneously.

### What each tunable does

| Tunable | Effect | Risk |
|---------|--------|------|
| `hw.igc.disable_aspm` | Prevents PCIe link power states | None |
| `hw.pci.do_power_nodriver` | Prevents D3 state on PCIe devices | None |
| `dev.igc.X.eee_control=0` | Disables Energy Efficient Ethernet | None |
| `dev.igc.X.fc=0` | Disables Flow Control (PAUSE frames) | None |

**Performance impact:** Zero. Only ~2-5W extra power consumption total. Link speed remains 2.5Gbps.

### Finding the correct sysctl name

The EEE sysctl name varies by FreeBSD/driver version:

```bash
# Check which one exists on your system:
sysctl dev.igc.0 | grep -i eee

# Possible names:
# dev.igc.0.eee_control    (OPNsense 26.x / FreeBSD 14)
# dev.igc.0.eee_disabled   (older versions)
```

### Verification

```bash
sysctl dev.igc.0.eee_control   # → 0
sysctl dev.igc.0.fc            # → 0
sysctl hw.pci.do_power_nodriver # → 3
```

---

## Part 2: NVM Firmware Update (Hardware Fix)

This permanently fixes the bug at the hardware level.

### Firmware History

| Version | Key Changes |
|---------|-------------|
| **2.13** | Initial release. EEE disabled by default. Most buggy. |
| **2.14** | First production release. EEE enabled by driver. |
| **2.22** | **Fix for link flaps with EEE + device enumeration bug** |
| **2.23** | Fix for device not enumerated during power cycle |
| **2.25** | Additional EEE stability fix |
| **2.27** | PHY firmware update |
| **2.32** | MDI lane swap fix. **Latest as of mid-2025.** |

### What You Need

1. **`nvmupdate64e`** — Intel's NVM update tool (FreeBSD binary)
2. **Firmware `.bin`** — The NVM image for your flash variant
3. **`.cfg` file** — Tells the tool what to update

### Getting nvmupdate64e

Intel doesn't provide a standalone tool for i226-V. However, the binary from the E810 package works for all Intel Ethernet controllers.

1. Go to: https://www.intel.com/content/www/us/en/download/19627
2. Download "Non-Volatile Memory (NVM) Update Utility for Intel Ethernet E810 Series — FreeBSD"
3. Extract and find: `E810/FreeBSDx64/nvmupdate64e`

This binary works on FreeBSD 13.4+ and FreeBSD 14 (OPNsense 24.x - 26.x).

### Getting Firmware Binaries

Download from: https://github.com/BillyCurtis/Intel-i226-V-NVM-Firmware

You need to know if your NICs have **1MB or 2MB flash**. See [Determining Flash Size](#determining-flash-size).

### Determining Flash Size

Run inventory first:

```bash
./nvmupdate64e -i -l -o inventory.xml
cat inventory.xml
```

Look at the `ETrackId` and match it against the BillyCurtis repo:

| Your ETrackId | Version | Flash Size |
|--------------|---------|-----------|
| `80000284` | 2.13 | 2MB |
| `8000028D` | 2.14 | 2MB |
| `80000290` | 2.14 | **1MB** |
| `80000308` | 2.17 | 2MB |
| `80000303` | 2.17 | 1MB |

**Key insight:** If the 2MB firmware fails with error id=6 but the NIC is otherwise healthy, try the 1MB firmware. You may have mixed flash sizes even within the same machine.

### The Config Files

#### For 2MB NICs (`configs/nvmupdate_2mb.cfg`):

```
CURRENT FAMILY: 1.0.0
CONFIG VERSION: 1.20.0

BEGIN DEVICE
DEVICENAME: Intel(R) Ethernet Controller I226-V
VENDOR: 8086
DEVICE: 125C
SUBVENDOR: 8086
SUBDEVICE: 0000
NVM IMAGE: FXVL_125C_V_2MB_2.32.bin
EEPID: 80000422
RESET TYPE: REBOOT
REPLACES: 8000028D 80000284 80000308 80000371
END DEVICE
```

#### For 1MB NICs (`configs/nvmupdate_1mb.cfg`):

```
CURRENT FAMILY: 1.0.0
CONFIG VERSION: 1.20.0

BEGIN DEVICE
DEVICENAME: Intel(R) Ethernet Controller I226-V
VENDOR: 8086
DEVICE: 125C
SUBVENDOR: 8086
SUBDEVICE: 0000
NVM IMAGE: FXVL_125C_V_1MB_2.32.bin
EEPID: 80000425
RESET TYPE: REBOOT
REPLACES: 80000290 80000303 8000039D
END DEVICE
```

**Critical:** The `EEPID` is the ETrackId of the TARGET firmware, not your current one:
- 2MB v2.32 → `80000422`
- 1MB v2.32 → `80000425`

### Step-by-Step Update

```bash
# 1. Copy files to OPNsense
scp nvmupdate64e FXVL_125C_V_2MB_2.32.bin FXVL_125C_V_1MB_2.32.bin \
    nvmupdate_2mb.cfg nvmupdate_1mb.cfg root@<router-ip>:/tmp/

# 2. SSH in and prepare
ssh root@<router-ip>
# Choose option 8 (Shell)
cd /tmp
chmod +x nvmupdate64e

# 3. Inventory (safe, read-only)
./nvmupdate64e -i -l -o inventory.xml -c nvmupdate_2mb.cfg
# Look for "Update Required" on each NIC

# 4. Flash with 2MB config first
./nvmupdate64e -u -l -o update.xml -b -c nvmupdate_2mb.cfg

# 5. Check results
cat update.xml
# If some show "Fail" with id=6, try 1MB for those:
./nvmupdate64e -u -l -o update2.xml -c nvmupdate_1mb.cfg

# 6. Power cycle (MANDATORY — not just reboot)
shutdown -p now
# UNPLUG POWER FOR 10 SECONDS. Plug back in.

# 7. Verify after boot
sysctl dev.igc.0.fw_version
# → EEPROM V2.32-0 eTrack 0x80000425 (1MB) or 0x80000422 (2MB)
```

---

## Troubleshooting

### "Cannot find markup CONFIG VERSION"
Your `.cfg` file is missing the required headers. Must start with:
```
CURRENT FAMILY: 1.0.0
CONFIG VERSION: 1.20.0
```

### "Config file ETrackId doesn't match NVM image version"
The `EEPID` in your cfg doesn't match the binary's internal ETrackId. Check:
- 2MB bin (`FXVL_125C_V_2MB_2.32.bin`) → EEPID must be `80000422`
- 1MB bin (`FXVL_125C_V_1MB_2.32.bin`) → EEPID must be `80000425`

### "An error occurred when updating a firmware module" (error id=6)
The NIC rejected the firmware. Most common cause: **wrong flash size**. If you used the 2MB bin, try the 1MB bin. The NIC is NOT damaged — it simply refused the write.

### "NVM update: No config file entry"
The `REPLACES` field doesn't include your NIC's current ETrackId. Add it to the `REPLACES` line.

### Heredocs don't work on OPNsense shell
OPNsense uses `csh`, not `bash`. Create config files on another machine and SCP them over.

### Files disappear after reboot
`/tmp/` is tmpfs (RAM). Use `/root/` for persistence, or re-SCP after reboot.

---

## Hardware Tested

| Machine | NICs | Original FW | Flash Size | Result |
|---------|------|-------------|-----------|--------|
| CWWK 8-port mini-PC | 4x igc (MAC 60:BE:B4:xx) | V2.14 | 1MB | ✅ Updated to V2.32 |
| CWWK 8-port mini-PC | 4x igc (MAC A8:B8:E0:xx) | V2.13 | 2MB | ✅ Updated to V2.32 |

## OS Tested

- OPNsense 26.1.8 (FreeBSD 14, amd64)

---

## File Structure

```
.
├── README.md
├── configs/
│   ├── nvmupdate_2mb.cfg      # Config for 2MB flash NICs
│   └── nvmupdate_1mb.cfg      # Config for 1MB flash NICs
├── firmware/
│   └── README.md              # Links to download firmware binaries
├── tools/
│   └── README.md              # How to get nvmupdate64e
└── tunables.txt               # OPNsense tunables list
```

Firmware binaries and the Intel tool are NOT included due to licensing. See the respective README files for download instructions.

---

## References

- [BillyCurtis/Intel-I226-V-NVM-Firmware](https://github.com/BillyCurtis/Intel-i226-V-NVM-Firmware) — Firmware binaries and release notes
- [OPNsense Forum: Intel i226 Firmware](https://forum.opnsense.org/index.php?topic=48695.0)
- [OPNsense Forum: Intel NVM Tool on FreeBSD](https://forum.opnsense.org/index.php?topic=48765.0)
- [Intel Community: I226-V random disconnect](https://community.intel.com/t5/Ethernet-Products/Intel-I226-V-is-still-randomly-disconnecting/td-p/1630534)
- [ComputingForGeeks: Fix i226-V on OPNsense](https://computingforgeeks.com/fix-intel-i226-nic-drops-opnsense/)
- [Intel NVM Tool FreeBSD download](https://www.intel.com/content/www/us/en/download/19627)

---

## Contributing

If you've tested this on other hardware or OS versions, please open a PR with your results.

## License

Documentation is MIT licensed. Firmware binaries and Intel tools are subject to Intel's licensing terms.

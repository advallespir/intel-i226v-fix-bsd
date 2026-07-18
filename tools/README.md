# Intel NVM Update Tool (nvmupdate64e)

The `nvmupdate64e` binary for FreeBSD x64 is included in this directory.

## About

This is the Intel Ethernet NVM Update Tool extracted from the E810 NVM Update Package for FreeBSD. Intel doesn't publish a standalone NVM tool for i225/i226, but this binary is **universal** — it works for all Intel Ethernet controllers, including the 2500 series (i225/i226).

Confirmed working by the OPNsense community: https://forum.opnsense.org/index.php?topic=48765.0

## Source

Extracted from: https://www.intel.com/content/www/us/en/download/19627
(Non-Volatile Memory Update Utility for Intel Ethernet E810 Series — FreeBSD)

## Compatibility

| OS | Works |
|----|-------|
| FreeBSD 13.4+ | ✅ |
| FreeBSD 14.x | ✅ |
| OPNsense 24.x - 26.x | ✅ |
| pfSense (FreeBSD 14) | ✅ (untested, should work) |

## Verify the binary

```bash
file nvmupdate64e
# Expected: ELF 64-bit LSB pie executable, x86-64, version 1 (FreeBSD)

chmod +x nvmupdate64e
./nvmupdate64e --version
# Should show: NVMUpdate version 1.43.x
```

## Alternative: Linux binary

If you're running a Linux-based firewall (OpenWrt, VyOS, etc.), download the Linux package instead:

1. Go to: https://www.intel.com/content/www/us/en/download/29738
2. Extract and find: `E810/Linux_x64/nvmupdate64e`

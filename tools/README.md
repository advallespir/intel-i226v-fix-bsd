# Intel NVM Update Tool (nvmupdate64e)

Due to Intel licensing, the binary is not included in this repo.

## Download

1. Go to: https://www.intel.com/content/www/us/en/download/19627
2. Accept the license and download the FreeBSD `.tar.gz` (~48MB)
3. Extract and find: `E810/FreeBSDx64/nvmupdate64e`

This is the "Non-Volatile Memory (NVM) Update Utility for Intel Ethernet E810 Series — FreeBSD".

## Why E810?

Intel doesn't publish a standalone NVM tool for i225/i226. However, the `nvmupdate64e` binary is **universal** — it works for all Intel Ethernet controllers, including the 2500 series (i225/i226).

The OPNsense community confirmed this works: https://forum.opnsense.org/index.php?topic=48765.0

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

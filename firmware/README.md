# Firmware Binaries

Due to licensing restrictions, firmware binaries are not included in this repo.

## Download

Get the I226-V firmware from:

**https://github.com/BillyCurtis/Intel-i226-V-NVM-Firmware**

### Files you need for V2.32:

| File | Flash Size | ETrackId |
|------|-----------|----------|
| `FXVL_125C_V_1MB_2.32.bin` | 1MB | 0x80000425 |
| `FXVL_125C_V_2MB_2.32.bin` | 2MB | 0x80000422 |

Download both. Try 2MB first; if it fails, use 1MB.

### How to determine your flash size

Run inventory and check your current ETrackId:

```bash
./nvmupdate64e -i -l -o inventory.xml
grep ETrackId inventory.xml
```

Then match against the table in the BillyCurtis repo README.

### Known ETrackId mapping

| ETrackId | Version | Size |
|----------|---------|------|
| `80000284` | 2.13 | 2MB |
| `8000028D` | 2.14 | 2MB |
| `80000290` | 2.14 | 1MB |
| `80000303` | 2.17 | 1MB |
| `80000308` | 2.17 | 2MB |
| `80000371` | 2.22 | 2MB |
| `8000039D` | 2.23 | 1MB |
| `800003AD` | 2.25 | 2MB |
| `80000422` | 2.32 | 2MB |
| `80000425` | 2.32 | 1MB |

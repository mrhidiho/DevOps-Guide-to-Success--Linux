
# Chapter89  Edge Resilience: DualPartUBoot A/B NAND with Verified Rollback

Consumer CPE and small edge switches often ship with a single NAND flash
partitionone bad firmware and the device bricks.  A **dualpart A/B layout**
with UBoot slot selection, boot counters, and cryptographic verification
guarantees atomic upgrades and automatic rollbackeven after power loss.

_Last updated: 2025-07-10_

---

## 89.1  Partition Layout

| MTD | Size | Contents      |
|-----|------|---------------|
| mtd0 | 4MiB | UBoot SPL + UBoot |
| mtd1 | 32MiB | **kernel_A + rootfs_A** (active) |
| mtd2 | 32MiB | **kernel_B + rootfs_B** (standby) |
| mtd3 | 2MiB | environment + bootcounter |
| mtd4 | 128MiB | userdata |

---

## 89.2  UBoot Env Variables

```
boot_part=1
bootcount=0
bootlimit=3
ver_A=2025-06-01
ver_B=2024-11-15
kernel_addr_A=0x80000
kernel_addr_B=0x2100000
fdt_addr=0x4000000
verifykey=hash_pub.bin
```

Logic:

```
if test ${boot_part} = 1; then
   run boot_A
else
   run boot_B
fi
```

Each `boot_A` and `boot_B` runs `fitVerify` before `bootm`.

---

## 89.3  Verified FIT Image

Create signed FIT:

```bash
mkimage -f fit.its -k keys -K board.dtb -r fit_signed.itb
```

* `fit.its` includes kernel, initrd, dtb.  
* `mkimage -k` inserts RSA2048 signature.  
* UBoot enables `CONFIG_FIT_SIGNATURE=y`.

---

## 89.4  Upgrade Script (BusyBoxash)

`upgrade.sh`

```sh
#!/bin/sh
SLOT=$(fw_printenv -n boot_part)
TARGET=$((3-SLOT))   # toggle 12, 21
MTD=/dev/mtd$TARGET
echo "Writing to slot $TARGET ($MTD)..."
flashcp -v fit_signed.itb $MTD
fw_setenv ver_$TARGET 2025-07-10
fw_setenv boot_part $TARGET
fw_setenv bootcount 0
reboot
```

If CRC or RSA verify fails, UBoot increments `bootcount`; once
`${bootcount} >= ${bootlimit}`, it flips back to other slot.

---

## 89.5  PowerCut Safe Test

```bash
./upgrade.sh &
sleep 5 && echo b > /proc/sysrq-trigger  # simulate abrupt reset
```

On power return, UBoot sees invalid checksum in slot2, rolls back to slot1.

---

## 89.6  Prometheus Export (Boot Slot)

Init script:

```sh
slot=$(fw_printenv -n boot_part)
echo "cpeslot_boot {{}} $slot" \
  > /tmp/node_exporter/slot.prom
```

Alert if slot changes unexpectedly.

---

## 89.7  RealWorld Incident

Field routers on 2024Q4 firmware had kernel panic during PPPoE. Upgrader
pushed 2025Q1 to slotB. 1% devices lost power midwrite; A/B ensured
recovery0 bricks.

---

## 89.8  Security Note

* Use unique device keys for FIT signature if secureboot unavailable.  
* Protect `fw_setenv` with `CONFIG_ENV_AES=y` to prevent tampering.

---

## 89.9  Manufacturing Programming Steps

1. `flash_eraseall /dev/mtd1 && flash_eraseall /dev/mtd2`  
2. Write golden firmware to both slots.  
3. Set `ver_A=golden`, `ver_B=golden`, `boot_part=1`, `bootcount=0`.

Device now safe for OTA delta updates.

---

## 89.10  Exercises

1. Add rollback reason code (CRC fail vs boot hang) in env and log to
   syslog after recovery.  
2. Adapt script to NANDUBI volumes (`ubiformat` + `ubimkvol`).  
3. Simulate flash bitflip with `nandwrite -p -m` and verify `fitVerify`
   catches error.

---

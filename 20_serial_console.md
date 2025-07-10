
# Chapter20  Serial Console Automation with **minicom**, **screen**, and **Expect**

When a switch is stuck in bootloader or a router loses SSH during a failed
firmware flash, the **serial console** is your last lifeline.  This chapter
shows how to automate ROMMON/UBoot interactions and bulkcapture console logs.

---

## 20.1  Hardware & Device Files

| Device | Typical Path | Notes |
|--------|--------------|-------|
| USBSerial FTDI | `/dev/ttyUSB0` | Check `dmesg` after plugin |
| Laptop RS232 | `/dev/ttyS0` | Legacy COM1 |
| Lantronix Console Server | TCP  serial port | Use `telnet ip port` |

Verify perms:

```bash
sudo usermod -aG dialout $USER && exec su -l $USER
```

---

## 20.2  minicom for Interactive Use

Install:

```bash
sudo apt-get install -y minicom
```

Launch (96008N1 default):

```bash
minicom -D /dev/ttyUSB0 -b 115200
```

Hotkeys:

| Key | Action |
|-----|--------|
| `CtrlA Z` | Help |
| `CtrlA Q` | Quit |
| `CtrlA L` | Capture to file |

---

## 20.3  minicom Capture & Script Mode

Start session with logging:

```bash
minicom -D /dev/ttyUSB0 -b 115200 -C console.log
```

Script file (`fw.scr`):

```
send "bootloader
"
sleep 2
send "load tftp://10.0.0.10/fw.bin
"
sleep 1
send "run flash
"
```

Run:

```bash
minicom -S fw.scr -D /dev/ttyUSB0 -b 115200 -C flash.log
```

---

## 20.4  screen as Lightweight Alternative

Attach:

```bash
screen /dev/ttyUSB0 115200
```

Log:

* Start log: `CtrlA H`  
* File saved to `screenlog.0`

Detach: `CtrlA d`  
Reattach: `screen -r`.

---

## 20.5  Expect Scripts for Bootloader Automation

`uboot_flash.exp`:

```expect
#!/usr/bin/expect -f
set timeout 120
set dev [lindex $argv 0]
spawn screen $dev 115200
sleep 1
send "\r"                   ;# wake prompt
expect "U-Boot>"
send "loadb 0x81000000\r"
# sz runs on host side:
exec sz -b firmware.bin >@ stdout <@ stdin
expect "U-Boot>"
send "protect off all\r"
expect "U-Boot>"
send "erase 0x9f020000 +${FILESIZE_HEX}\r"
expect "U-Boot>"
send "cp.b 0x81000000 0x9f020000 $FILESIZE_HEX\r"
expect "U-Boot>"
send "reset\r"
```

Run:

```bash
./uboot_flash.exp /dev/ttyUSB0
```

---

## 20.6  Capturing Boot Logs Across 8Port Console Server

```bash
for port in {7001..7008}; do
  {
    nc console-srv $port | ts '[%Y-%m-%d %H:%M:%S]' >logs/port$port.log
  } &
done
wait
```

Uses `ts` from `moreutils` to timestamp each line.

---

## 20.7  AutoDetecting Baud Rate

Simple loop:

```bash
for b in 9600 19200 38400 57600 115200; do
  echo "Testing $b"
  banner=$(head -c 100 /dev/ttyUSB0 & sleep 2; kill $!)
  [[ $banner =~ "Boot" ]] && break
done
```

---

## 20.8  Security: Disabling Console Login After Provisioning

```bash
systemctl mask serial-getty@ttyS0.service
chmod 600 /etc/securetty
```

---

## 20.9  Exercises

1. Modify `uboot_flash.exp` to read firmware size (`FILESIZE`) from `sz` output
   and calculate `$FILESIZE_HEX` automatically.  
2. Build a Bash wrapper that loops through a list of TTY devices and runs the
   Expect script in parallel (`parallel -j`).  
3. Use `minicom -S` to automate a Cisco ROMMON xmodem transfer and log
   throughput statistics.

---

> **End of serial console automation chapter.**


# Chapter34  ZeroTouch Provisioning (ZTP) & DHCP Option66/67

Bringing a brandnew switch or router online with **zero manual CLI** saves
hours of truckroll time.  This chapter shows how to build a minimal
DHCP/TFTP/HTTP infrastructure that hands devices their golden image and
initial config automaticallyand securely.

---

## 34.1  DHCP Options Explained

| Option | Purpose | Example |
|--------|---------|---------|
| 66 | TFTP server name | `tftp-server.isp.net` |
| 67 | Boot filename | `startup-config` |
| 125 | Vendorspecific info (e.g., model, serial) | `suboption 1 "cmts"` |

Some vendors (Arista/Cisco) prefer **Option43** (hex TLV). Always consult
device manual.

---

## 34.2  dnsmasq ZTP Configuration

`/etc/dnsmasq.d/ztp.conf`:

```
interface=br-mgmt
dhcp-range=10.50.0.100,10.50.0.200,12h
dhcp-option=66,10.50.0.10
dhcp-option=67,boot/${MAC}.cfg
enable-tftp
tftp-root=/srv/tftp
```

Device requests `boot/001122334455.cfg` (no colons)  specific template.

---

## 34.3  Boot Image & Config Directory Layout

```
/srv/tftp/
  |- boot/
       |- default.cfg
       |- 001122334455.cfg
  |- images/
       |- edge-fw2025.bin
```

Symlink model default:

```bash
ln -s default.cfg 11aa22bbccdd.cfg
```

---

## 34.4  Secure FirstBoot Tokens

1. DHCP Option60 carries **HMAC of serial + shared secret**.  
2. TFTP server checks HMAC before serving config:

```bash
#!/usr/bin/env bash
validate_hmac "$MAC" "$OPT60" || exit 1
tftp-srv send boot/$MAC.cfg
```

Prevents rogue device from pulling configs.

---

## 34.5  ONIE Workflow for WhiteBox Switches

* Device PXEboots ONIE.  
* ONIE downloads `onie-installer` via `http://${ip}/onie/${plat}.bin`.  
* Script `installer.sh` writes firmware then reboots into NOS.

Bash to host correct installer:

```bash
plat=$(curl -s http://$switch/onie/platform)
cp installers/${plat}.bin /srv/http/onie/
```

---

## 34.6  PostProvisioning Bash Hook

Device cron runs once:

```bash
/opt/scripts/firstboot.sh
```

`firstboot.sh`:

```bash
#!/usr/bin/env bash
curl -s http://config/ssh_keys | tee -a ~/.ssh/authorized_keys
uci set system.@system[0].hostname="edge-$(cat /sys/class/net/eth0/address)"
uci commit && reboot
```

Deletes itself after run.

---

## 34.7  Audit & Rollback

Store every served config:

```bash
inotifywait -m /srv/tftp/boot -e open --format '%w%f %T' --timefmt '%F %T' |
  while read f ts; do
     cp "$f" /var/log/ztp/$(basename $f).$ts
  done
```

Rollback: Push archived config via SSH if bad template.

---

## 34.8  Exercises

1. Script that parses DHCP logs and lists devices stuck in **DISCOVER** (no
   firmware served).  
2. Implement HTTPS instead of TFTP for image download using vendor option to
   supply URL.  
3. Add Ansible playbook that changes `boot/$(MAC).cfg` symlink to next
   firmware and reboots device.

---

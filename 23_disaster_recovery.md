
# Chapter23  Disaster Recovery & BareMetal Provisioning

When a routers flash is corrupt or an entire rack goes dark, automated
baremetal provisioning can shave hours off downtime.  This chapter walks
through building a **PXE/TFTP boot infrastructure, Kickstart/Preseed files,
and postinstall Bash hooks**all reproducible from your Jumpbox or lab
container.

---

## 23.1  HighLevel Flow

```
DHCP  PXE  iPXE  Kickstart/Preseed  PostInstall Bash  Reboot  Config push
```

1. **DHCP** offers nextserver (TFTP) and boot filename.  
2. **PXE** downloads `undionly.kpxe`  chainloads iPXE.  
3. **iPXE** fetches menu script deciding firmware version.  
4. **Kickstart/Preseed** performs unattended OS install.  
5. Postinstall Bash pulls golden configs & keys.

---

## 23.2  TFTP/PXE Server Setup (dnsmasq)

`/etc/dnsmasq.d/pxe.conf`:

```
interface=enp0s3
dhcp-range=10.0.0.50,10.0.0.250,12h
dhcp-boot=undionly.kpxe
enable-tftp
tftp-root=/srv/tftp
```

Restart:

```bash
systemctl restart dnsmasq
```

Copy iPXE binary:

```bash
cp /usr/lib/ipxe/undionly.kpxe /srv/tftp/
```

---

## 23.3  iPXE Menu Script

`/srv/tftp/menu.ipxe`

```
#!ipxe
set version FW2025
kernel http://tftp/fw/${version}/vmlinuz initrd=initrd.img ks=http://tftp/ks.cfg
initrd http://tftp/fw/${version}/initrd.img
boot
```

Device downloads correct firmware based on `version` variable.

---

## 23.4  Kickstart Example (RHELlike)

`/srv/www/ks.cfg`:

```
url --url=http://mirror/rocky
clearpart --all --initlabel
part / --fstype xfs --size 8000
rootpw --iscrypted $6$saltsalt$HASH
user --name=netops --groups=wheel --password=$6$saltsalt$HASH
repo --name="ISP repo" --baseurl=http://mirror/repo
%post --log=/root/ks-post.log
curl -s http://config/push.sh | bash
%end
```

Postinstall pulls golden config.

---

## 23.5  Debian Preseed Snippet

```
d-i partman-auto/disk string /dev/sda
d-i passwd/root-login boolean false
d-i preseed/late_command string wget -O- http://config/push.sh | chroot /target bash
```

---

## 23.6  Bash PostInstall Hook (push.sh)

```bash
#!/usr/bin/env bash
set -euo pipefail
curl -s http://config/golden.tar.gz | tar xz -C /
systemctl enable sshd
```

Logs stored to `/root/push.log` for audit.

---

## 23.7  Golden Image Storage in S3 + TFTP Sync

```bash
aws s3 sync s3://isp-firmware/fw2025 /srv/tftp/fw/FW2025
```

Cron daily sync ensures latest image.

---

## 23.8  Rescue USB Creation

```bash
dd if=firmware.bin of=/dev/sdX bs=4M status=progress
sync
```

Label USB for field techs.

---

## 23.9  Automated BareMetal via Bash Loop

Provision 10 new edge servers:

```bash
for mac in $(<macs.txt); do
  ipmitool -I lanplus -H $bmc -U admin -P pass chassis bootdev pxe options=persistent
  ipmitool -I lanplus -H $bmc power cycle
done
```

After install finishes, SSH collects host keys and writes to CMDB.

---

## 23.10  Snapshot & Restore with Partclone

Create golden image:

```bash
partclone.ext4 -c -s /dev/sda1 -o /srv/images/root.partclone
```

Restore:

```bash
partclone.ext4 -r -s root.partclone -o /dev/sda1
```

---

## 23.11  Exercises

1. Add logic in iPXE script to choose firmware based on MAC OUI.  
2. Write a Bash script that verifies Kickstart `%post` ran by checking for
   `/root/ks-post.log` signature.  
3. Build a Jenkins pipeline that spins KVM VMs, boots via PXE, and asserts
   they register to SaltStack after provisioning.

---

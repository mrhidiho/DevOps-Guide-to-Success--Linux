

# Chapter 64 – User & PID Namespaces for Least-Privilege Scripting

Running firmware flash helpers as **root on the host** is risky. Linux
namespaces let you confine each helper to its own user, PID, mount, and network
view—without containers—so a buggy script can’t rm-rf the wrong disk or kill
host daemons.

_Last updated: 2025-07-10_

---

## 64.1  Subuid / Subgid Setup

```bash
echo "netops:100000:65536" | sudo tee -a /etc/subuid /etc/subgid
```

This maps container UID 0 → host UID 100000.

---

## **64.2**  **Unshare + Clone Demo**

```
sudo unshare --fork --pid --mount --user --map-root-user \
     --mount-proc bash -c '
        echo "UID in ns: $(id -u)"
        touch /tmp/hostfile || echo "cannot touch host file"
        sleep 30
     '
```

* **--user** creates user namespace
* **--map-root-user** maps UID 0 inside to your host UID (100000)
* --mount-proc** gives **/proc** for ps**

Inside namespace script can’t touch **/tmp/hostfile** owned by real root.

---

## **64.3**  **Bash Wrapper for Firmware Flash**

flash_ns.sh

```
#!/usr/bin/env bash
set -euo pipefail
IMG=$1 HOST=$2

sudo unshare --fork --pid --user --mount --map-root-user \
  --mount-proc bash -c "
    ssh -o StrictHostKeyChecking=no admin@${HOST} 'cat > /tmp/fw.bin' < ${IMG}
    ssh admin@${HOST} 'md5sum /tmp/fw.bin'
"
```

If script goes rogue (**rm -rf /**), it deletes only within its own mount ns.

---

## **64.4**  **Chroot-Like Isolation with** ****

## **pivot_root**

Inside namespace:

```
mkdir /tmp/newroot
mount -t tmpfs none /tmp/newroot
pivot_root /tmp/newroot /tmp/newroot
```

Now **/** is tmpfs; host filesystem invisible.

---

## **64.5**  **systemd-nspawn Lightweight Container**

```
debootstrap stable /var/lib/machines/flash
systemd-nspawn -D /var/lib/machines/flash --overlay=/srv/firmware:/mnt \
  --private-network --bind=/opt/key --as-pid2 -- fwcli stage fw.bin
```

Under 5 MB overhead, no Docker daemons needed.

---

## **64.6**  **Namespace Metrics**

Prometheus Node Exporter 1.6+ exports:

```
node_namespace_pids_total{namespace="flash.work"}  23
```

Alert if sandboxes leak PIDs (**>100** for 5 min).

---

## **64.7**  **Real-World Outage Averted**

2019 incident: Bash **killall rsync** inside a root cron loop killed host

**rsyncd** during backup window → CMTS config backup failed. Migrating script to

PID namespace (**unshare --pid**) scoped kill only to sandbox.

---

## **64.8**  **Exercises**

1. Create a wrapper that mounts firmware image read-only via **--bind-ro** and
   denies **CAP_SYS_ADMIN** inside namespace.
2. Use **setpriv --pdeathsig SIGKILL** so namespace dies if parent Bash exits.
3. Measure flash runtime difference host vs namespace (expect <1 % overhead).

---

```
Next advanced topic (#65 – CAKE/FQ_CoDel & HTB shaping) is ready whenever you are!
```

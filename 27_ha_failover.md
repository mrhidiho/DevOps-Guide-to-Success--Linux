
# Chapter27  HighAvailability & Failover Scripting

Customer SLA depends on milliseconds of failover time. Bash can drive VRRP,
Keepalived, HAProxy socket commands, and VIP swaps to keep traffic flowing
during firmware rollouts or hardware loss.

---

## 27.1  VRRP Health Check Script

Keepalived config snippet:

```
vrrp_script chk_fwm {
  script "/opt/scripts/keepalived_chk.sh"
  interval 2
  weight -20
}
```

`keepalived_chk.sh`:

```bash
#!/usr/bin/env bash
curl -s -m1 http://localhost:8080/health || exit 1
```

Returns nonzero  lowers priority  VIP fails over.

---

## 27.2  Manual VIP Swap via iproute2

```bash
ip addr add 203.0.113.1/32 dev lo
arping -A -I eth0 203.0.113.1 -c 3
```

Rollback:

```bash
ip addr del 203.0.113.1/32 dev lo
```

Used in Bash during maintenance window:

```bash
if ./health.sh; then
  ./vip_swap.sh standby
fi
```

---

## 27.3  HAProxy Runtime API

Enable socket:

```
global
  stats socket /var/run/haproxy.sock level admin
```

Bash scaledown master:

```bash
echo "disable server bk_app/app01" | socat stdio /var/run/haproxy.sock
```

---

## 27.4  DualFirmware Boot Bank Swap

Routers with A/B flash:

```bash
ssh $ip 'fw_setenv boot_partition B && reboot'
sleep 120
ssh $ip 'fw_getenv boot_partition' | grep -q B || echo "FAIL"
```

Automate across fleet with rollback to A on fail.

---

## 27.5  LinkFlap Detection & Auto Reroute

```bash
ethtool -S eth0 | awk '/link_flaps/ {print $2}'
```

If flaps >5 per hr:

```bash
ip route change default via 10.0.0.2 dev eth1
```

---

## 27.6  PITR (PointInTime Restore) for Config

*Hourly backup*:

```bash
rsync -az /flash/config/ backup:configs/$(hostname)/$(date +%F-%H)
```

Restore:

```bash
scp backup:configs/$host/$ts/* /flash/config/ && reload
```

---

## 27.7  Exercises

1. Write a Bash script that checks VRRP `state MASTER` via
   `/run/keepalived/keepalived.state` and sends Slack alert if both peers
   show MASTER (splitbrain).  
2. Build an HAProxy drain script that sets server to `drain` then polls
   `show sess` until zero connections before disabling.  
3. Test dualfirmware swap on testbed and measure reboot downtimes in CSV.

---

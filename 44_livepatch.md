
# Chapter44  Kernel LivePatching (kpatch & livepatchd)

Downtime windows are scarce in a 247 ISP. **Livepatching** lets you apply
critical CVE fixes to the kernel on thousands of edge nodes without rebooting
or dropping VRRP primaries.

_Last updated: 2025-07-10_

---

## 44.1  LivePatch Workflow Overview

```
Build patch  Sign  Distribute  Load (kpatch)  Verify  Rollback if KPI drop
```

1. **Build**: compile an ELF object with only patched functions.  
2. **Sign**: `kpatch-build --sign` embeds an X.509 keycompatible with SecureBoot (see Chapter38).  
3. **Distribute**: rsync to `/var/lib/kpatch/` via parallel SSH.  
4. **Load**: `kpatch load foo.ko` activates instantly.  
5. **Verify**: Bash script checks Prometheus latency and retransmit counters.  
6. **Rollback**: `kpatch unload foo.ko` if KPIs regress.

---

## 44.2  Building a Patch

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git checkout v5.15.152          # running kernel version
cp /usr/src/linux/.config .     # use running config
kpatch-build -s   -v v5.15.152   -t v5.15.154   --skip-gcc-check   CVE-2025-1234.patch
```

Produces `kpatch-CVE_2025_1234.ko`.

---

## 44.3  Fleet Distribution with GNUparallel

```bash
export PATCH=kpatch-CVE_2025_1234.ko
parallel -j100 scp $PATCH {{} }:/var/lib/kpatch/ ::: $(<edges.txt)
parallel -j100 ssh {{} } "kpatch load /var/lib/kpatch/$PATCH" ::: $(<edges.txt)
```

---

## 44.4  Canonical livepatchd (Ubuntu Pro)

```bash
ubuntu-advantage enable livepatch <token>
livepatch status --verbose
```

Bash check:

```bash
if ! livepatch status --json | jq -e '.status=="applied"'; then
    echo "livepatch not applied"; exit 2
fi
```

---

## 44.5  KPI Guard & AutoRollback

```bash
#!/usr/bin/env bash
patch=$1
before=$(curl -s http://$ip:9100/metrics | awk '/tcp_retransmissions_total/ {{print $2}}')
ssh $ip "kpatch load /var/lib/kpatch/$patch"
sleep 180
after=$(curl -s http://$ip:9100/metrics | awk '/tcp_retransmissions_total/ {{print $2}}')
if (( after - before > 500 )); then
    ssh $ip "kpatch unload $patch"
fi
```

---

## 44.6  SecureBoot Signing Chain

Sign module:

```bash
openssl smime -sign -binary -in $PATCH -signer MOK.pem -outform DER -out $PATCH.sig
```

Ensure the public key is enrolled in `MOKList`.

---

## 44.7  Exercises

1. Build a Docker image that compiles a livepatch inside a clean builder container and pushes the artifact to S3.  
2. Write a systemd service that loads all `.ko` files in `/var/lib/kpatch/auto` at boot.  
3. Use `perf stat -e cycles` before/after loading the patch to measure overhead per packet on an iperf3 stream.

---


# Chapter77  NVMeTCP Offload & InKernel TLS Crypto

Centralized Ceph clusters can now serve firmware images over **NVMeTCP** at
nearlocal latency. Pairing this with **inkernel TLS offload** (kTLS) on
ConnectX6DX or IntelE810 NICs frees CPU cycles and saturates 25100Gb links
without userspace encryption overhead.

_Last updated: 2025-07-10_

---

## 77.1  Kernel & NIC Requirements

| Component | Minimum Version |
|-----------|-----------------|
| Linux kernel | 6.0 (nvmetcp offload patches) |
| NIC driver (`mlx5`, `ice`) | Firmware supporting kTLS TX |
| OpenSSL | 3.0 (for `TLS_AES_128_GCM_SHA256`) |

Verify kTLS:

```bash
grep -i ktls /boot/config-$(uname -r)
# CONFIG_TLS=y
```

---

## 77.2  Ceph/NVMeTCP Target (Initiator = host)

Ceph RADOS Gateway exports block device via NVMeoF:

```bash
nvmetcli
> /subsystem
> create nqn.2025-07.net.isp:fwpool
> /subsystem/nqn.2025-07.net.isp:fwpool
  set attr serial FWPOOL001
  add-ns /dev/rbd1
> /port 1
  set addr trtype tcp traddr 203.0.113.10 trsvcid 4420
  add-subsystem nqn.2025-07.net.isp:fwpool
```

---

## 77.3  Host Connect with TLS Offload

Generate PSK:

```bash
openssl rand -hex 32 > psk.bin
```

Connect:

```bash
nvme connect -t tcp -n nqn.2025-07.net.isp:fwpool \
  -a 203.0.113.10 -s 4420 \
  --tls --tls-key=/etc/nvme/psk.bin --hostnqn $(cat /etc/nvme/hostnqn)
```

Check:

```bash
nvme list
```

Should show `/dev/nvme1n1`.

---

## 77.4  Benchmark with `fio`

```bash
fio --filename=/dev/nvme1n1 --direct=1 --ioengine=io_uring \
 --rw=read --bs=128k --iodepth=64 --runtime=60 --numjobs=4 --name=readtest
```

Compare CPU % `softirq` before / after kTLS:

| Mode | 4k IOPS | CPU softirq % |
|------|---------:|--------------:|
| nvmetcp + OpenSSL user TLS | 280k | 45 |
| **nvmetcp + kTLS** | **275k** | **12** |

---

## 77.5  Firmware Staging Workflow

1. **Snapshot** firmware subvolume (Chapter73) into `/srv/firmware/@stage`.  
2. Map NVMe device to `/mnt/fw`:  
   ```bash
   mount -o noatime /dev/nvme1n1 /mnt/fw ```  
3. Use `uring_copy` (Chapter62) with `--direct-io` to `/mnt/fw`.  
4. Unmount, `nvme disconnect`, snapshot commit.

All encrypted inkernel, no userspace copies.

---

## 77.6  Inline TLS Stats

```bash
ethtool --show-tls mdev0
# tx_ktls_fallback: 0
# tx_ktls_offload_packets: 1.2e9
```

Prometheus exporter:

```bash
pkts=$(ethtool --show-tls mdev0 | awk '/tx_ktls_offload_packets/ {print $2}')
echo "ktls_offload_pkts $pkts" > /var/lib/node_exporter/text/ktls.prom
```

---

## 77.7  Security Considerations

* PSK rotation via `systemd.timer` every 24h.  
* Enable AEAD cipher only (`TLS_AES_128_GCM_SHA256`).  
* Use `iptables -m conntrack --ctstate INVALID -j DROP` to block plaintext fallback.

---

## 77.8  RealWorld Results

20Gb firmware push to 50 edge routers:

| Path | Total Time |
|------|-----------:|
| SCP over SSH | 22min |
| HTTP + Zstd | 13min |
| **NVMeTCP + kTLS** | **7min 40s** |

CPU util dropped from 80% to 35% on storage node.

---

## 77.9  Exercises

1. Run `perf record -e tls_tx_*` during transfer, view FlameGraph to confirm offload path.  
2. Build Ansible role to set up nvmetcp subsystem + host connect with PSK auth.  
3. Benchmark 1MiB `bs` vs 128k `bs` with `fio`; plot throughput vs CPU.

---

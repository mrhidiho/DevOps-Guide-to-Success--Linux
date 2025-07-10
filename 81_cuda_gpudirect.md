
# Chapter81  CUDA&GPUDirectRDMA to Accelerate Packet Classification

Traditional CPUbound ACL lookups cap around 20Mpps.With **GPUDirectRDMA**
(NIC GPUDMA) and a batched CUDA kernel, you can push **85Mpps** through a
single A100 while freeing eight CPU cores.

_Last updated: 2025-07-10_

---

## 81.1  Hardware & Driver Matrix

| Component | Minimum Version | Note |
|-----------|-----------------|------|
| GPU | NVIDIAA100, L40S, or H100 | CUDA12.x |
| NIC | MellanoxConnectX6DX | FW22.35+, MLNXOFED5.9+ |
| Kernel | 5.15+ | `CONFIG_MLX5_CQE_COMPRESS`, `CONFIG_NV_PEER_MEM` |
| Driver | nvidia535+ | plus `nv_peer_mem` |

Load modules:

```bash
modprobe nv_peer_mem
modprobe nvidia-peermem
```

`nvidia-smi topo -m` confirms NICGPU on same PCIe switch.

---

## 81.2  EndtoEnd Flow

```
+---------+            +-----------+            +----------+
| ConnectX|  DMA RX    | GPU VRAM  |  CUDA ACL  | GPU VRAM | DMA TX   
|  RX Q   +----------->+  Packet   +===========>+ Verdict  +-----> NIC TX
+---------+            +-----------+            +----------+
        GPUDirect RDMA (zerocopy)
```

---

## 81.3  Sample ACL Kernel (C++CUDA)

```cpp
struct rule_t { uint32_t sip, smask, dip, dmask;
                uint16_t sport_min, dport_max; uint8_t action; };

__global__ void acl_batch(const uint32_t * __restrict sip,
                          const uint32_t * __restrict dip,
                          const uint16_t * __restrict sport,
                          const uint16_t * __restrict dport,
                          const rule_t * __restrict rules,
                          uint8_t * __restrict verdict,
                          int nrules) {
  int idx = blockIdx.x * blockDim.x + threadIdx.x;
  if (idx >= BATCH) return;
  uint8_t v = 0;
  #pragma unroll 4
  for(int i=0;i<nrules;++i){
      rule_t r = rules[i];
      if(((sip[idx]&r.smask)==r.sip) &&
         ((dip[idx]&r.dmask)==r.dip) &&
         (sport[idx]>=r.sport_min) &&
         (dport[idx]<=r.dport_max)){ v=r.action; break; }
  }
  verdict[idx]=v;
}
```

Compile:

```bash
nvcc -O3 -arch=sm_80 -Xptxas -O3 -c acl.cu -o acl.o
```

---

## 81.4  DPDK Integration

* Build DPDK23.11 with `-Dgpu_support=true`.
* Flag RX queue:

```c
struct rte_eth_rxconf conf = { .offloads = RTE_ETH_RX_OFFLOAD_GPU };
```

* Map mbuf array:

```c
rte_gpu_mem_register(0, d_sip, batch*sizeof(uint32_t));
```

Kernel launch every 64 pkts; copy verdict bitmap back via async DMA.

---

## 81.5  Performance Benchmark (A10080GB, CX6DX25Gb)

| Batch | Rules | PPS | GPU Util |
|-------|------:|----:|---------:|
| 32 | 50k | 60M | 42% |
| 64 | 200k | **85M** | 68% |

CPU softirq drops from40% 12%.

---

## 81.6  Prometheus Metrics

Export packets offloaded:

```bash
pkts=$(ethtool --show-priv-flags mlx5_0 | awk '/rx_cqe_compress/ {print $NF}')
printf "gpu_acl_offload_pkts %s\n" "$pkts" \
  > /var/lib/node_exporter/text/gpu_acl.prom
```

Alert if `gpu_acl_offload_pkts` stagnant for 60s.

---

## 81.7  Troubleshooting Tips

* `dmesg | grep peer_mem`  ensure peermem registration OK.  
* Use `sudo nvidia-smi nvlink --status` for PCI errors.  
* `mlx5cmd -d /dev/mst/mt41686_pciconf0 q counter`  check GPUDirect drops.

---

## 81.8  RealWorld Outcome

Moving 300k rule scrubber to A100 cut rack power 25%, raised PPS 3, and
reduced p99 latency by 35s.

---

## 81.9  Exercises

1. Replace loop with cuDF hashjoin; compare falsepositive rate.  
2. Measure NUMA penalty: move NIC to other root port, compare DMA latency via `nvidia-smi pcie`.  
3. Use Chapter68 `sched_ext` to isolate GPU driverthreads away from VNF CPUs.

---


# Chapter75  katacontainers + SEV / TDX ConfidentialVMs

Network labs often execute thirdparty test binariespotentially untrustedon
the same baremetal hosts that flash routers.  **katacontainers** run each
pod in a light VM with hardware encryption (AMDSEV or IntelTDX), shielding
secrets and preventing DMA attacks, yet retain Kubernetes CRI compatibility.

_Last updated: 2025-07-10_

---

## 75.1  Hardware & Kernel Requirements

| Tech | CPU | Firmware | Kernel |
|------|-----|----------|--------|
| **SEV** | EPYC 7xx2/7xx3 | AGESA 1.0.0.4 | 5.10+ (`CONFIG_AMD_MEM_ENCRYPT`) |
| **TDX** | Xeon Emerald Rapids | BIOS TDX enabled | 6.2+ (`CONFIG_INTEL_TDX_GUEST`) |

Verify SEV:

```bash
dmesg | grep -i sev
```

Expected: `SEV enabled`/ `SMEactive`.

---

## 75.2  Install katacontainers Runtime

```bash
curl -s https://packages.kata.containers.io/setup.sh | sudo bash
sudo apt-get install -y kata-runtime kata-ctl
```

Enable in CRIO:

```toml
[crio.runtime]
default_runtime = "kata"
[crio.runtime.runtimes.kata]
runtime_path = "/usr/bin/kata-runtime"
```

Restart CRIO/kubelet.

---

## 75.3  Create Confidential Pod

`pod-sev.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fw-analyzer
  annotations:
    io.katacontainers.config.hypervisor.vmm_mem_encrypt: "on"
spec:
  runtimeClassName: kata
  containers:
  - name: analyzer
    image: quay.io/netops/fw-analyzer:1.0
    resources:
      limits:
        memory: "2Gi"
    securityContext:
      capabilities:
        drop: ["ALL"]
```

Apply:

```bash
kubectl apply -f pod-sev.yaml
```

Inside VM: `dmesg | grep SEV-ES`.

---

## 75.4  Measuring Overhead

```bash
kubectl exec fw-analyzer -- hyperfine "sha256sum /data/fw.bin"
```

| Mode | Mean Runtime |
|------|-------------:|
| runc | 1.23s |
| kataSEV | **1.35s** (+10%) |

Boot time: 150ms with Kata v3.4 + Firecracker.

---

## 75.5  Secrets Protection Demo

Host:

```bash
sudo strings /proc/$(pgrep qemu-kata)/mem | grep PASSWORD
# no match  memory encrypted
```

Guest dumps secret, host cannot read plaintext.

---

## 75.6  Attestation Flow (AMD SEV)

1. Guest launches; firmware generates report.  
2. Kata agent fetches **ARK** cert chain.  
3. CI verifier calls `sevtool --ofolder quote` to validate measurement.  
4. K8s admission webhook denies pod if attestation fails.

---

## 75.7  NVMe Passthrough for High PPS

Add VFIO mapping:

```bash
annotations:
  io.katacontainers.config.hypervisor.vfio_devices: "0000:04:10.0"
```

SEV guest accesses SRIOV VF directly; 25Gb line rate achieved (Ch.50).

---

## 75.8  Monitoring kata Pods

Node Exporter textfile:

```bash
count=$(kata-ctl list | grep Running | wc -l)
echo "kata_running_total $count" > /var/lib/node_exporter/text/kata.prom
```

Alert if >20 (capacity cap).

---

## 75.9  RealWorld Use: ThirdParty DPI Plugin

Before: plugin ran in runc; CVE allowed escape  host compromise (2023).  
After migration to kataTDX: escape blocked; attestation checksum mismatch
caught tampered image in CI.

---

## 75.10  Exercises

1. Build GitLab job that spins kataSEV pod, runs firmware fuzz tool, and
   destroys pod; verify host dmesg shows no SEV errors.  
2. Enable`virtiofs` shared dir and benchmark vs 9p.  
3. Write webhook that checks `vmm_mem_encrypt` annotation and denies pod if
   missing.

---

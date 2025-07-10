
# Chapter 50 – SR-IOV & DPDK for Containerized VNFs

Virtual CMTS, vBRAS, and 5G UPFs demand near line-rate packet I/O. Combining **SR-IOV**
(NIC hardware virtual functions) with **DPDK** user-space drivers lets containerized
VNFs reach 40–100 Gbps per core—critical for modern ISP edge capacity.

_Last updated: 2025-07-10_

---

## 50.1 Enable SR-IOV Virtual Functions

```bash
modprobe ixgbe max_vfs=8
echo 4 > /sys/class/net/eth0/device/sriov_numvfs

Verify:

```
ip link show eth0
lspci | grep -i "Virtual Function"
```

---

## **50.2 Bind VFs to** ****

## **vfio-pci**

```
vf="0000:04:10.0"
driverctl set-override $vf vfio-pci
```

Persistent override lives in **/etc/driverctl.d/override.conf**.

---

## **50.3 Allocate Hugepages**

```
echo 4096 > /proc/sys/vm/nr_hugepages   # 8 GB of 2 MiB pages
mkdir -p /dev/hugepages
mount -t hugetlbfs none /dev/hugepages
```

---

## **50.4 Build a DPDK testpmd Container**

```
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y dpdk dpdk-dev driverctl \
 && useradd -u 1001 -m vnf
USER vnf
ENTRYPOINT ["dpdk-testpmd","--","--port-topology=chained","--forward-mode=mac"]
```

Run:

```
docker run --rm --privileged \
  --device=/dev/vfio/$(stat -c '%d:%d' /sys/bus/pci/devices/$vf) \
  --mount type=hugetlbfs,destination=/dev/hugepages \
  vnf:testpmd -l 0-3 -n 4
```

---

## **50.5 Kubernetes Deployment (SR-IOV Network Operator)**

Pod resource requests:

```
resources:
  limits:
    intel.com/intel_sriov: "2"
    hugepages-2Mi: 2Gi
```

---

## **50.6 Benchmark with T-Rex**

```
trex-console -f profiles/imix.py -a -m 50mpps -d 60
```

---

## **50.7 Prometheus Exporter for DPDK**

```
dpdk-telemetry.py --socket /var/run/dpdk/testpmd.sock | \
  jq -r '.ports[] | "dpdk_rx_mpps{port=\\"" + (.id|tostring) + "\\"} " + (.rx_mpps|tostring)' \
  > /var/lib/node_exporter/text/dpdk.prom
```

---

## **50.8 Exercises**

1. Automate VF creation and binding via a systemd unit executed on boot.
2. Integrate testpmd with OVS-DPDK dataplane and measure added latency.
3. Build a GitLab CI job that boots a nested-virtualization VM, provisions SR-IOV VFs, and runs a **testpmd** throughput test validating ≥10 Gbps per VF.

---

```
Copy this Markdown into `50_sriov_dpdk.md`; if you still prefer a ZIP, let me know and I’ll repackage with simplified tooling. |oai:code-citation|
```

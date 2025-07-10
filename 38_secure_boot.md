
# Chapter38  Secure Boot & TPM / Trusted Firmware Concepts

Attackers love boot-time hooks.  SecureBoot chains and TPMsealed keys ensure
firmware authenticity on edge routers and servers.

---

## 38.1  SecureBoot Chain

1. **ROM** verifies bootloader signature (public key fused).  
2. **Bootloader (UBoot)** verifies kernel image + DTB.  
3. **Kernel** verifies initrd or rootfs (dmverity).  
4. **Application** verifies firmware blob via OpenSSL.

---

## 38.2  Signing UBoot FIT Image

```bash
mkimage -f fit.its -k keys -K keys/pubkey.dtb -r fitImage
```

`fit.its` specifies hashes and key name.

On boot:

```
FIT: Verifying Hash Integrity ... OK
FIT: Verifying signature ... OK
```

---

## 38.3  TPM2 Sealing

Generate key:

```bash
tpm2_createprimary -C o -g sha256 -G rsa -c prim.ctx
tpm2_create -C prim.ctx -u key.pub -r key.priv -a "fixedtpm|fixedparent|sensitivedataorigin|userwithauth|decrypt|restricted"
tpm2_load -C prim.ctx -u key.pub -r key.priv -c key.ctx
```

Seal secret:

```bash
tpm2_create -C key.ctx -i secret.bin -u seal.pub -r seal.priv
```

Unseal only if PCR0PCR7 match expected measurements.

---

## 38.4  Bash Verification Flow on Router

```bash
if ! fwupdmgr verify fw.bin --key /etc/keys/pub.pem; then
  echo "Signature BAD" >&2; reboot -f
fi
```

Kernel command line includes `lsm=integrity`.

---

## 38.5  dmverity RootFS

Generate hash tree:

```bash
veritysetup format rootfs.img rootfs.hash
```

Boot args:

```
root=/dev/dm-0 ro dm="1 vroot verity 254:0 ..."
```

Device reboots readonly if block tampered.

---

## 38.6  Exercises

1. Script that reads PCR0PCR4 via `tpm2_pcrread` and logs to Prometheus.  
2. Automate key rotation: generate new FIT signing key, update bootloader
   public key via dualimage process.  
3. Integrate dmverity status check into keepalived health script; if
   integrity fails, decrease VRRP priority.

---



# Chapter 72  Measured Boot & IMA Policy for Script Integrity

Modern Secure Boot verifies the kernel, but once youre past GRUB, who protects
your **Bash, Python, and firmware binaries**?  **IMA** (Integrity Measurement
Architecture) plus TPM **PCR measurement** creates an attestation chain that
logs every executables hashallowing remote verification and policybased
refusal to run unsigned scripts.

_Last updated: 2025-07-10_

---

## 72.1  TPM & IMA Overview

1. **Measured Boot** extends hashes of firmware  bootloader  kernel into TPM PCRs 09.  
2. **IMA** measures or appraises files at `execve`.  
3. Remote verifier issues **TPM Quote** to validate PCRs + IMA log.

---

## 72.2  Kernel Config

* Kernel  5.10 with `CONFIG_IMA=y` `CONFIG_IMA_APPRAISE=y`
* TPM 2.0 device `/dev/tpmrm0`
* Boot parameters:

```
ima_appraise=fix ima_policy=tcb ima_template=ima-sig
```

---

## 72.3  Create IMA Key & Install

```bash
openssl req -new -x509 -nodes -days 3650 \
  -keyout ima.key -out ima.crt -subj "/CN=IMA-CA/"
openssl x509 -in ima.crt -outform DER -out /etc/ima/ima_cert.cer
keyctl padd asymmetric "" %keyring:.ima < /etc/ima/ima_cert.cer
```

---

## 72.4  Sign Scripts & Binaries

```bash
evmctl ima_sign --key ima.key --cert ima.crt /usr/local/bin/flash_fw.sh
evmctl ima_verify /usr/local/bin/flash_fw.sh   # sanity check
```

---

## 72.5  Sample IMA Policy `/etc/ima/ima-policy`

```
measure  func=BPRM_CHECK uid=0
appraise func=BPRM_CHECK mask=MAY_EXEC uid=0 path=/usr/local/bin/*
appraise func=FILE_CHECK mask=MAY_READ  path=/etc/flowspec/*
```

Reload:

```bash
echo "1" | sudo tee /sys/kernel/security/ima/policy
```

---

## 72.6  BootTime Evidence

Check IMA log:

```bash
ima_measure --csv | head
```

Last entry hash must match `flash_fw.sh`.

---

## 72.7  Remote Attestation Workflow

Device side:

```bash
nonce=$(openssl rand -hex 16)
tpm2_quote -C 0x81010001 -l sha256:10 -q $nonce \
  -m quote.bin -s sig.bin -o pcrs.bin
scp quote.bin sig.bin pcrs.bin ima.log verifier:
```

Verifier:

```bash
tpm2_checkquote -u key.pub -m quote.bin -s sig.bin -q $nonce -f pcrs.bin
ima-evm-verifier --pcr pcrs.bin --log ima.log --allowed-hash flash_fw.sh.sha256
```

---

## 72.8  Monitoring Violations

Add to rsyslog:

```
:msg, contains, "IMA: status=violation" action(type="omprog" binary="/usr/local/bin/alert_violation")
```

Script posts to Slack and Prometheus counter.

---

## 72.9  RealWorld Flow

1. CI signs each `.pex` & `.sh`.  
2. GitOps pushes artifacts; IMA refuses exec if unsigned.  
3. TPM quote uploaded to Vault before Ansible trafficshift job.

---

## 72.10  Exercises

1. Tune policy to `appraise` only when SELinux label `firmware_exec_t`.  
2. Create a test that copies unsigned script, tries to exec, and asserts exit1.  
3. Extend PCRs with `evmctl ima_hash` for Windows EXE in dualboot lab and verify via quote.

---

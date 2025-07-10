
# Chapter8  Cryptography & Secrets Management

Whether pushing firmware or backing up configs, strong crypto keeps customer
data and network integrity safe.  Bash can orchestrate OpenSSL, SSH keys,
GPG, and even hardware tokenswithout heavyweight PKI servers.

---

## 8.1  SSH Key Architecture for Fleet Automation

| Key Type | Bits | Why / When |
|----------|------|------------|
| **ed25519** | 256 | Fast, small, modern. Recommended default. |
| rsa4096 | 4096 | Legacy NOS that lack ellipticcurve support. |
| ecdsanistp256 | 256 | Lightweight CPUs (old routers) but weaker than ed25519. |

### Generating a hardened key

```bash
ssh-keygen -t ed25519 -C "fw-bot@isp" -f /opt/keys/fw_ed25519 -a 100
```

* `-a 100` increases KDF rounds  resists offline key cracking.

### ForcedCommand for Least Privilege

`~/.ssh/authorized_keys` on device:

```
command="/usr/bin/fwupdate apply /tmp/fw.bin",from="10.0.0.0/8" ssh-ed25519 AAAA...
```

Key can **only** run firmware update; cant open shell.

---

## 8.2  OpenSSL  Signing & Verifying Firmware

### Create signing keypair

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out fw_priv.pem
openssl rsa -pubout -in fw_priv.pem -out fw_pub.pem
```

### Sign firmware blob

```bash
openssl dgst -sha256 -sign fw_priv.pem -out fw.sig firmware.bin
```

### Verify on target device

```bash
openssl dgst -sha256 -verify fw_pub.pem -signature fw.sig firmware.bin
```

Automate in Bash:

```bash
if openssl dgst -sha256 -verify /etc/keys/fw_pub.pem      -signature /tmp/fw.sig /tmp/fw.bin; then
   fwupdate apply /tmp/fw.bin
else
   echo "Signature invalid!" >&2; exit 2
fi
```

---

## 8.3  GPG for Config Backups

### Symmetric Encrypt

```bash
tar czf configs.tar.gz /flash/config
gpg -c --cipher-algo AES256 configs.tar.gz
rm configs.tar.gz
```

Password prompt can be automated with `--batch --passphrase-file` in CI.

### Decrypt

```bash
gpg -d configs.tar.gz.gpg >configs.tar.gz
```

*Why symmetric?* Devices lack user keypairs; a shared rotation password is
simpler for emergency restore.

---

## 8.4  YubiKey / PIV Hardware Signing

Enroll slot9c:

```bash
yubico-piv-tool -a generate -s 9c -A RSA2048 -o pub.pem
yubico-piv-tool -a verify-pin -a selfsign-certificate -s 9c -S '/CN=FW Signer' -i pub.pem -o cert.pem
```

Use in script:

```bash
yubico-piv-tool -a verify-pin -a sign -s 9c   -i firmware.bin -o firmware.sig
```

Touchtosign prevents malware autoload.

---

## 8.5  Secrets in `.env` + GPG Decrypt

`secrets.env.gpg` contains:

```
API_TOKEN=xyz
DB_PASS=supersecret
```

Load in script:

```bash
set -a                             # export all
gpg -d /etc/secure/secrets.env.gpg | source /dev/stdin
set +a
```

Environment now has `$API_TOKEN` without writing unencrypted file.

---

## 8.6  Hashicorp Vault (CLI Glimpse)

Authentication:

```bash
VAULT_TOKEN=$(cat /run/secrets/vault-token)
vault kv get -format=json secret/fw | jq -r .data.data.sign_key >fw_priv.pem
```

Use `vault ssh` for onetime cert authno permanent keys on disk.

---

## 8.7  Encrypting Backups with OpenSSL AES256

Bandwidthefficient backup over SSH:

```bash
tar cz /flash/config | openssl enc -aes-256-cbc -salt -k "$PASS" |    ssh storage 'cat >backup/$(hostname)-cfg.enc'
```

Decrypt offline:

```bash
openssl enc -d -aes-256-cbc -in HOST-cfg.enc -out HOST-cfg.tar.gz
```

---

## 8.8  Checksums & Integrity

Generate combined manifest:

```bash
sha256sum *.bin >MANIFEST.sha256
scp MANIFEST.sha256 *.bin tftp:/
```

Verify on device:

```bash
sha256sum -c MANIFEST.sha256 --ignore-missing
```

---

## 8.9  Exercise

1. Write a Bash function `verify_firmware` that downloads
   `fw.bin` + `fw.sig`, fetches the correct public key based on model, and
   returns 0 only if signature AND SHA256 match.  
2. Automate rotating GPG symmetric passphrase every quarter via Vault and
   reencrypting all backups.  
3. Build a YubiKeyprotected Git signing workflow for commit integrity of
   upgrade scripts.

---

> Next: **Chapter9  Full Playbooks & Blue/Green Rollouts**

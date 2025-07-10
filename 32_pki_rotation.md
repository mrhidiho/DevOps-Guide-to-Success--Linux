
# Chapter32  PKI Rotation & Hardware Security Module (HSM) Integration

Periodic key rotation and secure signing hardware are mandatory for
carriergrade security standards (PCI, SOC2).  This chapter extends Chapter8
by automating SSH certificate authorities (CA) with Hashicorp Vault and
firmwaresigning with **YubiHSM2**.

---

## 32.1  Vault SSH Certificate Authority

### Enable SSH secrets engine

```bash
vault secrets enable ssh
vault write ssh/config/ca generate_signing_key=true
vault write ssh/roles/edge      key_type=ca      allow_user_certificates=true      allowed_users="admin"      default_extensions='{"permit-pty":""}'      ttl=60s
```

### Bash wrapper to fetch 1minute cert

```bash
sign_cert(){
  vault write -field=signed_key ssh/sign/edge      public_key=@$HOME/.ssh/id_ed25519.pub >$HOME/.ssh/id_ed_cert
}
```

SSH with cert:

```bash
ssh -i ~/.ssh/id_ed25519 -i ~/.ssh/id_ed_cert admin@$ip
```

---

## 32.2  YubiHSM2 Firmware Signing

### Setup Slot

```bash
yubihsm-shell -a put-asymmetric-key -i 0x1234 -l "FW sign"    -p 1 -c sign-pkcs,sign-pss -A rsa2048
```

### Signing Script (`sign_fw.sh`)

```bash
fw=$1
yubihsm-shell -a sign-pss -i 0x1234 -p "1234" -m SHA256 -s $fw >$fw.sig
```

Verify (public key exported once).

---

## 32.3  Key Rotation Pipeline

1. Rotate signing key quarterly in Vault: `vault write ssh/config/ca rotate=true`.  
2. Issue new host certs via Ansible playbook.  
3. Revoke old CA by updating `known_hosts_certificate`.  
4. Update YubiHSM key label and archive old key under HSM audit policy.

---

## 32.4  Certificate Transparency Log

Push firmware signatures to CT-style log for audit:

```bash
echo "$(sha256sum fw.bin) $(date +%s)" >>fw_sig.log && git commit -a -m "sig"
```

---

## 32.5  Exercises

1. Write a Bash script to renew SSH cert every 45seconds during long
   firmware rollout (background daemon).  
2. Create a YubiHSM audit policy that only allows `sign-pss` by `fw-bot`
   operator.  
3. Implement a Vault sentinel policy rejecting cert sign requests from IPs
   outside build pipeline.

---

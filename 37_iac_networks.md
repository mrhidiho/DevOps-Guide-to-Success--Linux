
# Chapter37  InfrastructureasCode for Networks (Terraform + Ansible)

Treat VLANs, ACLs, and BGP neighbors like codereviewed, versioned, and
automatically applied.  This chapter combines **Terraform** (declarative
state) with **Ansible** (imperative steps) for ISPscale network automation.

---

## 37.1  Terraform Provider Options

| Provider | Devices | Notes |
|----------|---------|-------|
| `hashicorp/network` | Generic SNMP/CLI | Community |
| `Cisco/cdcloud` | IOSXE/IOSXR | Requires API |
| `Juniper/junos` | Junos via NETCONF | Supports commitconfirm |
| `arista/avd` | EOS CloudVision | gRPC eAPI |

If no vendor provider exists, wrap **Ansible** modules via `local-exec`.

---

## 37.2  Sample `network.tf`

```hcl
variable "edge_ips" { type = list(string) }

resource "junos_interface" "uplinks" {
  for_each = toset(var.edge_ips)
  name     = "ge-0/0/0"
  unit     = 0
  vlan_tagging = true
  description  = "Uplink to core (${each.key})"
}
```

`terraform plan` shows diff; `terraform apply` pushes via NETCONF.

---

## 37.3  State Storage & Locking

* **Backend S3** with DynamoDB lock tableprevents double apply.  
* Encrypt state (`bucket_key_enabled = true`) for ACL secrets.

---

## 37.4  Ansible Role Handoff

Use Terraform to **create inventory** then call Ansible:

```hcl
resource "local_file" "inventory" {
  content = join("
", var.edge_ips)
  filename = "${path.module}/inventory"
}

resource "null_resource" "ansible_cfg_push" {
  triggers = { inv_sha = sha1(local_file.inventory.content) }
  provisioner "local-exec" {
    command = "ansible-playbook -i inventory playbooks/push_cfg.yml"
  }
}
```

---

## 37.5  GitLab CI Stages

```yaml
stages: [validate, plan, apply]

validate:
  image: hashicorp/terraform:1.8
  script: terraform fmt -check && terraform validate

plan:
  script: terraform plan -out=plan.tfplan
  artifacts: { paths: [plan.tfplan] }

apply:
  when: manual
  script: terraform apply plan.tfplan
```

---

## 37.6  Drift Detection Job

Nightly pipeline:

```bash
terraform plan -detailed-exitcode
case $? in
  0) echo "No drift";;
  2) echo "Drift detected"; exit 2;;
esac
```

---

## 37.7  Exercises

1. Write a Terraform module that instantiates VRRP pairs with matching
   priority and preempts.  
2. Add an Ansible `assert` task verifying `show ip bgp summary` after
   Terraform apply.  
3. Use `terraform taint` to forcereplace a linkdown interface and reapply.

---


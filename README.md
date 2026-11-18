# blog-iscsi-initiator-ansible

Companion code for the blog post
[Automating iSCSI Initiator Configuration with Ansible](https://blog.bsdserver.nl/automating-iscsi-initiator-configuration-with-ansible/).

It builds both ends of the setup from the post:

- **storage01**: a Linux LIO iSCSI target (`targetcli-fb`) that exports its second disk as LUN 0,
  with one ACL per initiator and one-way CHAP.
- **node01**: an open-iscsi initiator configured by the playbook from the post (packaged as the
  `iscsi_initiator` role): `iscsid.conf` with CHAP, a unique IQN `iqn.2025-03.com.example:<hostname>`,
  discovery, login and automatic login at boot.

Terraform creates the two Ubuntu 24.04 VMs on Proxmox VE; Ansible configures them.

## Requirements

- Terraform >= 1.9 and a Proxmox VE API token (the bpg/proxmox provider, pinned in `terraform/versions.tf`).
  The Proxmox datastore for the cloud image needs the `import` content type.
- Ansible (ansible-core 2.17 or later; tested with 2.22). Only `ansible.builtin` modules are used.
- Ubuntu 24.04 on the target and the initiators.

## Quick start

```bash
# 1. VMs
cd terraform
cp terraform.tfvars.example terraform.tfvars   # set endpoint, token, node, IPs, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Secrets
cd ../ansible
cp group_vars/all/vault.yml.example group_vars/all/vault.yml   # set the CHAP username and password
ansible-vault encrypt group_vars/all/vault.yml

# 3. Configure target and initiator, then run the checks from the post
ansible-playbook site.yml --ask-vault-pass
ansible-playbook verify.yml --ask-vault-pass
```

No Proxmox? Skip step 1 and copy `ansible/inventory.example.yml` to `ansible/inventory.yml` with
your own hosts. The target needs an unused disk for the LUN.

## Variables

Set in `ansible/group_vars/all/vars.yml`:

| Variable | Default | Meaning |
| --- | --- | --- |
| `iscsi_target_ip` | `10.0.10.11` | Portal the initiators log in to (from the post) |
| `iscsi_target_name` | `iqn.2025-03.com.example:storage` | Target IQN (from the post) |
| `iscsi_chap_username` / `iscsi_chap_password` | from vault | CHAP credentials (`vault_iscsi_chap_*` in `vault.yml`) |
| `iscsi_target_luns` | `/dev/sdb` as `lun0` | Block devices exported by the target, with a fixed `wwn` (UUID) |
| `iscsi_target_initiators` | one IQN per host in `iscsi_initiators` | Target ACLs |
| `iscsi_target_chap_enabled` | `true` | One-way CHAP on every ACL |

The LUN's WWID on the initiator is `36001405` followed by the first 25 hex digits of the `wwn`
without dashes (`36001405c0ffee0012344abc8def01234` for the default), and it stays the same every
time the target configuration is applied.

The target role writes `/etc/rtslib-fb-target/saveconfig.json` and lets the
`rtslib-fb-targetctl` service restore it (also at boot). Changes made with `targetcli` by hand are
overwritten on the next run.

Note: the target host must not use the LUN disk itself. If the initiators put LVM on the LUN,
the target sees those PVs on its backing disk as well; keep the target out of them (for example with
an LVM `global_filter` that rejects the backing device).

## Tested

Lab-tested on Ubuntu 24.04 (kernel 6.8) with ansible-core 2.22: the first run configures both ends,
the second run reports `changed=0`, and `verify.yml` (the checks from the post) passes. Terraform is
checked with `fmt -check`, `validate` and `tflint`; `terraform apply` was not run as part of these tests.

## License

MIT

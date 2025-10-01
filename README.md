# homelab‑infra

[![Lint](https://github.com/cchacon-dev/homelab-infra/actions/workflows/lint.yml/badge.svg)](https://github.com/cchacon-dev/homelab-infra/actions/workflows/lint.yml)
[![Molecule](https://github.com/cchacon-dev/homelab-infra/actions/workflows/molecule.yml/badge.svg)](https://github.com/cchacon-dev/homelab-infra/actions/workflows/molecule.yml)

IaC repository to bootstrap and configure a bare‑metal homelab using **Ansible**. The goals are:

* Reproducible, production‑like setup on low‑cost hardware
* Idempotent provisioning via roles
* Easy horizontal growth (add a host → run one command)
* Separation of concerns: this repo is **infra config**; **GitOps/Kubernetes** will live in a separate repo

> ⚡️ **Note**: This is my **first Ansible project**, part of a personal **homelab** to learn and practice **DevSecOps**. Any suggestions or improvements are very welcome!

---

## Tooling & stack

[![Ansible](https://img.shields.io/badge/Ansible-Automation-black?logo=ansible)](https://www.ansible.com/)
[![Molecule](https://img.shields.io/badge/Molecule-Role%20testing-blue)](https://molecule.readthedocs.io/)
[![ansible‑lint](https://img.shields.io/badge/ansible--lint-QA-success)](https://ansible.readthedocs.io/projects/lint/)
[![yamllint](https://img.shields.io/badge/yamllint-style-green)](https://yamllint.readthedocs.io/)
[![pre‑commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit)](https://pre-commit.com/)

---

## Pre‑steps (do these once per machine)

Before using this repo you should have a fresh **Ubuntu Server 22.04/24.04 LTS** install on each node.

1. **Install Ubuntu Server** (minimal is fine) and set:

   * A hostname, e.g. `cp01-homelab`, `wk01-homelab`
   * A static or DHCP‑reserved IP

2. **Create the `ansible` user with sudo (NOPASSWD)** and a known password (we'll store it in Vault for the first bootstrap):

```bash
# Login as your initial admin, then:
sudo adduser ansible
sudo usermod -aG sudo ansible
# Optional: allow passwordless sudo for automation
echo 'ansible ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/90-ansible >/dev/null
sudo chmod 440 /etc/sudoers.d/90-ansible

# Set a password (temporarily; it will be stored encrypted in Ansible Vault)
sudo passwd ansible
```

3. **Install your SSH public key for the `ansible` user** (recommended):

```bash
# On your control machine
ssh-copy-id ansible@<host-ip>
```

> Rationale: the first bootstrap may use the **become password** from Vault while keys propagate. After the bootstrap role installs the public key and hardens SSH, you can disable password authentication for SSH entirely.

4. **Store secret values in Ansible Vault** (on your control machine):

```bash
# Create the vault file for production inventory
ansible-vault create ansible/inventory/prod/group_vars/all/vault.yml
```

Add at least:

```yaml
---
# Encrypted with Ansible Vault
ansible_user: ansible
ansible_become_password: "<the password you set for ansible user>"
# Add any other secrets (API tokens, Cloudflare, etc.) here
```

> Tip: If you use pre‑commit with linters, exclude `vault.yml` from yamllint/ansible‑lint to avoid decryption warnings.

---

## Repository layout

```
ansible/
  inventory/
    prod/
      hosts.yml            # inventory of production hosts
      group_vars/
        all/
          main.yml         # non‑secret defaults for all hosts
          vault.yml        # secrets (encrypted with Ansible Vault)
  playbooks/
    site.yml               # main entry point
    bootstrap.yml          # first‑run bootstrap (SSH keys, hardening, etc.)
  roles/
    base/                  # example role (common baseline)
      tasks/
      templates/
      files/

collections/
  requirements.yml         # optional Ansible Galaxy collections

.molecule/                 # optional global molecule config
.pre-commit-config.yaml
.ansible-lint
.yamllint
```

---

## Quick start

1. **Install local tooling**

```bash
# Python tooling
pipx install ansible-core
pipx install ansible-lint
pipx install molecule
pipx install yamllint
pipx inject molecule docker  # if using Docker driver

# Pre-commit hooks (recommended)
pipx install pre-commit
pre-commit install
```

2. **Install Ansible collections (if any)**

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

3. **Set your inventory** in `ansible/inventory/prod/hosts.yml`:

```yaml
---
all:
  children:
    control_plane:
      hosts:
        cp01-homelab:
          ansible_host: 192.168.1.10
    workers:
      hosts:
        wk01-homelab:
          ansible_host: 192.168.1.11
        wk02-homelab:
          ansible_host: 192.168.1.12
```

4. **First run (bootstrap)** — simulates what will change:

```bash
ansible-playbook ansible/bootstrap.yml --check --diff -i ansible/inventory/prod/hosts.yml \
  --ask-vault-pass
```

Then apply for real:

```bash
ansible-playbook ansible/bootstrap.yml -i ansible/inventory/prod/hosts.yml \
  --ask-vault-pass
```

5. **Main configuration**

Dry‑run first:

```bash
ansible-playbook ansible/playbooks/site.yml --check --diff -i ansible/inventory/prod/hosts.yml \
  --ask-vault-pass
```

Apply:

```bash
ansible-playbook ansible/playbooks/site.yml -i ansible/inventory/prod/hosts.yml \
  --ask-vault-pass
```

> After bootstrap, consider disabling SSH password auth in your role to harden nodes.

---

## Testing & quality

**Lint everything**

```bash
ansible-lint
yamllint .
```

**Molecule (per role)**

```bash
cd ansible/roles/base
molecule test
```

**Pre‑commit** (runs linters before each commit)

```bash
pre-commit run -a
```

> If you keep `vault.yml` encrypted, add it to excludes in `.yamllint` and `.ansible-lint` to avoid false positives.

---

## Reusing this repo (change only hosts)

1. Fork/clone the repo
2. Edit `ansible/inventory/prod/hosts.yml` with your hostnames/IPs
3. Create `group_vars/all/vault.yml` with your secrets via Vault
4. Run `bootstrap.yml` then `site.yml`

This pattern scales: add a new host under `workers:` and re‑run.

---

## Production‑like choices (on a budget)

* **Idempotent roles**: everything configured via tasks, not manual tweaks
* **Inventory per environment** (we keep only `prod/` now, but structure allows `dev/` in the future)
* **SSH key auth** and **passwordless sudo** for automation; then **disable PasswordAuthentication** in SSH
* **Pre‑commit** gates to keep the repo healthy
* **Secrets in Vault** (never commit plaintext secrets)

---

## Secrets management (Vault quick notes)

```bash
# create or edit
ansible-vault create ansible/inventory/prod/group_vars/all/vault.yml
ansible-vault edit   ansible/inventory/prod/group_vars/all/vault.yml

# encrypt existing
ansible-vault encrypt ansible/inventory/prod/group_vars/all/vault.yml

# run playbooks
ansible-playbook ... --ask-vault-pass
```

Example variables (inside the vault):

```yaml
---
ansible_user: ansible
ansible_become_password: "<secret>"
cloudflare_api_token: "<secret>"
```

> Do not store private keys in the repo. Keep them on the control machine and distribute public keys via roles.

---

## Troubleshooting

* **`Decryption failed` in linters** → exclude `vault.yml` from yamllint/ansible‑lint
* **`Permission denied (publickey)`** → run bootstrap again or ensure your `authorized_keys` task is applied
* **`Failed to become root`** → ensure `ansible_become_password` is set in Vault or sudoers allows NOPASSWD

---

## Roadmap

* Add roles: `k3s`, `docker`, `common-hardening`, `monitoring`
* Separate GitOps repo for cluster state (FluxCD)

---

## License

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

MIT © Carlos (homelab‑infra)

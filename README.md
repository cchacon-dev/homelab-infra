# 🧩 homelab-infra

[![Lint](https://github.com/cchacon-dev/homelab-infra/actions/workflows/lint.yml/badge.svg)](https://github.com/cchacon-dev/homelab-infra/actions/workflows/lint.yml)
[![Molecule](https://github.com/cchacon-dev/homelab-infra/actions/workflows/molecule.yml/badge.svg)](https://github.com/cchacon-dev/homelab-infra/actions/workflows/molecule.yml)

Infrastructure-as-Code repository to bootstrap and configure a **bare-metal homelab** using **Ansible**.
The project focuses on building a reproducible, production-like environment on low-cost hardware — fully automated and modular.

> ⚡️ This is part of my personal **DevOps learning journey**.
> It aims to explore **Ansible**, **K3s**, and **GitOps** workflows for real-world homelab operations.

---

## 🎯 Goals

- Reproducible configuration across all nodes
- Idempotent, role-based provisioning
- Seamless horizontal scaling (add a node → run one command)
- Clear separation of concerns:
  - **This repo:** Infrastructure provisioning
  - **Future repo:** GitOps-based cluster state (FluxCD, manifests, etc.)
- Easy recovery workflows, including **worker reset when the control plane is replaced**

---

## 🧰 Tooling & Stack

[![Ansible](https://img.shields.io/badge/Ansible-Automation-black?logo=ansible)](https://www.ansible.com/)
[![Molecule](https://img.shields.io/badge/Molecule-Role%20Testing-blue)](https://molecule.readthedocs.io/)
[![ansible-lint](https://img.shields.io/badge/ansible--lint-QA-success)](https://ansible.readthedocs.io/projects/lint/)
[![yamllint](https://img.shields.io/badge/yamllint-style-green)](https://yamllint.readthedocs.io/)
[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit)](https://pre-commit.com/)

**Current stack:**
- Ubuntu Server 22.04 / 24.04 LTS
- K3s (lightweight Kubernetes)
- Ansible roles for base setup, hardening, and networking (Cilium coming soon)
- GitHub Actions CI for linting and Molecule testing

---

## 🏗️ Project Layout

```
ansible/
  inventory/
    prod/
      hosts.yml
      group_vars/
        all/
          main.yml
          vault.yml              # Encrypted secrets (Ansible Vault)
  playbooks/
    bootstrap.yml                # First-run setup (SSH keys, sudo, hardening)
    site.yml                     # Main configuration for all nodes
    reset_workers.yml            # Reset K3s workers after CP reinstall
  roles/
    base/                        # Common baseline role
      tasks/
      templates/
      files/
collections/
  requirements.yml

.pre-commit-config.yaml
.ansible-lint
.yamllint
```

---

## ⚙️ Pre-Setup

Perform these steps **once per node** (control plane and workers):

1. **Install Ubuntu Server (minimal)**
   Assign a hostname like `cp01-homelab`, `wk01-homelab`, etc.

2. **Create the `ansible` user** with sudo privileges:

   ```bash
   sudo adduser ansible
   sudo usermod -aG sudo ansible
   echo 'ansible ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/90-ansible >/dev/null
   sudo chmod 440 /etc/sudoers.d/90-ansible
   sudo passwd ansible
   ```

3. **Copy your SSH public key:**

   ```bash
   ssh-copy-id ansible@<host-ip>
   ```

4. **Create a Vault file** for secrets:

   ```bash
   ansible-vault create ansible/inventory/prod/group_vars/all/vault.yml
   ```

   ```yaml
   ---
   ansible_user: ansible
   ansible_become_password: "<ansible password>"
   ```

---

## 🚀 Quick Start

1. **Install local tooling**

   ```bash
   pipx install ansible-core ansible-lint molecule yamllint pre-commit
   pipx inject molecule docker
   pre-commit install
   ```

2. **Install Ansible collections**

   ```bash
   ansible-galaxy collection install -r collections/requirements.yml
   ```

3. **Set your inventory**

   ```yaml
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

4. **Bootstrap (first run)**

   ```bash
   ansible-playbook ansible/playbooks/bootstrap.yml -i ansible/inventory/prod/hosts.yml --ask-vault-pass
   ```

5. **Main configuration**

   ```bash
   ansible-playbook ansible/playbooks/site.yml -i ansible/inventory/prod/hosts.yml --ask-vault-pass
   ```

---

## 🔁 Maintenance & Recovery

### 🔹 Reset Workers

If the control plane is reinstalled or replaced, you can **safely clean and re-join the worker nodes**:

```bash
ansible-playbook ansible/playbooks/reset_workers.yml -i ansible/inventory/prod/hosts.yml
```

This playbook:
- Stops the `k3s-agent` service
- Runs the uninstall script if available
- Removes all K3s directories and binaries
- Reloads systemd for a clean state

Afterwards, simply rerun your main playbook:

```bash
ansible-playbook ansible/playbooks/site.yml -i ansible/inventory/prod/hosts.yml --ask-vault-pass
```

---

## 🧪 Testing & Quality

- **Lint all YAML and roles**
  ```bash
  ansible-lint
  yamllint .
  ```

- **Run Molecule tests per role**
  ```bash
  cd ansible/roles/base
  molecule test
  ```

- **Run pre-commit hooks**
  ```bash
  pre-commit run -a
  ```

> Secrets files (`vault.yml`) should be excluded from linting to avoid decryption errors.

---

## 🧱 Production-like Choices (on a budget)

- Idempotent, self-documenting roles
- Passwordless sudo + SSH key-only access
- Secrets stored in **Ansible Vault**
- Consistent structure per environment (`prod`, future `dev`)
- CI lint + Molecule for every role

---

## 🗺️ Roadmap

- [ ] Add `docker` and `k3s` roles
- [ ] Add `cilium` for networking
- [ ] Implement `fluxcd` GitOps repo (separate project)
- [ ] Centralized monitoring stack

---

## 📜 License

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

MIT © Carlos — *homelab-infra*

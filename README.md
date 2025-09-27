# Homelab Infrastructure (Ansible Base)

[![Lint](https://github.com/cchacon-dev/homelab-infra/actions/workflows/lint.yml/badge.svg)](https://github.com/cchacon-dev/homelab-infra/actions/workflows/lint.yml)
[![Molecule](https://github.com/cchacon-dev/homelab-infra/actions/workflows/molecule.yml/badge.svg)](https://github.com/cchacon-dev/homelab-infra/actions/workflows/molecule.yml)

Infrastructure-as-Code for my **personal homelab**, currently running on **2 bare‑metal nodes** (soon **3**).
This repository focuses on **base provisioning with Ansible** (packages, time, SSH, firewall, kernel/sysctl for k8s, GitOps-ready structure).
> **Note:** **GitOps (manifests/apps)** will live in a **separate repository** (coming soon) so that runtime configuration and application delivery are cleanly decoupled.

---

## Why
- Avoid manual provisioning and enable **repeatable** and **scalable** changes.
- Practice **DevOps** workflows (Ansible, Molecule, pre-commit, CI).
- Keep the base layer minimal and production-like; layer **GitOps** on top in a dedicated repo.

---

## Scope & Separation
- **This repo (Ansible Base):** OS/base config across nodes (apt, timezone, SSH, UFW, swap/sysctl/modules for Kubernetes readiness, GitOps root).
- **GitOps repo (COMING SOON):** cluster state & apps (Flux/Argo CD, Helm releases, workloads, policies).
  _Placeholder_: `https://github.com/cchacon-dev/homelab-gitops` (to be published)

---

## Features
- Base role with:
  - Update/upgrade, **base packages**.
  - Timezone & timesync.
  - Swap off (where applicable), **sysctl**, and kernel modules for Kubernetes.
  - SSH hardening (authorized keys, disable password auth).
  - GitOps root directory.
  - Optional UFW rules.
- **Pre-commit** hooks: `yamllint`, `ansible-lint`.
- **GitHub Actions**: Lint + Molecule pipelines.
- **Molecule** tests for the base role (Docker driver).

---

## Repository Layout (excerpt)
```
ansible/
  playbooks/
    site.yml
  roles/
    base/
      tasks/
      handlers/
      molecule/
        default/
          molecule.yml
          converge.yml
  inventory/
    prod/
      hosts.yml
      group_vars/
        all.yml
        vault.yml
collections/
  requirements.yml
.github/
  workflows/
    lint.yml
    molecule.yml
.pre-commit-config.yaml
.ansible-lint
.yamllint
.editorconfig
```

---

## Requirements
- Control machine: Python **3.12+**.
- Ansible **10+**.
- (Optional) Docker for local Molecule tests.
- Target nodes: Linux (tested on **Ubuntu 22.04**).

Install local tooling:
```bash
python3 -m pip install --upgrade pip
pip install "ansible>=10" "molecule>=24.6.0" "molecule-plugins[docker]>=23.5.0" "pytest>=7" "docker>=7"
ansible-galaxy collection install -r collections/requirements.yml -p ~/.ansible/collections
export ANSIBLE_COLLECTIONS_PATH=~/.ansible/collections
```

---

## Quick Start (Reuse with your own hosts)

1) **Clone**
```bash
git clone https://github.com/cchacon-dev/homelab-infra.git
cd homelab-infra
```

2) **Inventory: set your hosts**
Edit `ansible/inventory/prod/hosts.yml`:
```yaml
---
all:
  hosts:
    cp01-homelab:
      ansible_host: 192.168.1.101
    wk01-homelab:
      ansible_host: 192.168.1.102
# add third node later:
#   wk02-homelab:
#     ansible_host: 192.168.1.103
```

3) **Group vars: base settings**
Edit `ansible/inventory/prod/group_vars/all.yml`:
```yaml
---
base_timezone: "America/Costa_Rica"
base_gitops_root: "/srv/gitops"
base_manage_ufw: false

base_k8s_sysctl:
  net.ipv4.ip_forward: 1
  net.bridge.bridge-nf-call-iptables: 1
  net.bridge.bridge-nf-call-ip6tables: 1

base_packages:
  - curl
  - vim
  - htop
  - jq
```

4) **(Optional) Secrets**
If you need secrets now, either encrypt `vault.yml`:
```bash
ansible-vault encrypt ansible/inventory/prod/group_vars/vault.yml
```
or keep it as an empty dictionary:
```yaml
---
{}
```

5) **Run the base playbook**
```bash
ansible-playbook ansible/playbooks/site.yml -i ansible/inventory/prod/hosts.yml
```

That’s it — your nodes get the base configuration.
When the **GitOps repo** is published, you’ll point Flux/Argo CD at that repo to deploy apps.

---

## Local Testing (Molecule)
From the role directory:
```bash
cd ansible/roles/base
molecule test
# or:
molecule create && molecule converge && molecule login
```

---

## CI/CD
- **Lint** workflow runs pre-commit hooks across the repository.
- **Molecule** workflow runs Docker-based role tests.
- Branch protection (recommended) requires both checks to pass before merge.

---

## Roadmap / TODO
- [ ] Publish the separate **GitOps** repository (Flux/Argo CD bootstrap + apps).
- [ ] Role for k3s/kubeadm install (control-plane & workers).
- [ ] Container runtime role (containerd).
- [ ] Monitoring/Observability stack (Prometheus, Grafana, Loki).
- [ ] Ingress, DNS, TLS (Cloudflare, cert-manager).
- [ ] Example WordPress deployment via GitOps.
- [ ] Hardening: SSH, firewall profiles, auditd.

---

## Contributing
This is part of my learning journey. Suggestions and issues are welcome.
For now I’m the sole maintainer — PRs may be reviewed at my pace.

---

## License
MIT © [cchacon-dev](https://github.com/cchacon-dev)

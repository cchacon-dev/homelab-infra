# Ansible & GitOps Style Guide (homelab)

## Goals
- Consistent, enterprise-ready conventions while keeping the project approachable (first Ansible project).
- Readable tasks, idempotent commands, minimal ambiguity.

## Ansible
- **Task names**: Imperative, concise, prefixed with role context.
  - Example: `Base | Update apt cache`
- **Handlers**: Past-tense description.
  - Example: `Restart containerd`
- **Loops**: Prefer `loop:` over `with_*`.
- **Includes**: Prefer `import_tasks` for static, `include_tasks` for dynamic.
- **Booleans**: Use `true/false` (not yes/no).
- **Command/Shell**: Prefer modules; when unavoidable, add `creates:`/`removes:` or `changed_when:`.
- **Kubernetes**: Use FQCN `kubernetes.core.k8s` and `kubernetes.core.k8s_info`.
- **Variables**: `snake_case`; defaults in `roles/<role>/defaults/main.yml`; required vars documented in role README.
- **Vault**: Keep only *encrypted* values in `group_vars/*/vault.yml` or `secrets.yml` with `ansible-vault` and `--vault-id` usage.

## GitOps (Flux)
- **APIs**: use stable `v1` APIs (`source.toolkit.fluxcd.io/v1`, `kustomize.toolkit.fluxcd.io/v1`, `helm.toolkit.fluxcd.io/v2`).
- **Secrets**: Avoid plaintext `Secret` in Git. Prefer SOPS or ExternalSecrets.
- **Structure**: `clusters/<name>/kustomization.yaml` -> references `apps/*` and `infra/*` with clear layering.
- **Naming**: `<team>-<app>-<env>` or `<stack>-<component>` where relevant.

## Commit messages
- Conventional Commits subset:
  - `feat(role/base): ...`
  - `fix(role/k3s): ...`
  - `docs(gitops): ...`
  - `refactor(ansible): ...`

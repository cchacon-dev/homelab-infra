# Comments Convention

- Use full-line comments at top of task files to explain *why*, not *what*.
- For tasks, keep `- name:` under ~60 chars; details go in module args.
- When using `shell:` explain idempotence guards:
  ```yaml
  - name: Base | Initialize cluster token (idempotent)
    shell: |
      set -euo pipefail
      k3s token create --print > /var/lib/rancher/k3s/server/token
    args:
      creates: /var/lib/rancher/k3s/server/token
  ```
- When templating kube manifests, annotate with owner and change rationale:
  ```yaml
  metadata:
    annotations:
      owner: homelab
      rationale: allow kubelet metrics scrape
  ```

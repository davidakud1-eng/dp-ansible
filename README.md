# dp-ansible

Ansible configuration for the DP platform: three EC2 hosts running a
self-hosted CI/CD toolchain (Jenkins, Artifactory, a build agent), plus
the Kubernetes workloads (application, ingress, monitoring) deployed onto
an ephemeral Amazon EKS cluster.

This repository is one of three that make up the DP platform:

| Repository | Responsibility |
|---|---|
| [`dp-platform-infra`](../dp-platform-infra) | AWS infrastructure (VPC, EC2, IAM, EKS) via Terraform |
| `dp-ansible` (this repo) | Host configuration, CI/CD toolchain, and cluster workloads |
| [`dp-mywebapp`](../dp-mywebapp) | The application deployed onto the platform |

---

## Documentation

- [`docs/architecture.md`](docs/architecture.md) — the overall system
  picture: what runs where, and why
- [`docs/network.md`](docs/network.md) — VPC layout, security groups, EKS
  access path, and the two SSH trust chains
- [`docs/pipeline.md`](docs/pipeline.md) — how all three Jenkins pipelines
  work, including the JCasC bootstrap mechanism
- [`docs/roles.md`](docs/roles.md) — a one-line reference for every role
  in this repository

Start with `architecture.md` if you're new to this repo — everything else
links back to it.

---

## Repository layout

```
dp-ansible/
├── ansible.cfg
├── requirements.yml          # ansible.posix, kubernetes.core collections
├── inventory/
│   ├── hosts-dev.yml         # local control node (e.g. WSL)
│   ├── hosts-prod.yml        # platform-ops as control node
│   └── group_vars/all/       # shared vars + Ansible Vault secrets
├── playbooks/
│   ├── site.yml              # full platform, all hosts
│   ├── jenkins_controller.yml
│   ├── build_agent.yml
│   └── platform_ops.yml
├── roles/                    # see docs/roles.md
└── docs/
```

---

## Prerequisites

- Ansible (installed via the `ansible` role itself on `platform-ops`; on
  a local control node, install per your platform)
- SSH access to the target hosts, and an Ansible Vault password for the
  secrets in `inventory/group_vars/all/vault.yml`
- The collections in `requirements.yml`:

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

---

## Running the playbooks

Full platform, against the dev inventory (local control node):

```bash
ansible-playbook -i inventory/hosts-dev.yml playbooks/site.yml \
  --vault-password-file .vault-pass
```

A single host group:

```bash
ansible-playbook -i inventory/hosts-dev.yml playbooks/platform_ops.yml \
  --vault-password-file .vault-pass
```

A single role, via tags — useful for validating one component without
re-running the whole platform:

```bash
ansible-playbook -i inventory/hosts-dev.yml playbooks/platform_ops.yml \
  --vault-password-file .vault-pass \
  --tags kube_prometheus_stack
```

EKS-dependent roles (the AWS Load Balancer Controller,
`kube_prometheus_stack`) are gated behind the `eks_runtime_enabled`
variable, which defaults to `false`. Pass
`--extra-vars "eks_runtime_enabled=true"` to include them.

In CI, all of this runs via the `Jenkinsfile` in this repo — see
[`docs/pipeline.md`](docs/pipeline.md) for the full pipeline breakdown.

---

## A few design decisions worth knowing about before you dig in

- **EKS is ephemeral by design.** The cluster, its node group, and the
  NAT Gateway it needs don't exist unless `eks_runtime_enabled` (mirrored
  as `enable_eks_runtime` on the Terraform side) is explicitly turned on
  — see `docs/architecture.md` §4.
- **Two separate SSH identities**, not one shared key, because Jenkins
  acting as CI orchestrator and Platform Ops acting as the Ansible
  control node are different roles with different blast radii — see
  `docs/network.md` §4.
- **Workload AWS access uses EKS Pod Identity**, not IRSA — fewer moving
  parts (no OIDC provider or per-role trust-policy management to
  maintain), and it's AWS's current recommended mechanism.
- **No long-lived AWS credentials live in Jenkins.** Both the
  infrastructure pipeline and the EKS deploy stage of the application
  pipeline authenticate via the IAM role attached to the `platform-ops`
  EC2 instance itself.

---

## Known gaps

See `docs/architecture.md` §6 for the current list — kept there rather
than duplicated here so there's one place it can go stale.

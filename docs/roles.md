# DP Platform — Ansible Roles

A one-line reference for every role in this repository, grouped by what
they do. For how these roles fit into the overall system, see
[`architecture.md`](architecture.md); for how they're invoked, see
[`pipeline.md`](pipeline.md).

---

## Toolchain (installed on one or more of the three hosts)

| Role | Installs / configures |
|---|---|
| `common` | Baseline packages, timezone, and system updates applied to every host |
| `git` | Git, verified post-install |
| `java` | JDK, configured for Maven and Jenkins builds |
| `maven` | Maven, configured with the settings the build pipeline needs |
| `docker` | Docker Engine, configured and verified, used for image builds and Artifactory's container |
| `terraform` | Terraform CLI on `platform-ops`, pinned version, verified |
| `ansible` | Ansible itself on `platform-ops` (the control node re-provisioning its own tooling) |
| `kubectl` | kubectl on `platform-ops`, checksum-verified download |
| `helm` | Helm on `platform-ops`, checksum-verified download |
| `trivy` | Trivy container-image scanner on `build-agent`, used by the `dp-mywebapp` pipeline |
| `node_exporter` | Prometheus node_exporter as a systemd service, deployed to all three hosts and self-verified via a local `/metrics` check |

## CI/CD core

| Role | Purpose |
|---|---|
| `jenkins` | Installs Jenkins and drives its entire configuration via JCasC: security realm, credentials store, tool config (SonarCloud, Snyk), and idempotent job provisioning against the real Jenkins REST API |
| `jenkins_agent` | Configures a host as a Jenkins SSH build agent (`build-agent`, `platform-ops`) |
| `jenkins_ssh` | Generates and distributes the SSH identity Jenkins Controller uses to reach its permanent build agents; ends with a live connectivity check |
| `platform_ssh` | Generates and distributes the separate SSH identity Platform Ops uses as the Ansible control node in production; also ends with a live connectivity check |
| `platform_ops` | Baseline role for the Platform Ops host itself |
| `artifactory` | Runs JFrog Artifactory OSS as a Docker container with a persistent volume, idempotent lifecycle management, and a polling-based admin bootstrap |

## Kubernetes workloads (gated by `eks_runtime_enabled`)

| Role | Deploys |
|---|---|
| `aws_load_balancer_controller` | The AWS Load Balancer Controller via Helm into `kube-system`, authenticated via EKS Pod Identity |
| `kube_prometheus_stack` | Prometheus, Grafana, Alertmanager, kube-state-metrics, and the node-exporter DaemonSet via Helm into `monitoring`, with explicit resource sizing and additional scrape configs reaching the three EC2 hosts over the VPC |

## Intentional stubs

These are empty role directories, left in the tree deliberately rather
than deleted, documented here so their absence of content isn't mistaken
for an oversight:

| Role | Status |
|---|---|
| `snyk` | Not built as a host-level role. Snyk is currently wired in exclusively through the `dp-mywebapp` Jenkins pipeline (the `snykSecurity` step) and the Jenkins tool/credential configuration in the `jenkins` role's JCasC templates |
| `datadog` | Not currently integrated anywhere in the platform. `node_exporter` + `kube_prometheus_stack` is the observability stack actually in use |
| `eksctl` | Not used. The EKS cluster and node group are managed directly via Terraform (`eks.tf`), which supersedes any need for the `eksctl` CLI |
| `jfrog` | Superseded by the `artifactory` role, which is the one actually in use |
| `build_agent` | Superseded by the `jenkins_agent` role, which is the one actually in use |

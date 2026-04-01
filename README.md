# Homelab Infrastructure

Self-hosted environment running 25+ services across a K3s cluster and a Docker host on three Proxmox hypervisors. Everything is managed through GitOps, backed by proper security tooling, and monitored end to end.

## What's Running

### GitOps & Kubernetes

The K3s cluster runs on three nodes — one control plane, two workers — all on Fedora Server VMs. ArgoCD watches a private Gitea repo and syncs every manifest automatically. No manual `kubectl apply`, ever. If something drifts, ArgoCD corrects it.

All secrets are handled with Bitnami Sealed Secrets. The encrypted blobs live in Git, and only the cluster can decrypt them. The Sealed Secrets master key is backed up to NFS — losing it would mean re-sealing everything.

Longhorn handles persistent storage with single-replica volumes, hourly and daily snapshots, and automatic scheduling across nodes. Velero runs a daily backup of all cluster resources to a MinIO S3 bucket running in-cluster.

Traefik is the ingress controller with cert-manager handling Let's Encrypt certificates via Cloudflare DNS-01 challenges. Every exposed service gets TLS automatically.

### Security

This is the part I've spent the most time on.

**Wazuh** runs as a full SIEM — the manager sits bare-metal on the control plane node, with agents deployed to every host in the fleet. It collects file integrity monitoring events, authentication logs, and integrates with Suricata on pfSense via syslog. CIS benchmark scans run against all hosts and currently score between 56–78% depending on the host role.

**CrowdSec** runs on the Docker host, parsing Caddy access logs and SSH auth logs. It pulls shared threat intelligence and automatically bans IPs matching known attack patterns. An n8n workflow checks active CrowdSec decisions every hour and sends a summary to ntfy.

**Trivy Operator** runs in-cluster as a Helm chart managed by ArgoCD. It continuously scans every container image for vulnerabilities and generates VulnerabilityReport CRDs. A weekly n8n workflow queries these reports and the Docker host's Trivy scans, then pushes a combined summary to ntfy.

**TruffleHog** runs as a K8s CronJob every Sunday. It pulls a list of all repos from the Gitea API, clones each one, and scans the full git history for leaked secrets. Results go to ntfy.

All VMs are provisioned with **CIS-aligned hardening** baked into the cloud-init template — disabled unused kernel modules, restricted sysctl parameters, SSH hardened (no root login, strong ciphers only, limited auth tries), auditd rules for identity and sudoers changes, AIDE file integrity, and firewalld with default deny.

### Monitoring

Prometheus scrapes node-exporter and cAdvisor across all hosts. Loki collects logs from Promtail DaemonSets on every K3s node, plus a Promtail container on the Docker host shipping system, journal, and container logs. Grafana ties it all together.

n8n is where a lot of the operational glue lives. It runs daily log analysis across both K3s pods and Docker containers (via Ansible playbooks over SSH), condenses error logs with Ollama (Gemma 3 12B running locally), and pushes readable summaries to ntfy. There's also a weekly health report that pulls Prometheus metrics, disk/SMART data, Docker bloat stats, and K3s cluster health into one notification.

### Networking

pfSense handles routing and firewalling. The trusted server VLAN (86) is separate from IoT/guest traffic (VLAN 87). Pi-hole with Unbound provides recursive DNS — no upstream DNS forwarding. Suricata runs on pfSense as an IDS with logs forwarded to Wazuh.

Caddy on the Docker host acts as the external reverse proxy. Every service goes through Authelia for SSO via forward auth, with bypass rules only for apps that need their own mobile auth (Immich, ntfy). Tailscale provides remote access without exposing anything to the public internet.

### Infrastructure as Code

Terraform provisions VMs on Proxmox using the bpg/proxmox provider. The cloud-init template is a standalone CIS-hardened Fedora config that gets injected via `templatefile()`. One `terraform apply` gives you a fully hardened VM with SSH keys, firewalld, SELinux enforcing, auditd, and AIDE — ready for Ansible to take over.

Ansible handles ongoing state — fleet-wide package updates, kernel hardening enforcement, Docker daemon config, service management, and cleanup jobs. Playbooks also power the n8n workflows (SSH failures, CrowdSec checks, log scanning, Trivy results).

### Backup Strategy

Four layers: Longhorn snapshots (hourly + daily retention) for K3s volumes, Velero daily backups to MinIO for cluster resources, Kopia for Docker bind mounts to NAS, and Proxmox vzdump for full VM backups. Each layer covers a different failure scenario.

## Repositories

| Repository | What's in it |
|---|---|
| **[Kubernetes](https://github.com/rasmusbakandersen/Kubernetes)** | Every K3s manifest — app deployments, ArgoCD applications, Sealed Secrets, Longhorn/Velero/Traefik infra, monitoring stack, Trivy Operator, TruffleHog CronJob. |
| **[Docker](https://github.com/rasmusbakandersen/Docker)** | Compose stacks for the Docker host — Caddy + Authelia forward auth, CrowdSec, Gitea, Kopia, monitoring agents, Watchtower, and Printarr (a custom print/scan web app I built). |
| **[Automation](https://github.com/rasmusbakandersen/Automation)** | Ansible playbooks for hardening, updates, log scanning, NUT UPS deployment. Terraform configs for Proxmox VM provisioning. Operational scripts and PowerShell tooling. |

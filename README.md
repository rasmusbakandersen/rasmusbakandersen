# Homelab Infrastructure

Production-grade self-hosted platform running 25+ services across a three-node K3s cluster and a dedicated Docker host, built on three Proxmox hypervisors. The entire stack is GitOps-driven via ArgoCD, secured with a full SIEM/IDS pipeline, and monitored through Prometheus, Loki, and Grafana. Built as a hands-on complement to my professional work in cloud infrastructure — every design choice reflects real-world patterns I'd apply at scale.

## Platform Overview

### GitOps & Kubernetes

The K3s cluster runs on three nodes — one control plane, two workers — all on Fedora Server VMs. ArgoCD watches a private Gitea repo and syncs every manifest automatically. No manual `kubectl apply`, ever. If something drifts, ArgoCD corrects it.

Secrets are handled per-layer: Kubernetes uses Bitnami Sealed Secrets with encrypted blobs committed to Git — only the cluster can decrypt them. Docker services pull from `.env` files excluded from version control. Terraform loads sensitive variables from git-ignored `.tfvars` files. Nothing sensitive is committed in plaintext.

Traefik is the ingress controller with cert-manager handling Let's Encrypt certificates via Cloudflare DNS-01 challenges. Every exposed service gets TLS automatically. Velero runs daily cluster backups to a MinIO S3 bucket running in-cluster.

### Security

**Wazuh** is the backbone of the security stack, serving as both SIEM and SOAR. It collects and correlates logs from every host — authentication events, file integrity changes (via AIDE), and Suricata IDS alerts forwarded from pfSense over syslog. Active response rules automatically block IPs, kill processes, or trigger alerts based on correlation logic.

Wazuh also runs SCA modules that continuously scan all hosts against CIS benchmarks, currently scoring 56–78% depending on host role, with ongoing remediation targeting 80%+. The manager runs bare-metal on the control plane node with agents deployed to every host in the fleet.

**CrowdSec** runs on the Docker host, parsing Caddy access logs and SSH auth logs. It pulls shared threat intelligence and automatically bans IPs matching known attack patterns. An n8n workflow checks active CrowdSec decisions every hour and sends a summary to ntfy.

**Trivy Operator** runs in-cluster as a Helm chart managed by ArgoCD. It continuously scans every container image for vulnerabilities and generates VulnerabilityReport CRDs. A weekly n8n workflow queries these reports and the Docker host's Trivy scans, then pushes a combined summary to ntfy.

**TruffleHog** runs as a K8s CronJob every Sunday. It pulls a list of all repos from the Gitea API, clones each one, and scans the full git history for leaked secrets. Results go to ntfy.

**Container Hardening**. All containers — both K3s and Docker — run with `cap_drop: ALL`, `no-new-privileges: true`, `read_only` root filesystems where possible, and explicit resource limits (CPU and memory). This limits blast radius in case of a container escape and ensures no service can silently consume host resources.

All VMs are provisioned with **CIS-aligned hardening** baked into the cloud-init template — disabled unused kernel modules, restricted sysctl parameters, SSH hardened (no root login, strong ciphers only, limited auth tries), auditd rules for identity and sudoers changes, AIDE file integrity, and firewalld with default deny.

### Monitoring

Grafana is the central dashboard for everything. It pulls metrics from Prometheus, which scrapes node-exporter and cAdvisor across all hosts, and queries logs from Loki, which collects from Promtail DaemonSets on every K3s node plus a Promtail container on the Docker host shipping system, journal, and container logs.

n8n is where a lot of the operational glue lives. It runs daily log analysis across both K3s pods and Docker containers (via Ansible playbooks over SSH), condenses error logs with Ollama (Gemma 3 12B running locally), and pushes readable summaries to ntfy. There's also a weekly health report that pulls Prometheus metrics, disk/SMART data, Docker bloat stats, and K3s cluster health into one notification.

### Networking

pfSense handles routing and firewalling. The network is segmented following least-privilege principles — the trusted server VLAN (86) is isolated from IoT and guest traffic (VLAN 87), with firewall rules restricting inter-VLAN communication to only what's explicitly needed. Pi-hole with Unbound provides recursive DNS — no upstream DNS forwarding.

Caddy on the Docker host acts as the external reverse proxy. Every service goes through Authelia for SSO via forward auth, with bypass rules only for apps that need their own mobile auth (Immich, ntfy). Tailscale provides remote access without exposing anything to the public internet.

### Infrastructure as Code

Terraform provisions VMs on Proxmox using the bpg/proxmox provider. Each VM is cloned from a Fedora Cloud template and gets a CIS-hardened cloud-init config injected at creation — SSH lockdown, SELinux enforcing, auditd, AIDE, and firewalld all configured before the VM even boots.

Ansible handles ongoing state — fleet-wide package updates, kernel hardening enforcement, Docker daemon config, service management, and cleanup jobs. Playbooks also power the n8n workflows (SSH failures, CrowdSec checks, log scanning, Trivy results).

### Backup Strategy

Four layers, each covering a different failure scenario: Longhorn snapshots (hourly + daily) for K3s volumes, Velero daily backups to MinIO for full cluster resources, Kopia for Docker bind mounts to NAS, and Proxmox vzdump for VM-level recovery. Restores have been tested — including full Velero cluster recovery and Kopia file-level restores — to verify the chain actually works end to end.


## Repositories

| Repository | What's in it |
|---|---|
| **[Kubernetes](https://github.com/rasmusbakandersen/Kubernetes)** | Every K3s manifest — app deployments, ArgoCD applications, Sealed Secrets, Velero/Traefik infra, monitoring stack, Trivy Operator, TruffleHog CronJob. |
| **[Docker](https://github.com/rasmusbakandersen/Docker)** | Compose stacks for the Docker host — Caddy + Authelia forward auth, CrowdSec, Gitea, Kopia, monitoring agents, Watchtower, and Printarr (a custom print/scan web app I built). |
| **[Automation](https://github.com/rasmusbakandersen/Automation)** | Ansible playbooks for hardening, updates, log scanning, NUT UPS deployment. Terraform configs for Proxmox VM provisioning. Operational scripts and PowerShell tooling. |

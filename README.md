# Homelab Infrastructure

I self host  30+ services across a three node Kubernetes cluster and a dedicated Docker host, built on four Proxmox hypervisors. The entire Kubernetes stack is GitOps driven via ArgoCD, secured with a SIEM/SOAR solution, and monitored through Prometheus and Grafana. I built it as a hands on complement to my professional work in cloud infrastructure - and to be less dependent on big tech.

## Platform Overview

### GitOps & Kubernetes

The Kubernetes cluster runs on three nodes, one control plane and two workers - all on Fedora Server VMs. ArgoCD watches a Gitea repo and syncs every manifest automatically.

Secrets are handled per-layer. Kubernetes uses Sealed Secrets with encrypted blobs committed to Git — only the cluster can decrypt them. Docker services pull from .env files excluded from version control. Nothing sensitive is committed in plaintext.

Traefik is the ingress controller with certmanager handling Let's Encrypt certificates via Cloudflare DNS-01 challenges. Every exposed service gets TLS automatically. Longhorn is the storage solution for pods and also runs daily cluster backups and snapshots.

### Security

**Wazuh** is the backbone of the security stack, serving as both SIEM and SOAR. It collects and correlates logs from every host — authentication events, file integrity changes (via AIDE), etc. Active response rules automatically block IPs, kill processes, or trigger alerts based on correlation logic.

Wazuh also runs SCA modules that continuously scan all hosts against CIS benchmarks, currently scoring 56–78% depending on host role, with ongoing remediation targeting 80%+. The manager runs bare metal on the control plane node with agents deployed to every host in the fleet.

**TruffleHog** runs as a K8s CronJob every Sunday. It pulls a list of all repos from the Gitea API, clones each one, and scans the full git history for leaked secrets. Results go to ntfy.

**Container Hardening**. All containers — both Kubernetes and Docker — run with `cap_drop: ALL`, `no-new-privileges: true`, `read_only` root filesystems where possible, and explicit resource limits (CPU and memory). This limits blast radius in case of a container escape and ensures no service can silently consume host resources.

All VMs are provisioned with **CIS-aligned hardening** baked into the cloud-init template — disabled unused kernel modules, restricted sysctl parameters, SSH hardened (no root login, strong ciphers only, limited auth tries), auditd rules for identity and sudoers changes, AIDE file integrity, and firewalld with default deny.

### Monitoring

Grafana is the central dashboard for everything. It pulls metrics from Prometheus, which scrapes node-exporter and cAdvisor across all hosts, and queries logs from Loki, which collects from Promtail DaemonSets on every K3s node plus a Promtail container on the Docker host shipping system, journal, and container logs.

n8n is where a lot of the operational glue lives. It runs daily log analysis across both K3s pods and Docker containers (via Ansible playbooks over SSH) and pushes readable summaries to ntfy. There's also a weekly health report that pulls Prometheus metrics, disk/SMART data, Docker bloat stats, and K3s cluster health into one notification.

### Networking

pfSense handles routing and firewalling. The network is segmented following least-privilege principles — the trusted server VLAN (86) is isolated from IoT and guest traffic (VLAN 87), with firewall rules restricting inter-VLAN communication to only what's explicitly needed. Pi-hole with Unbound provides recursive DNS - no upstream DNS forwarding.

Traefik on the Kubernetes Cluster acts as the external reverse proxy. Every service goes through Authelia for SSO via forward auth, with bypass rules only for apps that need their own mobile auth (Immich, ntfy). Tailscale provides remote access without exposing anything to the public internet.

### Infrastructure as Code


Ansible handles ongoing state — fleet-wide package updates, kernel hardening enforcement, Docker daemon config, service management, and cleanup jobs. Playbooks also power the n8n workflows (SSH failures, backup status, pod log scanning, etc.).

### Backup Strategy

Four layers, each covering a different failure scenario: Daily Longhorn snapshots and backups for Kubernetes volumes, Kopia for Docker bind mounts to NAS, and Proxmox vzdump for VM-level recovery. Restores have been tested — including full Longhorn cluster recovery and Kopia file-level restores — to verify the chain actually works end to end.



## Repositories

| Repository | What's in it |
|---|---|
| **[Kubernetes](https://github.com/rasmusbakandersen/Kubernetes)** | Every K3s manifest — app deployments, ArgoCD applications, Sealed Secrets, Velero/Traefik infra, monitoring stack, Trivy Operator, TruffleHog CronJob. |
| **[Docker](https://github.com/rasmusbakandersen/Docker)** | Compose stacks for the Docker host — Caddy + Authelia forward auth, CrowdSec, Gitea, Kopia, monitoring agents, Watchtower, and Printarr (a custom print/scan web app I built). |
| **[Automation](https://github.com/rasmusbakandersen/Automation)** | Ansible playbooks for hardening, updates, log scanning, NUT UPS deployment. Terraform configs for Proxmox VM provisioning. Operational scripts and PowerShell tooling. |




### Fun Side Project: Nokia E72 from 2008

I have an old Nokia E72 from 2008 that I've tried to push to its limit in 2026. It's held back by old protocols and outdated security, but with a few creative workarounds I got email, calendar, contacts and to-do lists working with two-way sync.

It's not something anyone would use in production - I did it to challenge myself to find creative ways around real technical limits, and to see how far I could get with what I had.

To make it work I run a couple of Docker containers in my homelab that bridge the phone to modern services.

**1. Email bridge**

The E72 can't connect to Gmail directly because it doesn't support the TLS versions and authentication Gmail requires. I built a mail bridge container using stunnel. The phone connects to it over my local network, and the bridge handles the secure connection to Gmail. No changes to the Google account and no third-party service involved.

**2. Calendar, contacts and to-do lists**

A separate Docker project handles the sync:

- **Radicale** (CalDAV/CardDAV) holds the calendar, contacts and tasks
- **Z-Push** sits in front of it so the phone's built-in Mail for Exchange client can sync
- **DAVe** is a web client for editing the same data from any modern device

Changes on the phone show up on the server and the other way around.

**3. Using email for more**

Since the phone can read email reliably, I use email to deliver other things. For example, an n8n workflow on my K3s cluster sends a plain-text morning briefing every day with weather, calendar, tasks and news headlines.

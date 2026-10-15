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


### Fun Side Project on a Nokia Phone from 2008

I have an old Nokia phone from 2008 (The E72 model), which i have tried to push to its absolute limit in 2026. Its very limited by it's old protocols and outdated systems, but with a few creative work arounds i got 2-way sync working for Email, Calendar, Contacts, To-Do lists, and more.

To make this work i setup a couple of docker containers in my HomeLab, which bridge the old Nokia workings to modern systems

Here's what i did to make it work:
1. Email bridge (modern Gmail on a legacy mail client)

The E72's native mail client cannot connect to Gmail directly. It does not support the modern authentication and TLS versions Gmail requires.

Built a self-hosted mail bridge container on my Docker host, using stunnel to terminate modern TLS toward Gmail and present a connection the phone can understand
The phone connects to the bridge over my local network, and the bridge handles the secure connection upstream
Result: the phone sends and receives real Gmail again, with no changes to the Google account and no third-party service in the middle

2. Two-way sync for calendar, contacts and to-do lists

A separate Docker project provides the sync backend:

Radicale (CalDAV/CardDAV server) as the single source of truth for calendar, contacts and tasks
Z-Push in front of it, so the phone's built-in Mail for Exchange client can sync natively
DAVe web client, so the same calendar and tasks can be browsed and edited from any modern device
Changes made on the phone show up on the server and vice versa, both directions

3. Using the platform for more

Because the phone can now reliably read email, I use email as the transport for further features, for example a daily morning briefing generated by an n8n workflow running on my K3s cluster (weather, calendar, tasks, news headlines), delivered as plain text the phone handles well.

## Repositories

| Repository | What's in it |
|---|---|
| **[Kubernetes](https://github.com/rasmusbakandersen/Kubernetes)** | Every K3s manifest — app deployments, ArgoCD applications, Sealed Secrets, Velero/Traefik infra, monitoring stack, Trivy Operator, TruffleHog CronJob. |
| **[Docker](https://github.com/rasmusbakandersen/Docker)** | Compose stacks for the Docker host — Caddy + Authelia forward auth, CrowdSec, Gitea, Kopia, monitoring agents, Watchtower, and Printarr (a custom print/scan web app I built). |
| **[Automation](https://github.com/rasmusbakandersen/Automation)** | Ansible playbooks for hardening, updates, log scanning, NUT UPS deployment. Terraform configs for Proxmox VM provisioning. Operational scripts and PowerShell tooling. |

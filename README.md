# Homelab Automation

A Linux homelab where I automate the operations work I used to do by hand.

![Progress](https://img.shields.io/badge/progress-1%20of%204%20done-E8962E)
![Linux](https://img.shields.io/badge/Linux-FCC624?logo=linux&logoColor=black)
![Bash](https://img.shields.io/badge/Bash-4EAA25?logo=gnubash&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-EE0000?logo=ansible&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)

Every server in this lab started life configured by hand: SSH in, edit config files, install packages, and hope I remembered every step on the next machine. This repository replaces that with code. Servers are hardened to a known baseline, provisioned with Ansible, watched by Prometheus and Grafana, and (eventually) run container workloads on a small Kubernetes cluster. If a machine dies, I reinstall the OS, run one bootstrap script and one playbook, and it comes back exactly as it was.

The full build, step by step, is in *[docs/GUIDE.md](docs/GUIDE.md)*.

---

## Roadmap

| Phase | Milestone | Status |
| :---: | --- | --- |
| 1 | Linux servers set up and hardened | ✅ Done |
| 2 | Provisioning with Ansible playbooks | 🚧 In progress |
| 3 | Metrics and alerts with Prometheus and Grafana | ⏳ Planned |
| 4 | Kubernetes cluster for container workloads | ⏳ Planned |

- [x] *Phase 1:* Install Linux, set static addresses, lock down SSH, enable a default-deny firewall, brute-force protection and automatic security updates.
- [ ] *Phase 2:* Turn every manual step from Phase 1 into idempotent Ansible roles, with secrets in Ansible Vault and a single site.yml entry point.
- [ ] *Phase 3:* Ship node metrics from every host to Prometheus, route alerts through Alertmanager, and build Grafana dashboards that are provisioned from code.
- [ ] *Phase 4:* Deploy a k3s cluster with Ansible and run real container workloads on it.

---

## Architecture

mermaid
flowchart LR
    subgraph CTL["Control node (workstation)"]
        A["Git repo<br/>Ansible + Bash"]
    end

    subgraph MON["lab-mon-01"]
        P["Prometheus :9090"]
        AM["Alertmanager :9093"]
        G["Grafana :3000"]
    end

    subgraph K8S["k3s cluster"]
        N1["lab-node-01<br/>server"]
        N2["lab-node-02<br/>agent"]
        N3["lab-node-03<br/>agent"]
    end

    A -- "SSH (key only)" --> MON
    A -- "SSH (key only)" --> K8S
    P -- "scrape :9100" --> N1 & N2 & N3
    P --> AM --> NOTIFY["Discord / email"]
    G -- "PromQL" --> P

| Host | Example IP | Role |
| --- | --- | --- |
| Control node | 192.168.1.10 | My workstation. Holds this repo and runs Ansible. |
| lab-mon-01 | 192.168.1.20 | Prometheus, Alertmanager, Grafana |
| lab-node-01 | 192.168.1.21 | k3s server (control plane) |
| lab-node-02 | 192.168.1.22 | k3s agent |
| lab-node-03 | 192.168.1.23 | k3s agent |

<!-- Hardware: replace with your own, e.g. "3× mini PCs + 1 VM on Proxmox, Ubuntu Server LTS" -->
All hosts run Ubuntu Server LTS (Debian works with no changes). Every node exports metrics, every node is managed by the same baseline playbook, and nothing is exposed to the internet.

---

## Tech stack

| Tool | What it does in this lab |
| --- | --- |
| *Linux* | Ubuntu Server LTS on every host, hardened to a common baseline |
| *Bash* | One-time bootstrap of fresh servers and quick health checks from the control node |
| *Ansible* | Declarative, idempotent configuration of every host, organized into roles |
| *Prometheus* | Scrapes node_exporter on each host, evaluates alert rules, forwards alerts to Alertmanager |
| *Grafana* | Dashboards for host health, with data sources and dashboards provisioned from this repo |
| k3s (Phase 4) | Lightweight Kubernetes distribution for container workloads |

---

## Repository layout

text
homelab-automation/
├── ansible.cfg
├── collections/
│   └── requirements.yml          # community.general, ansible.posix
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── all/
│           ├── vars.yml
│           └── vault.yml         # encrypted with ansible-vault
├── playbooks/
│   ├── site.yml                  # runs everything, in order
│   ├── baseline.yml              # common + hardening (Phases 1–2)
│   ├── monitoring.yml            # node_exporter, Prometheus, Grafana (Phase 3)
│   ├── k3s.yml                   # Kubernetes cluster (Phase 4)
│   └── patch.yml                 # rolling OS updates, one host at a time
├── roles/
│   ├── common/                   # packages, hostname, timezone, auto-updates
│   ├── hardening/                # SSH, UFW, fail2ban, sysctl
│   ├── node_exporter/
│   ├── prometheus/               # Prometheus + Alertmanager + alert rules
│   ├── grafana/
│   └── k3s/
├── scripts/
│   ├── bootstrap.sh              # prepares a fresh server for Ansible
│   └── healthcheck.sh            # lab status from the control node
└── docs/
    └── GUIDE.md                  # step-by-step build guide

---

## Getting started

### Prerequisites

- One or more machines (physical or VM) running a fresh install of Ubuntu Server LTS with OpenSSH enabled
- A control node with Ansible installed (pipx install --include-deps ansible)
- An SSH key pair dedicated to the lab (ssh-keygen -t ed25519 -f ~/.ssh/homelab_ed25519)

### Quick start

# 1. Clone and install collections
git clone https://github.com/<your-username>/homelab-automation.git
cd homelab-automation
ansible-galaxy collection install -r collections/requirements.yml

# 2. Prepare each new server once (creates the 'ansible' user with key-only SSH)
scp scripts/bootstrap.sh <you>@192.168.1.21:/tmp/
ssh -t <you>@192.168.1.21 "sudo bash /tmp/bootstrap.sh '$(cat ~/.ssh/homelab_ed25519.pub)'"

# 3. Create the vault password file (never committed)
( umask 077; read -rsp 'Vault password: ' p; printf '%s\n' "$p" > .vault_pass )

# 4. Confirm Ansible can reach every host
ansible homelab -m ansible.builtin.ping

# 5. Preview, then apply
ansible-playbook playbooks/site.yml --check --diff
ansible-playbook playbooks/site.yml

---

## What gets automated

### Hardening baseline (Phase 1 → codified in Phase 2)

| Area | Baseline |
| --- | --- |
| SSH | Key-only authentication, no root login, AllowUsers allow-list, short login grace time |
| Firewall | UFW default-deny inbound; SSH and service ports open to the lab subnet only |
| Brute force | fail2ban sshd jail reading from the systemd journal |
| Updates | unattended-upgrades installs security patches daily |
| Kernel | sysctl hardening (no ICMP redirects, no source routing, SYN cookies, restricted dmesg) |
| Access | Dedicated ansible service account with passwordless sudo, separate from my admin user |

### Playbooks

| Playbook | Targets | Purpose |
| --- | --- | --- |
| site.yml | all | Imports every playbook below in order |
| baseline.yml | homelab | common and hardening roles |
| monitoring.yml | homelab, monitoring | node_exporter everywhere, Prometheus and Grafana on the monitoring host |
| k3s.yml | k3s_cluster | k3s server first, then agents join with the server token |
| patch.yml | homelab | apt dist-upgrade and reboot if required, serial: 1 |

### Alerts (Phase 3)

| Alert | Fires when |
| --- | --- |
| InstanceDown | A scrape target is unreachable for 2 minutes |
| HostHighCpu | CPU above 90% for 10 minutes |
| HostHighMemory | Memory above 90% for 10 minutes |
| HostDiskAlmostFull | A filesystem is more than 85% full for 15 minutes |

---

## Everyday commands

# Dry run with a diff of every file that would change
ansible-playbook playbooks/site.yml --check --diff

# Only re-apply the hardening role
ansible-playbook playbooks/baseline.yml --tags hardening

# Target a single host
ansible-playbook playbooks/site.yml --limit lab-node-02

# Rolling OS updates
ansible-playbook playbooks/patch.yml

# Edit secrets
ansible-vault edit inventory/group_vars/all/vault.yml

# Lint before committing
ansible-lint

# Quick lab status
./scripts/healthcheck.sh

---

## Security notes

- SSH accepts keys only, and only for the users in the allow-list.
- Prometheus, Alertmanager and Grafana are reachable from the lab subnet only. node_exporter accepts connections from the monitoring host only.
- Secrets (Grafana admin password, alert webhook URL) live in vault.yml, encrypted with Ansible Vault.
- .vault_pass and the fetched kubeconfig (.kube/) are listed in .gitignore and never committed.
- Every config file is validated before it replaces the live one (sshd -t, promtool, amtool), so a typo fails the play instead of breaking a service.

---

## Design decisions

- *Roles per concern, tagged.* Each role does one job and can be applied on its own with --tags, which keeps runs fast and changes easy to review.
- *Distro packages for the Prometheus stack.* Updates come through apt like everything else. The trade-off is slightly older versions than upstream, which is fine for a homelab.
- *k3s over kubeadm.* A single binary with sensible defaults and low resource use, which suits small homelab hardware while still being real Kubernetes.
- *Idempotency is the test.* A second run of any playbook must report changed=0. If it doesn't, the role is wrong.

---

## License

MIT

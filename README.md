# Homelab Automation

A Linux homelab where I automate the operations work I used to do by hand.

Setting up a server, locking it down, installing the same packages on the next one, checking whether anything is on fire. I got tired of doing all of that manually, so this repo is where I'm turning it into scripts and playbooks.

## Progress

- [x] Linux servers set up and hardened
- [ ] Provisioning with Ansible playbooks (in progress)
- [ ] Metrics and alerts with Prometheus and Grafana
- [ ] Kubernetes cluster for container workloads

## Tools

| Tool       | What I use it for                             |
| ---------- | --------------------------------------------- |
| Linux      | Base OS for every machine in the lab          |
| Bash       | Small scripts for hardening and one-off tasks |
| Ansible    | Provisioning and keeping servers configured   |
| Prometheus | Collecting metrics from each server           |
| Grafana    | Dashboards and alerts                         |

## Repo layout

homelab-automation/
├── bash/            # hardening and helper scripts
├── ansible/
│   ├── inventory/   # hosts and groups
│   ├── playbooks/   # site.yml and friends
│   └── roles/       # base, ssh, firewall, node_exporter
├── monitoring/
│   ├── prometheus/  # prometheus.yml, alert rules
│   └── grafana/     # exported dashboards
└── docs/
    └── GUIDE.md     # step-by-step setup notes

## Quick start

You need a control machine with Ansible installed and SSH access to the servers you want to manage.

git clone https://github.com/<your-username>/homelab-automation.git
cd homelab-automation/ansible

# check that Ansible can reach everything
ansible all -i inventory/hosts.ini -m ping

# dry run first
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --check

# then for real
ansible-playbook -i inventory/hosts.ini playbooks/site.yml

The full walkthrough, from a fresh install to dashboards, is in [docs/GUIDE.md](docs/GUIDE.md).

## Notes

- Everything here is meant to be safe to run more than once.
- Secrets (passwords, tokens) are not committed. I use Ansible Vault for those.
- This is a learning project, so things will change as I go.

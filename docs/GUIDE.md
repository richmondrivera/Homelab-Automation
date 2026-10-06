# Setup Guide

The order I built things in. Stages 1 is done, stage 2 is in progress, the rest are planned.

## Stage 1: Linux servers set up and hardened (done)

*1. Install the OS.*
Install a minimal Debian or Ubuntu Server on each machine (or VM). Give each one a static IP or a DHCP reservation on your router so the address never changes.

*2. Create an admin user.*
Don't work as root.

adduser admin
usermod -aG sudo admin

*3. Set up SSH keys.*
On your control machine:

ssh-keygen -t ed25519
ssh-copy-id admin@192.168.1.10

*4. Lock down SSH.*
Edit /etc/ssh/sshd_config:

PermitRootLogin no
PasswordAuthentication no

Test a new login in a second terminal before closing your current session, then restart:

sudo systemctl restart ssh

*5. Turn on the firewall.*

sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable

*6. Add basic protection.*

sudo apt install fail2ban unattended-upgrades
sudo systemctl enable --now fail2ban

*7. Put it in a Bash script.*
Once you've done the steps above by hand, wrap them in bash/harden.sh so a new server takes minutes. This script is also what the Ansible roles replace in stage 2.

## Stage 2: Provisioning with Ansible (in progress)

*1. Install Ansible on the control machine.*

sudo apt install ansible
ansible --version

*2. Write an inventory.*
ansible/inventory/hosts.ini:

ini
[lab]
server1 ansible_host=192.168.1.10
server2 ansible_host=192.168.1.11

[lab:vars]
ansible_user=admin

*3. Test the connection.*

ansible all -i inventory/hosts.ini -m ping

You should see pong from every host.

*4. Turn the hardening steps into roles.*
Make one role per job so they stay small:

- base: packages, timezone, unattended upgrades
- ssh: the sshd settings from stage 1
- firewall: ufw rules
- node_exporter: added later for monitoring

cd ansible
ansible-galaxy init roles/base

*5. Tie the roles together in a playbook.*
playbooks/site.yml:

- hosts: lab
  become: true
  roles:
    - base
    - ssh
    - firewall

*6. Dry run, then apply.*

ansible-playbook -i inventory/hosts.ini playbooks/site.yml --check --diff
ansible-playbook -i inventory/hosts.ini playbooks/site.yml

*7. Run it twice.*
The second run should report zero changes. If it doesn't, something in a role isn't idempotent and needs fixing.

*8. Keep secrets out of git.*

ansible-vault create group_vars/lab/vault.yml

## Stage 3: Metrics and alerts (planned)

*1. Install node_exporter on every server.*
Add it as an Ansible role so it's part of provisioning. It exposes metrics on port 9100.

*2. Install Prometheus on one server.*
Point it at your hosts in prometheus.yml:

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['192.168.1.10:9100', '192.168.1.11:9100']

Open http://<prometheus-ip>:9090/targets and check that everything shows as UP.

*3. Install Grafana.*
Add Prometheus as a data source (http://localhost:9090 if it's on the same machine).

*4. Import a dashboard.*
Start with the community "Node Exporter Full" dashboard (ID 1860) instead of building one from scratch. Export your changes to monitoring/grafana/ so they're in git.

*5. Add alerts.*
Start with the ones that actually matter:

- A host is down
- Disk is over 85% full
- Memory is nearly exhausted

Send them somewhere you'll see them, like email or a chat webhook.

## Stage 4: Kubernetes cluster (planned)

*1. Pick a distribution.*
k3s is a good fit for a homelab because it's light and installs with one command.

*2. Provision the nodes with the existing Ansible roles.*
Reuse base, ssh, firewall and node_exporter so the cluster nodes are managed like everything else.

*3. Install k3s.*
On the first node:

curl -sfL https://get.k3s.io | sh -
sudo cat /var/lib/rancher/k3s/server/node-token

On the other nodes, join with that token and the first node's address.

*4. Check the cluster.*

kubectl get nodes

*5. Deploy something small.*
Run a simple test workload first, then move real services over one at a time.

*6. Monitor the cluster.*
Point Prometheus and Grafana at the nodes and the workloads running on them.

## Troubleshooting

- *Locked out of SSH:* keep a console or the hypervisor's console open while you change SSH settings.
- *Ansible can't connect:* run ssh admin@<ip> by hand first. If that fails, Ansible will too.
- *Prometheus target shows DOWN:* check that port 9100 is allowed in ufw.

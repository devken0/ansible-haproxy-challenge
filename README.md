# Technical Challenge — Worklog

## Environment

- Cloud provider: AWS — region is ap-southeast-2a
- OS: Ubuntu 26.04 LTS on all three machines
- Machine A: 172.31.8.122 (public IP 16.176.183.131) — haproxy + Ansible control node
- Machine B: 172.31.10.57 — nginx
- Machine C: 172.31.0.221 — nginx

## Ansible Playbooks

### SSH passwordless authentication from Machine A to B and C

```yml
---
- name: Read Machine A's public key from the control node
  ansible.builtin.slurp:
    src: ~/.ssh/id_ed25519.pub
  register: control_pubkey
  delegate_to: localhost      # read the file on A (the control node), not on B/C
  run_once: true

- name: Ensure Machine A's public key is authorized on the web servers
  ansible.posix.authorized_key:
    user: ubuntu
    state: present
    key: "{{ control_pubkey.content | b64decode }}"
```

### Nginx on B and C

```yaml
---
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true

- name: Deploy the nginx site config (listens on 8080)
  ansible.builtin.template:
    src: site.conf.j2
    dest: /etc/nginx/sites-available/challenge.conf
  notify: Restart nginx

- name: Enable our site (symlink into sites-enabled)
  ansible.builtin.file:
    src: /etc/nginx/sites-available/challenge.conf
    dest: /etc/nginx/sites-enabled/challenge.conf
    state: link
	force: true
  notify: Restart nginx

- name: Remove the default nginx site (it listens on 80 and would conflict)
  ansible.builtin.file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: Restart nginx

- name: Deploy the test.html page (unique per machine)
  ansible.builtin.template:
    src: test.html.j2
    dest: /var/www/html/test.html

- name: Ensure nginx is started and enabled at boot
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
  when: not ansible_check_mode
```

### HA Proxy

```yml
---
- name: Install haproxy
  ansible.builtin.apt:
    name: haproxy
    state: present
    update_cache: true

- name: Deploy the haproxy configuration
  ansible.builtin.template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
    validate: haproxy -c -f %s     # refuse to install a broken config
  notify: Restart haproxy

- name: Ensure haproxy is started and enabled at boot
  ansible.builtin.service:
    name: haproxy
    state: started
    enabled: true
  when: not ansible_check_mode
```

### Iptables configuration

```yaml
---
# --- Rules common to ALL machines ---

- name: Allow loopback traffic
  ansible.builtin.iptables:
    chain: INPUT
    in_interface: lo
    jump: ACCEPT
    action: insert          # insert at the top so it's evaluated first

- name: Allow established and related connections
  ansible.builtin.iptables:
    chain: INPUT
    ctstate: ESTABLISHED,RELATED
    jump: ACCEPT

- name: Allow inbound SSH (port 22)
  ansible.builtin.iptables:
    chain: INPUT
    protocol: tcp
    destination_port: "22"
    jump: ACCEPT

# --- Service port for the LOAD BALANCER (Machine A): HTTP on 80 ---

- name: Allow inbound HTTP on the load balancer
  ansible.builtin.iptables:
    chain: INPUT
    protocol: tcp
    destination_port: "80"
    jump: ACCEPT
  when: "'loadbalancer' in group_names"

# --- Service port for the WEB SERVERS (B and C): 8080, but only from A ---

- name: Allow inbound 8080 on the web servers, only from the load balancer
  ansible.builtin.iptables:
    chain: INPUT
    protocol: tcp
    destination_port: "8080"
    source: 172.31.8.122        # only Machine A may reach the backends
    jump: ACCEPT
  when: "'webservers' in group_names"

# --- Lock it down: default-drop everything else inbound ---

- name: Set default INPUT policy to DROP
  ansible.builtin.iptables:
    chain: INPUT
    policy: DROP

- name: Set default FORWARD policy to DROP
  ansible.builtin.iptables:
    chain: FORWARD
    policy: DROP

# --- Make the rules survive a reboot ---
- name: Persist running iptables rules to a file
  community.general.iptables_state: 
    state: saved 
    path: /etc/iptables.rules
```

### Master playbook (`site.yml`)

```yaml
---
- name: Configure passwordless SSH from A to the web servers
  hosts: webservers
  roles:
    - ssh_access

- name: Configure the web servers (nginx)
  hosts: webservers
  roles:
    - nginx

- name: Configure the load balancer (haproxy)
  hosts: loadbalancer
  roles:
    - haproxy

- name: Lock down all machines with iptables
  hosts: all
  roles:
    - firewall
```
## Steps taken 

1. Signed up for an AWS account
2. Provisioned three Ubuntu VMs in a single 172.31.0.0/20 subnet; A given a public IP.
3. Configured security groups: A allows 22, 80; B/C allow 22 and 8080 from subnet.
	1. Login and verify ssh access from local machine:

```bash
# ssh to Machine A
ssh -i ~/.ssh/mykey.pem ubuntu@16.176.183.131

# confirm Machine A's private IP
ip -4 addr show | grep inet
```

5. Installed Ansible on Machine A (control node).

```sh
sudo apt update 
sudo apt install -y ansible
ansible --version # confirm installation
```

6. Setup Ansible directory structure

```
challenge/
├── ansible.cfg                  # Ansible's settings for this project
├── inventory.ini                # the list of A, B, C
├── site.yml                     # the master playbook that ties everything together
├── group_vars/
│   ├── all.yml                  # variables for every host
│   ├── webservers.yml           # variables for B and C
│   └── loadbalancer.yml         # variables for A
└── roles/
    ├── ssh_access/tasks/main.yml
    ├── nginx/
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── site.conf.j2
    │       └── test.html.j2
    ├── haproxy/
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/haproxy.cfg.j2
    └── firewall/
        └── tasks/main.yml
```

Quick commands to create the directories:

```sh
mkdir -p challenge/group_vars
mkdir -p challenge/roles/ssh_access/tasks
mkdir -p challenge/roles/nginx/{tasks,handlers,templates}
mkdir -p challenge/roles/haproxy/{tasks,handlers,templates}
mkdir -p challenge/roles/firewall/tasks
cd challenge
```

7. Configure Ansible, variables and create inventory file

**`ansible.cfg`**

```
[defaults]
inventory = ./inventory.ini
remote_user = ubuntu
host_key_checking = False
interpreter_python = auto_silent

[privilege_escalation]
become = True
become_method = sudo
```

**`inventory.ini`**

```
[loadbalancer]
machineA ansible_host=172.31.8.122 ansible_connection=local

[webservers]
machineB ansible_host=172.31.10.57 machine_label="Machine B"
machineC ansible_host=172.31.0.221 machine_label="Machine C"

[all:vars]
ansible_user=ubuntu
```

**`group_vars/loadbalancer.yml`**

```
---
haproxy_frontend_port: 80
backend_servers:
  - name: webB
    address: 172.31.10.57
    port: 8080
  - name: webC
    address: 172.31.0.221
    port: 8080
redirect_url: "https://www.google.com"
```

**`group_vars/webservers.yml`**

```
---
nginx_port: 8080
```

8. Generated an ed25519 key pair on A; distributed A's public key to B and C via ssh-copy-id. Verified passwordless login.

```sh
# generate Machine A's own key pair
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# copy the public key to the authorized_keys file of Machine B and C.
cat ~/.ssh/id_ed25519.pub

# confirm ssh connection from Machine A
ssh ubuntu@172.31.10.57 hostname
ssh ubuntu@172.31.0.221 hostname
```

9. Wrote Ansible roles: ssh_access, nginx, haproxy, firewall.
10. Ran `ansible-playbook site.yml`; verified idempotency on a second run (all tasks 'ok').
11. Tested round-robin (curl loop — alternates B/C), redirect (curl -I — 302 to Google), listening ports, and firewall rules.

## Verification evidence

*Successful Ansible playbook execution*

<img width="1920" height="940" alt="image" src="https://github.com/user-attachments/assets/832406df-6550-4f77-8e73-7e8e3c8d43d6" />


*Verified idempotency on a second run*

<img width="1920" height="940" alt="image" src="https://github.com/user-attachments/assets/c02459b5-2e82-4107-bb37-17114071573c" />

*Tested ACL*

<img width="621" height="355" alt="image" src="https://github.com/user-attachments/assets/855e547d-fbed-459c-aad0-55c419d7958a" />
<img width="846" height="180" alt="image" src="https://github.com/user-attachments/assets/6999972a-0e5b-4539-9b12-1ff2a082b31b" />

*Tested round-robin*

<img width="1375" height="236" alt="image" src="https://github.com/user-attachments/assets/aa2f29c8-3f93-4ccf-a465-ed72a7576810" />

## Deviations

- Used Ubuntu LTS 26.04 for all three machines to keep it standard, stable and efficient.
- Used the modern ed25519 instead of the older RSA default for generated SSH keys
- Used the provided subnet 172.31.0.0/20 in AWS
- **Backend port 8080 restricted to Machine A's IP** (`-s 172.31.8.122`) — defense in depth; the backends only ever receive traffic from the load balancer, so exposing 8080 more widely is unnecessary.
- **OUTPUT policy left as ACCEPT** - outbound must stay open for DNS, package updates, and haproxy→backend traffic. A locked OUTPUT chain is possible but would require explicitly permitting each of those.

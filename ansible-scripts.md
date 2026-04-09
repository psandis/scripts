# Ansible Scripts & Playbooks

Configuration management and server provisioning. Covers Docker, Nginx, PostgreSQL, Certbot SSL, and deploying containerized web apps.

---

## Table of Contents

- [Ansible Basics](#ansible-basics)
- [Inventory](#inventory)
- [Playbook: Docker Setup](#playbook-docker-setup)
- [Playbook: Nginx Reverse Proxy](#playbook-nginx-reverse-proxy)
- [Playbook: Certbot SSL](#playbook-certbot-ssl)
- [Playbook: PostgreSQL](#playbook-postgresql)
- [Playbook: Deploy Dockerized Web App](#playbook-deploy-dockerized-web-app)
- [Playbook: Full Stack (All-in-One)](#playbook-full-stack-all-in-one)
- [Useful Ad-Hoc Commands](#useful-ad-hoc-commands)
- [Roles Structure](#roles-structure)
- [Tips](#tips)

---

## Ansible Basics

### Install Ansible

```bash
# macOS
brew install ansible

# Ubuntu/Debian
sudo apt update && sudo apt install -y ansible

# pip (any platform)
pip install ansible
```

---

### Core Commands

```bash
ansible --version                         # Check version
ansible all -m ping                       # Ping all hosts
ansible-playbook playbook.yml             # Run a playbook
ansible-playbook playbook.yml --check     # Dry run
ansible-playbook playbook.yml -v          # Verbose
ansible-playbook playbook.yml --limit web # Run only on 'web' group
ansible-playbook playbook.yml --tags ssl  # Run only tagged tasks
ansible-vault encrypt secrets.yml         # Encrypt a file
ansible-vault decrypt secrets.yml         # Decrypt a file
ansible-galaxy init my_role               # Create a role skeleton
```

---

## Inventory

### hosts.ini

```ini
[web]
web1 ansible_host=203.0.113.10 ansible_user=deploy

[db]
db1 ansible_host=10.0.10.5 ansible_user=deploy

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### Dynamic inventory with Terraform

After `terraform apply`, generate inventory from outputs:

```bash
echo "[web]" > hosts.ini
echo "web1 ansible_host=$(terraform output -raw web_public_ip) ansible_user=deploy" >> hosts.ini
```

---

## Playbook: Docker Setup

Install Docker CE and Docker Compose on Ubuntu.

```yaml
# docker-setup.yml
---
- name: Install Docker
  hosts: web
  become: true

  tasks:
    - name: Install prerequisites
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: true

    - name: Add Docker GPG key
      ansible.builtin.apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker repository
      ansible.builtin.apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present

    - name: Install Docker
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-compose-plugin
        state: present
        update_cache: true

    - name: Start and enable Docker
      systemd:
        name: docker
        state: started
        enabled: true

    - name: Add deploy user to docker group
      user:
        name: deploy
        groups: docker
        append: true
```

---

## Playbook: Nginx Reverse Proxy

Install Nginx and configure it as a reverse proxy for a Docker container running on port 3000.

```yaml
# nginx-setup.yml
---
- name: Setup Nginx reverse proxy
  hosts: web
  become: true
  vars:
    domain_name: example.com
    upstream_port: 3000

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: true

    - name: Remove default site
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent

    - name: Create Nginx config
      copy:
        dest: /etc/nginx/sites-available/{{ domain_name }}
        content: |
          server {
              listen 80;
              server_name {{ domain_name }} www.{{ domain_name }};

              location / {
                  proxy_pass http://127.0.0.1:{{ upstream_port }};
                  proxy_http_version 1.1;
                  proxy_set_header Upgrade $http_upgrade;
                  proxy_set_header Connection 'upgrade';
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Proto $scheme;
                  proxy_cache_bypass $http_upgrade;
              }
          }
      notify: Reload Nginx

    - name: Enable site
      file:
        src: /etc/nginx/sites-available/{{ domain_name }}
        dest: /etc/nginx/sites-enabled/{{ domain_name }}
        state: link
      notify: Reload Nginx

    - name: Test Nginx configuration
      command: nginx -t
      changed_when: false

    - name: Start and enable Nginx
      systemd:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded
```

---

## Playbook: Certbot SSL

Install Certbot and obtain Let's Encrypt SSL certificates. Requires Nginx already configured and DNS pointing to the server.

```yaml
# certbot-setup.yml
---
- name: Setup Certbot SSL
  hosts: web
  become: true
  vars:
    domain_name: example.com
    email: admin@example.com

  tasks:
    - name: Install Certbot and Nginx plugin
      apt:
        name:
          - certbot
          - python3-certbot-nginx
        state: present
        update_cache: true

    - name: Obtain SSL certificate
      command: >
        certbot --nginx
        -d {{ domain_name }}
        -d www.{{ domain_name }}
        --email {{ email }}
        --agree-tos
        --non-interactive
        --redirect
      args:
        creates: /etc/letsencrypt/live/{{ domain_name }}/fullchain.pem

    - name: Setup auto-renewal cron
      cron:
        name: "Certbot renewal"
        minute: "0"
        hour: "3"
        job: "certbot renew --quiet --post-hook 'systemctl reload nginx'"

    - name: Test renewal
      command: certbot renew --dry-run
      changed_when: false
```

---

## Playbook: PostgreSQL

Install and configure PostgreSQL on a dedicated database server.

```yaml
# postgres-setup.yml
---
- name: Setup PostgreSQL
  hosts: db
  become: true
  vars:
    db_name: appdb
    db_user: appuser
    db_password: "{{ vault_db_password }}"
    allowed_network: "10.0.0.0/16"

  tasks:
    - name: Install PostgreSQL
      apt:
        name:
          - postgresql
          - postgresql-contrib
          - python3-psycopg2
        state: present
        update_cache: true

    - name: Start and enable PostgreSQL
      systemd:
        name: postgresql
        state: started
        enabled: true

    - name: Create database
      become_user: postgres
      community.postgresql.postgresql_db:
        name: "{{ db_name }}"
        state: present

    - name: Create database user
      become_user: postgres
      community.postgresql.postgresql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        db: "{{ db_name }}"
        priv: "ALL"
        state: present

    - name: Allow remote connections in pg_hba.conf
      lineinfile:
        path: /etc/postgresql/15/main/pg_hba.conf
        line: "host    {{ db_name }}    {{ db_user }}    {{ allowed_network }}    md5"
        insertafter: "# IPv4 local connections"
      notify: Restart PostgreSQL

    - name: Listen on all interfaces
      lineinfile:
        path: /etc/postgresql/15/main/postgresql.conf
        regexp: "^#?listen_addresses"
        line: "listen_addresses = '*'"
      notify: Restart PostgreSQL

  handlers:
    - name: Restart PostgreSQL
      systemd:
        name: postgresql
        state: restarted
```

### Encrypted secrets (Ansible Vault)

```bash
# Create encrypted variables file
ansible-vault create group_vars/db/vault.yml
```

```yaml
# group_vars/db/vault.yml (encrypted)
vault_db_password: "super-secure-password-here"
```

```bash
# Run playbook with vault
ansible-playbook postgres-setup.yml --ask-vault-pass
```

---

## Playbook: Deploy Dockerized Web App

Deploy a containerized web app with Docker Compose.

```yaml
# deploy-app.yml
---
- name: Deploy web application
  hosts: web
  become: true
  vars:
    app_name: myapp
    app_dir: /opt/{{ app_name }}
    image: "ghcr.io/psandis/myapp:latest"
    container_port: 3000

  tasks:
    - name: Create app directory
      file:
        path: "{{ app_dir }}"
        state: directory
        owner: deploy
        group: deploy

    - name: Create docker-compose.yml
      copy:
        dest: "{{ app_dir }}/docker-compose.yml"
        owner: deploy
        content: |
          services:
            app:
              image: {{ image }}
              container_name: {{ app_name }}
              restart: unless-stopped
              ports:
                - "{{ container_port }}:{{ container_port }}"
              environment:
                - NODE_ENV=production
                - DATABASE_URL=postgresql://appuser:${DB_PASSWORD}@db1:5432/appdb
              healthcheck:
                test: ["CMD", "curl", "-f", "http://localhost:{{ container_port }}/health"]
                interval: 30s
                timeout: 10s
                retries: 3
      notify: Restart app

    - name: Create .env file
      copy:
        dest: "{{ app_dir }}/.env"
        owner: deploy
        mode: "0600"
        content: |
          DB_PASSWORD={{ vault_db_password }}
      notify: Restart app

    - name: Pull latest image
      command: docker compose pull
      args:
        chdir: "{{ app_dir }}"
      changed_when: true

    - name: Start application
      command: docker compose up -d
      args:
        chdir: "{{ app_dir }}"
      changed_when: true

  handlers:
    - name: Restart app
      command: docker compose up -d --force-recreate
      args:
        chdir: "{{ app_dir }}"
```

---

## Playbook: Full Stack (All-in-One)

Run everything in order.

```yaml
# site.yml
---
- name: Full stack deployment
  hosts: all
  become: true

  tasks:
    - name: Update all packages
      apt:
        upgrade: safe
        update_cache: true

- import_playbook: docker-setup.yml
- import_playbook: nginx-setup.yml
- import_playbook: certbot-setup.yml
- import_playbook: postgres-setup.yml
- import_playbook: deploy-app.yml
```

```bash
# Run everything
ansible-playbook -i hosts.ini site.yml --ask-vault-pass

# Run only SSL setup
ansible-playbook -i hosts.ini site.yml --tags ssl

# Dry run first
ansible-playbook -i hosts.ini site.yml --check
```

---

## Useful Ad-Hoc Commands

### Ping all hosts

```bash
ansible all -i hosts.ini -m ping
```

### Check uptime

```bash
ansible web -i hosts.ini -a "uptime"
```

### Restart a service

```bash
ansible web -i hosts.ini -b -m systemd -a "name=nginx state=restarted"
```

### Copy a file

```bash
ansible web -i hosts.ini -m copy -a "src=./app.conf dest=/etc/nginx/conf.d/app.conf" -b
```

### Check disk space

```bash
ansible all -i hosts.ini -a "df -h"
```

### Run on a single host

```bash
ansible web1 -i hosts.ini -a "docker ps"
```

### Gather facts

```bash
ansible web -i hosts.ini -m setup | grep ansible_distribution
```

---

## Roles Structure

For larger projects, organize playbooks into roles:

```
roles/
├── docker/
│   ├── tasks/main.yml
│   ├── handlers/main.yml
│   └── defaults/main.yml
├── nginx/
│   ├── tasks/main.yml
│   ├── handlers/main.yml
│   ├── templates/
│   │   └── site.conf.j2
│   └── defaults/main.yml
├── certbot/
│   ├── tasks/main.yml
│   └── defaults/main.yml
├── postgres/
│   ├── tasks/main.yml
│   ├── handlers/main.yml
│   └── defaults/main.yml
└── app/
    ├── tasks/main.yml
    ├── handlers/main.yml
    ├── templates/
    │   └── docker-compose.yml.j2
    └── defaults/main.yml
```

Use roles in a playbook:

```yaml
---
- name: Web servers
  hosts: web
  become: true
  roles:
    - docker
    - nginx
    - certbot
    - app

- name: Database servers
  hosts: db
  become: true
  roles:
    - postgres
```

---

## Tips

- Always use `--check` (dry run) before applying to production
- Use `ansible-vault` for all secrets, never plain text passwords
- Use `ansible-lint` to check playbooks for best practices
- Use `--diff` flag to see what changes Ansible makes to files
- Set `ansible.cfg` in your project root for consistent settings
- Use tags to run specific parts: `ansible-playbook site.yml --tags docker,nginx`
- Use `become: true` only where needed, not globally
- Test playbooks against a local VM or Docker container first

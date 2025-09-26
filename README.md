# Libernetixrulz
My first project
# WordPress Deployment with Ansible and Docker

## Objective
Provision an Ubuntu server using Ansible, install Docker and Docker Compose, and deploy a WordPress and MySQL stack using Docker Compose. Additionally, HTTPS is configured via Caddy and sslip.io.

## How to Run

### 1. Requirements
- Linux or WSL2 environment
- Ansible installed:
    sudo apt update && sudo apt install ansible -y
- SSH key provided for server access:
    chmod 600 ~/.ssh/cert.pem

### 2. Inventory
File: inventories/hosts.ini
    [wordpress]
    13.48.24.171 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/cert.pem

### 3. Run the playbook
    ansible-playbook -i inventories/hosts.ini site.yml

When finished:
- Site via HTTP: http://13.48.24.171/
- Site via HTTPS: https://13-48-24-171.sslip.io/

## Project Structure

    wp-ansible/
    ├── inventories/
    │   └── hosts.ini
    ├── group_vars/
    │   └── all.yml
    ├── templates/
    │   ├── docker-compose.yml.j2
    │   ├── env.j2
    │   ├── Caddyfile.j2
    │   ├── jail.local.j2
    │   ├── 20auto-upgrades.j2
    │   └── 50unattended-upgrades.j2
    ├── site.yml
    ├── README.md
    └── NOTES.md

## HTTPS with Caddy and sslip.io

### How it works
- Caddy runs as a reverse proxy in front of WordPress.
- It automatically issues and renews TLS certificates via Let’s Encrypt.
- Instead of a custom domain, the setup uses sslip.io which maps the server IP to a hostname automatically. Example: 13.48.24.171 becomes 13-48-24-171.sslip.io.

### Configuration
- In group_vars/all.yml:
    tls_host: "13-48-24-171.sslip.io"

- In templates/docker-compose.yml.j2:
  - remove the port mapping from wordpress (no ports: - "80:80")
  - add a caddy service listening on ports 80 and 443.

- In templates/Caddyfile.j2:
    {{ tls_host }} {
        reverse_proxy wordpress:80
    }

## Troubleshooting

### Check running containers
    docker compose -f /opt/wordpress/docker-compose.yml ps

### View logs
    docker compose -f /opt/wordpress/docker-compose.yml logs wordpress
    docker compose -f /opt/wordpress/docker-compose.yml logs caddy

### If the site looks broken (missing CSS/JS)
Update WordPress URLs:
    docker compose -f /opt/wordpress/docker-compose.yml exec -u www-data wordpress php -r "require 'wp-load.php'; update_option('home','https://13-48-24-171.sslip.io'); update_option('siteurl','https://13-48-24-171.sslip.io'); echo \"OK\n\";"

### If HTTPS is not working
- Ensure ports 80 and 443 are open in UFW and in the cloud security group.
- Check Caddy logs:
    docker compose -f /opt/wordpress/docker-compose.yml logs caddy | tail -n 50

## Features Implemented
- Automated provisioning with Ansible:
  - System updates
  - Docker and Docker Compose installation
  - UFW firewall (22, 80, 443)
  - Fail2ban
  - SSH hardening
  - Unattended security upgrades
- Deployment of WordPress and MySQL via Docker Compose.
- HTTPS using Caddy and sslip.io.

## Additional Notes
See NOTES.md for details about challenges, blockers, and possible improvements.

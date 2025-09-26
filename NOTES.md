# Notes about the Project

## Challenges and Errors Encountered

### WSL installation and setup
- Initially WSL would not start and exited immediately.
- Required enabling **Virtualization (SVM)** in BIOS and enabling **Windows Subsystem for Linux** and **Virtual Machine Platform** features.
- After enabling, WSL2 was installed and an Ubuntu environment was prepared for running Ansible.

### SSH and server access
- The provided certificate needed to be renamed (`cert.pem`) and permissions fixed with `chmod 600 ~/.ssh/cert.pem`.
- In `hosts.ini` the wrong file name (`wp-cert.pem`) was used at first; changed to `cert.pem`.
- The first Ansible `ping` failed because of missing or misconfigured key, fixed after correcting the path.

### Ansible playbook errors
- **Docker GPG key task failed**: error `/bin/sh: 1: set: Illegal option -o pipefail`.  
  - Fix: added `executable: /bin/bash` to the task so Bash would handle `pipefail`.

### Docker and Docker Compose
- Docker installed successfully, but warning appeared about `version: "3.9"` being obsolete in `docker-compose.yml`.  
  - Fix: can remove the `version` key, but left it in place as it does not break execution.

### Caddy reverse proxy and HTTPS
- First error: container `caddy` failed with  
  `unable to start container process: error mounting "/opt/wordpress/Caddyfile"... not a directory`.  
  - Cause: `/opt/wordpress/Caddyfile` was a directory instead of a file.  
  - Fix: removed it, recreated as a file with the correct configuration.

- Second error: `Bind for 0.0.0.0:80 failed: port is already allocated`.  
  - Cause: WordPress container still had `ports: - "80:80"`.  
  - Fix: removed the port mapping from `wordpress` in `docker-compose.yml.j2`, leaving ports only for Caddy.

### WordPress application issues
- After HTTPS setup, WordPress opened but appeared broken (CSS/JS not loading).  
  - Cause: WordPress still thought the site was at `http://13.48.24.171`.  
  - Fix: updated `home` and `siteurl` directly in the database using a PHP command inside the container:
    ```bash
    docker compose exec -u www-data wordpress php -r \
    "require 'wp-load.php'; update_option('home','https://13-48-24-171.sslip.io'); update_option('siteurl','https://13-48-24-171.sslip.io'); echo \"OK\n\";"
    ```

- Attempt to use `WORDPRESS_CONFIG_EXTRA` in the Compose file caused HTTP 500 due to formatting issues in injected PHP code.  
  - Fix: removed it from the Compose file and instead added a small snippet directly to `wp-config.php` for handling `X-Forwarded-Proto`.

### Final working setup
- WordPress reachable at `https://13-48-24-171.sslip.io/` with valid Let’s Encrypt certificate.  
- Caddy successfully reverse-proxies WordPress.  
- Firewall (UFW) allows only ports 22, 80, and 443.  
- SSH hardened (no root login, no password login, key-only).  
- Fail2ban enabled.  
- Automatic security updates enabled.

## Improvements with More Time
- Move all passwords from `group_vars/all.yml` into **Ansible Vault** for security.
- Add **automated backups** (mysqldump + WordPress volumes to remote storage).
- Set up a real **domain name** instead of sslip.io for production readiness.
- Break the playbook into **roles** (`security`, `docker`, `wordpress`, `proxy`) for cleaner structure.
- Integrate into a **CI/CD pipeline** for automatic deployments from Git.

## Key Learnings
- How to structure an Ansible project with inventories, variables, templates, and playbooks.
- How to debug common Ansible and Docker issues.
- How to use Caddy and sslip.io for simple HTTPS without owning a domain.
- How WordPress manages its site URL and why it needs to be updated when moving behind a reverse proxy.

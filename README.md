# Ansible Home Lab Deployment

This project provides a comprehensive automated deployment system for a home lab environment using Ansible and Docker. It features a modular architecture with role-based service deployment, Traefik reverse proxy integration, and extensive monitoring capabilities. It assumes you have purchased a domain for setup and is using Cloudflare to ensure dynamic IP tagging. As a backup, it also offers no-ip dynamic client as a support service.

## Project Structure

```
deploy_home_lab_playbook.yml    # Main deployment playbook
group_vars/
├── main.yml                    # Default configuration variables
├── vault.yml                   # Sensitive data (credentials, API keys)
└── vault.yml.template          # Template for creating vault.yml
inventory/
└── hosts                       # Target machine definitions
roles/
├── base_network_watchtower/    # Network setup and container updates
│   ├── tasks/
│   └── templates/
├── core_infrastructure/        # Core services (Traefik, Pi-hole, Portainer)
│   ├── files/
│   ├── tasks/
│   └── templates/
├── coding_tools/               # Development tools (VS Code Server, IT Tools)
│   ├── tasks/
│   └── templates/
├── remote_access/              # Remote connectivity (WireGuard, Putty, NoIP)
│   ├── tasks/
│   └── templates/
└── dashboards/                 # Monitoring (Homepage, Glances, Uptime Kuma)
    ├── tasks/
    └── templates/
```

## Key Features

### Core Infrastructure
- **Traefik**: Reverse proxy with automatic SSL via Cloudflare
- **Pi-hole + Unbound**: DNS and ad-blocking with recursive DNS
- **Portainer**: Container management UI

### Development Tools
- **VS Code Server**: Browser-based code editor
- **IT Tools**: Collection of development utilities

### Remote Access
- **WireGuard**: VPN server with web UI
- **Cloudflare DDNS**: Dynamic DNS updates
- **NoIP**: Alternative dynamic DNS provider
- **Putty**: Web-based SSH client

### Monitoring
- **Homepage**: Customizable dashboard
- **Glances**: System monitoring
- **Uptime Kuma**: Service uptime monitoring

Note: The `roles/dashboards/templates/` folder contains pre-prepared homepage configuration files. Additional optional services and bookmark templates are maintained in the same GitHub account that owns this repository: https://github.com/cyph3ramfm — feel free to browse or reuse them.

## Prerequisites

Before you begin, ensure the target host(s) meet these prerequisites:
- Ubuntu Linux (or Debian based system)
- Docker Engine (20.x+) and Docker Compose v2 (or the Docker CLI with compose support)
- Python 3 and Ansible (2.10+ recommended)
- The `community.docker` collection installed (`ansible-galaxy collection install community.docker`)
- A registered domain and Cloudflare account (for automatic TLS via DNS-01). No-IP is supported as a fallback.
- Sudo/become access to the deployment user on target hosts (the playbook uses privilege escalation to create system-level directories)

If you need to install Ansible and Docker, you can leverage the scripts provided in: https://github.com/cyph3ramfm/ubuntu_install_ansible_docker

## Getting Started

1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd home_lab_setup
   ```

2. **Configure secrets**:
   ```bash
   # Create initial vault file
   cp group_vars/vault.yml.template group_vars/vault.yml

   # Create a secure password file (store safely!)
   openssl rand -base64 32 > ~/.vault_pass.txt
   chmod 600 ~/.vault_pass.txt

   # Encrypt your vault file
   ansible-vault encrypt --vault-password-file ~/.vault_pass.txt group_vars/vault.yml

   # Edit encrypted vault (will decrypt, open editor, then re-encrypt)
   ansible-vault edit --vault-password-file ~/.vault_pass.txt group_vars/vault.yml

   # View encrypted vault contents
   ansible-vault view --vault-password-file ~/.vault_pass.txt group_vars/vault.yml
   ```

   The vault file contains sensitive data like:
   - API keys (Cloudflare, NoIP)
   - Credentials (Traefik dashboard, Pi-hole)
   - Email settings (Watchtower notifications)
   - Domain names and configurations

3. **Configure deployment**:
   - Edit `inventory/hosts` to specify target machines
   - Modify `group_vars/main.yml` for non-sensitive configuration
   - Customize roles in `group_vars/main.yml`:
     ```yaml
     deploy_base_network_watchtower: true
     deploy_core_infrastructure: true
     deploy_coding_tools: true
     deploy_remote_access: true
     deploy_dashboards: true
     ```

4. **Run the deployment**:
    ```bash
    # Run playbook with vault password and prompt for privilege escalation (recommended for interactive runs)
    ansible-playbook -i inventory/hosts \
       --vault-password-file ~/.vault_pass.txt \
       --ask-become \
       deploy_home_lab_playbook.yml

    # Alternative: prompt for vault password (and become password)
    ansible-playbook -i inventory/hosts \
       --ask-vault-pass \
       --ask-become \
       deploy_home_lab_playbook.yml
    ```

   For development or testing without vault encryption:
    ```bash
    # Using --vault-id @prompt is more secure than --ask-vault-pass (prompts for both vault and become passwords)
    ansible-playbook -i inventory/hosts \
       --vault-id @prompt \
       --ask-become \
       deploy_home_lab_playbook.yml
    ```

## Network Architecture

The deployment uses two Docker networks:
- **home_lab_network**: Internal service communication (172.16.3.0/24)
- **proxy_network**: External services behind Traefik

## Security and Vault Management

### Managing Sensitive Data
All sensitive data is stored in `group_vars/vault.yml` and should be encrypted using ansible-vault. Common operations:

```bash
# Encrypt an existing vault file
ansible-vault encrypt group_vars/vault.yml

# Edit the vault (decrypts, opens editor, re-encrypts)
ansible-vault edit group_vars/vault.yml

# View vault contents
ansible-vault view group_vars/vault.yml

# Change vault password
ansible-vault rekey group_vars/vault.yml
```

For automated deployments, store the vault password in a secure file:
```bash
# Create secure password file
openssl rand -base64 32 > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt

# Use in commands
ansible-vault edit --vault-password-file ~/.vault_pass.txt group_vars/vault.yml
```

### Best Practices
- Never commit unencrypted vault files
- Use different vault passwords for development and production
- Regularly rotate vault passwords
- Backup vault password files securely
- Use `vault.yml.template` as reference for required variables

## Service Configuration

### Adding New Services
1. Create service files in the appropriate role:
   ```
   roles/role_name/
   ├── tasks/service.yml        # Deployment logic
   └── templates/service.yml.j2 # Docker Compose template
   ```

2. Add configuration variables to `group_vars/main.yml`

3. Include the task in the role's `main.yml`

### NFS share role

This repository includes a `nfs_share` role to help mount an NFS export from a NAS so container data (under `/var/docker_data/` or similar) can live on persistent storage outside the host.

What the role does (brief):

- Installs NFS client utilities (`nfs-common`) on Debian/Ubuntu hosts
- Creates the configured mount point (default permissions `0777` to avoid permission friction)
- Adds an entry to `/etc/fstab` for the configured NFS export and mounts it

Key variables (set these in `group_vars/main.yml` or `group_vars/vault.yml` as appropriate):

- `deploy_nfs_share` (bool) — set to `true` to enable the role
- `nfs_server` — IP or hostname of the NFS server (NAS)
- `nfs_path` — exported path on the NFS server (e.g., `/exports/docker_data`)
- `mount_point` — local mount point on the host (e.g., `/var/docker_data`)

Notes and recommendations:

- The role runs with `become: true` because mounting and editing `/etc/fstab` require elevated privileges.
- The default directory mode is `0777` to reduce permission issues; consider tightening ownership and permissions after the first successful mount to match container UIDs/GIDs.
- Ensure the NAS export has appropriate export options (e.g., `rw`, `no_root_squash` if necessary) and that your network and firewall allow NFS traffic.
- After enabling and running this role, subsequent roles that create `/var/docker_data/...` directories will operate against the mounted NAS location.

### External Access
Services requiring external access must include Traefik labels:
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.service.rule=Host(`prefix.domain`)"
  - "traefik.http.services.service.loadbalancer.server.port=PORT"
```

## Troubleshooting

1. **Network Issues**:
   - Verify Docker networks exist
   - Check Traefik is running
   - Validate DNS resolution

2. **Service Deployment**:
   - Enable `debug_mode: true` in `group_vars/main.yml` to keep rendered compose files in `/tmp/<role>/` for inspection.
   - Check rendered files in `/tmp/<role>/` (templates are rendered there before `docker_compose_v2` is called).
   - View container logs through Portainer or `docker logs <container>`.

### Quick checks for beginners

- If you see permission errors when creating directories under `/var/docker_data/`, run the playbook with `--ask-become` so Ansible can escalate privileges.
- If Traefik fails to obtain certificates, verify `vault_cf_api_email` and `vault_cf_dns_api_token` (Cloudflare token) are populated in `group_vars/vault.yml`.
- For DNS issues: ensure the `home_lab_network` and `proxy` Docker networks are present (the playbook creates them if missing) and that Pi-hole is configured as your primary DNS if you're using the homepage local DNS hints.

3. **Common Fixes**:
   - Network dependencies: Ensure proper role execution order

## License

This project is licensed under the MIT License. See the LICENSE file for more details.
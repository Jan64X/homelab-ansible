# Setup Guide

> **🔐 SECURITY FIRST**
> 
> This playbook makes significant system changes including:
> - User account creation and modification
> - Firewall rule configuration  
> - Service installation and configuration
> - SSH access modifications
>
> **ALWAYS** test in a non-production environment first!

This guide walks you through setting up this Ansible playbook to manage your infrastructure.

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [SSH Key Configuration](#ssh-key-configuration)
4. [Inventory Configuration](#inventory-configuration)
5. [Variable Configuration](#variable-configuration)
6. [Credential Management](#credential-management)
7. [First Run](#first-run)
8. [Service-Specific Setup](#service-specific-setup)
9. [Troubleshooting](#troubleshooting)

## Prerequisites

> **⚠️ CRITICAL: Host Preparation Required**
> 
> This playbook does NOT perform initial host setup. You must manually prepare each host with:
> - Debian 12 installed
> - An `ansible` user with UID/GID 1000
> - Passwordless sudo configured
> - SSH key authentication enabled
> - Python 3 installed
>
> Attempting to run this playbook on unprepared hosts will fail.

### Control Node (Your Machine)
- **Ansible**: Version 2.9 or later
  ```bash
  # Ubuntu/Debian
  sudo apt update
  sudo apt install ansible
  
  # macOS
  brew install ansible
  
  # Python pip
  pip3 install ansible
  ```

- **Git**: For cloning and version control
  ```bash
  sudo apt install git  # Ubuntu/Debian
  brew install git      # macOS
  ```

### Managed Hosts

**IMPORTANT**: This playbook assumes your managed hosts have already been prepared with the following configuration during OS installation:

- **Debian 12 OS**
- **Python 3** installed (usually included by default)
- **SSH server** running and accessible
- **User Setup** (configured during OS installation):
  - An `ansible` user exists with **UID/GID 1000**
  - `sudo` package is installed
  - The `ansible` user has **passwordless sudo** access
  - The Ansible public key is already in `/home/ansible/.ssh/authorized_keys`

#### How to Prepare a Fresh Debian Host

If you're setting up a new host from scratch, during Debian installation:

1. **Create the ansible user** when prompted for user creation
   - Username: `ansible`
   - This will automatically get UID/GID 1000 (first user)

2. **After installation**, configure passwordless sudo:
   ```bash
   # Login as ansible user or switch to it
   su - ansible
   
   # Add ansible user to sudo group (if not already)
   sudo usermod -aG sudo ansible
   
   # Configure passwordless sudo
   echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers
   ```

3. **Add your Ansible SSH public key**:
   ```bash
   # On the managed host as ansible user
   mkdir -p ~/.ssh
   chmod 700 ~/.ssh
   echo "your-ansible-public-key-here" >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

### Network Requirements
- SSH access (port 22) from control node to all managed hosts
- Internet access on managed hosts for package installation
- (Optional) NAS accessible via CIFS/SMB for backups

## Initial Setup

### 1. Clone the Repository

```bash
git clone <your-repo-url> ansible
cd ansible
```

### 2. Verify Directory Structure

Ensure the following structure exists:
```
ansible/
├── credentials/
│   ├── hosts/
│   └── ssh_keys/
├── group_vars/
├── inventory/
└── roles/
```

## SSH Key Configuration

### 1. Generate SSH Keys

You need to create SSH keys for different purposes:

```bash
cd credentials/ssh_keys/
```

#### Ansible Admin Key
Used by Ansible to connect to all hosts:
```bash
ssh-keygen -t ed25519 -f ansible_admin.key -C "ansible-admin"
mv ansible_admin.key.pub ansible_admin.pub
```

#### Sysadmin Keys
Personal SSH keys for manual access:

**For homelab hosts:**
```bash
ssh-keygen -t ed25519 -f sysadmin_homelab.key -C "sysadmin-homelab"
mv sysadmin_homelab.key.pub sysadmin_homelab.pub
```

**For VPS hosts:**
```bash
ssh-keygen -t ed25519 -f sysadmin_vpses.key -C "sysadmin-vpses"
mv sysadmin_vpses.key.pub sysadmin_vpses.pub
```

### 2. Verify SSH Access

Since your hosts should already have the ansible user configured with the public key, verify SSH access:

```bash
ssh -i credentials/ssh_keys/ansible_admin.key ansible@<host-ip>
```

If this works without prompting for a password, you're ready to proceed.

## Inventory Configuration

### 1. Create Inventory File

```bash
cp inventory/hosts.yml.example inventory/hosts.yml
```

### 2. Edit Inventory

Edit `inventory/hosts.yml` and replace example values:

```yaml
all:
  children:
    homelab:
      hosts:
        nginx-gateway:
          ansible_host: 192.168.1.10  # Replace with actual IP
          ansible_user: ansible       # Pre-configured ansible user
          ansible_python_interpreter: /usr/bin/python3
          ansible_become_method: sudo
          system_shell: /bin/bash
          os_family: Debian
          host_roles:
            - base
            - nginx_gateway
          hostname: nginx-gateway
        # Add more homelab hosts...
    
    vpses:
      hosts:
        vps1:
          ansible_host: 203.0.113.10  # Replace with VPS IP
          ansible_user: ansible       # Pre-configured ansible user
          # ... rest of configuration
```

**Key Points:**
- Replace `x.x.x.x` with actual IP addresses
- Use `ansible` as the user (should already be configured on hosts)
- The ansible user should already have passwordless sudo
- Adjust `host_roles` to control which services run on each host

### 3. Host Role Assignment

Available roles:
- `base` - Base configuration (recommended for all hosts)
- `nginx_gateway` - Nginx reverse proxy
- `gitlab` - GitLab CE
- `sharkey` - Sharkey/Misskey server
- `searxng` - SearXNG search
- `immich` - Immich photos
- `navidrome` - Navidrome music
- `torrent-down` - qBittorrent + Flood
- `filescdn` - File CDN
- `unificontroller` - UniFi controller
- `observability` - Prometheus + Grafana
- `monitoring-hl` - Node Exporter (homelab)
- `monitoring-vps` - Node Exporter (VPS)

Example minimal setup:
```yaml
host_roles:
  - base
  - monitoring-hl
```

## Variable Configuration

### 1. Global Variables

```bash
cp group_vars/all.yml.example group_vars/all.yml
```

Edit `group_vars/all.yml`:

```yaml
---
# System Configuration
# The ansible user (UID/GID 1000) already exists on hosts
# This creates an ADDITIONAL sysadmin user for manual access
system_user: sysadmin          # Your personal username on managed systems
system_user_groups:
  - wheel
  - ssh

ansible_user: ansible          # Ansible automation user (already exists)
ansible_user_groups: 
  - ssh

# SSH Configuration
# Homelab Configuration
# SSH key for all homelab hosts
sysadmin_ssh_key: "{{ lookup('file', playbook_dir + '/credentials/ssh_keys/sysadmin_homelab.pub') }}"

# Base domain for all services (change this to your domain)
base_domain: example.com

# Nginx Configuration for public-facing servers
ssl_enabled: true
cloudflare_enabled: true      # Set to false if not using Cloudflare
nginx_ssl_cert_path: /etc/nginx/ssl/{{ base_domain }}.pem
nginx_ssl_key_path: /etc/nginx/ssl/{{ base_domain }}.key
nginx_ssl_client_cert_path: /etc/nginx/ssl/origin_ca_rsa_root.pem
domain_name: "{{ base_domain }}"

# SSL certificates to copy from NAS
# Each certificate is copied from NAS path: /tank/Server/nginx/ssl/<src>
nginx_ssl_certificates:
  - src: "{{ base_domain }}.pem"
    dest: "{{ nginx_ssl_cert_path }}"
  - src: "{{ base_domain }}.key"
    dest: "{{ nginx_ssl_key_path }}"
  - src: "origin_ca_rsa_root.pem"
    dest: "{{ nginx_ssl_client_cert_path }}"

# Nginx virtual hosts to configure
# Each entry corresponds to a template file: templates/<name>.conf.j2
nginx_vhosts:
  - search
  - gitlab
  - base_domain
  - sharkey
  - filescdn

# Service port mappings (ports exposed by each service)
# Adjust these if your services use different ports
service_ports:
  search: 8080       # SearXNG default port
  gitlab: 80         # GitLab HTTP port
  sharkey: 3000      # Sharkey/Misskey default port
  filescdn: 8080     # File CDN service port

# Services configuration
# These use hostvars to automatically pull backend IPs from inventory/hosts.yml
services:
  search:
    domain: "search.{{ base_domain }}"
    backend_server: "http://{{ hostvars['searxng']['ansible_host'] }}:{{ service_ports.search }}"
  gitlab:
    domain: "git.{{ base_domain }}"
    backend_server: "http://{{ hostvars['gitlab']['ansible_host'] }}:{{ service_ports.gitlab }}"
  sharkey:
    domain: "fedi.{{ base_domain }}"
    backend_server: "http://{{ hostvars['sharkey']['ansible_host'] }}:{{ service_ports.sharkey }}"
  filescdn:
    domain: "cdn.{{ base_domain }}"
    backend_server: "http://{{ hostvars['filescdn']['ansible_host'] }}:{{ service_ports.filescdn }}"

# Service base URLs (used by various roles)
searxng_base_url: "https://search.{{ base_domain }}"
sharkey_base_url: "https://fedi.{{ base_domain }}"

# GitLab configuration (homelab specific)
gitlab_force_restore: false
gitlab_force_fresh_install: false

# monitoring - observability role with prometheus and grafana
monitoring_force_fresh_install: false

# immich backup restoration settings, set to true to force a fresh install
immich_force_restore: false

# NAS IP address (adjust to your network)
nas_ip: "x.x.x.x"

# Static website configuration
# make a public repo with index.html, styles.css, script.js and so on and replace the placeholders below
static_website:
  enabled: true
  repo_url: "https://github.com/yourusername/yourwebsite.git"
  repo_branch: "main"
  local_path: "/srv/nginx/{{ base_domain }}"
  github_api_url: "https://api.github.com/repos/yourusername/yourwebsite/commits/main"

```

### 3. VPS Variables

```bash
cp group_vars/vpses.yml.example group_vars/vpses.yml
```

## Credential Management

### 1. NAS Credentials (for NAS backups)

Create `credentials/nas_creds.txt`:

```yaml
---
nas_user: your_nas_username
nas_password: your_nas_password
```

### 2. Host-Specific Credentials

The playbook automatically generates credentials for each host in:
```
credentials/hosts/<hostname>/
├── root_pass.txt
├── ansible_pass.txt
├── <system_user>_pass.txt    # your system_user name
└── salts/
```

**Important:** These are auto-generated on first run. Keep them secure!

## First Run

### 1. Verify Host Prerequisites

Before running the playbook, verify each managed host meets the requirements:

```bash
# Test SSH access with the ansible user
ssh -i credentials/ssh_keys/ansible_admin.key ansible@<host-ip>

# Once logged in, verify:
# 1. Check UID/GID of ansible user
id ansible
# Should show: uid=1000(ansible) gid=1000(ansible)

# 2. Verify passwordless sudo
sudo whoami
# Should return "root" without asking for password

# 3. Check Python is available
python3 --version

# 4. Exit back to control node
exit
```

### 2. Test Connectivity

Verify Ansible can reach all hosts:
```bash
ansible all -m ping
```

Expected output:
```
nginx-gateway | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### 2. Dry Run (Check Mode)

Test without making changes:
```bash
ansible-playbook site.yml --check
```

### 3. Run the Playbook

Since the ansible user and SSH access are already configured, you can run the playbook directly:

```bash
ansible-playbook site.yml
```

Or start with just the base role to configure the foundation:

Edit `site.yml` and uncomment the base role:
```yaml
  roles:
    - role: base  # Always applied to all hosts
```

Then run:
```bash
ansible-playbook site.yml
```

This will configure:
- System users (creating the sysadmin user with UID/GID defined in variables)
- SSH access for the sysadmin user
- UFW firewall rules
- Security packages (fail2ban, etc.)
- Base system utilities

### 4. Deploy Services

If you ran only the base role first, comment it out in `site.yml`:
```yaml
#  roles:
#    - role: base  # Always applied to all hosts
```

Then run the full playbook to deploy services based on `host_roles`:
```bash
ansible-playbook site.yml
```

Or target specific hosts:
```bash
ansible-playbook site.yml --limit nginx-gateway,gitlab
```

## Service-Specific Setup

### Nginx Gateway

1. **SSL Certificates**: 
   - If using Cloudflare, obtain Origin CA certificates
   - Place in NAS at configured path, or modify role to use Let's Encrypt
   - Certificates are copied from NAS during deployment

2. **DNS Configuration**:
   - Point your domains to the nginx-gateway IP (public IP)
   - Configure Cloudflare DNS (if using)

### GitLab

1. **Backup/Restore**:
   - On first run, set `gitlab_force_fresh_install: true` for fresh install
   - Or place backup in NAS and set `gitlab_force_restore: true`

2. **Root Password**:
   - Located in `credentials/hosts/gitlab/gitlab_root_pass.txt`

3. **Access**: `https://git.yourdomain.com`

### Sharkey

1. **Initial Setup**:
   - After deployment, access `https://fedi.yourdomain.com`
   - Complete web-based setup wizard
   - Admin credentials in `credentials/hosts/sharkey/`

### Observability (Prometheus + Grafana)

1. **Grafana Access**:
   - Access: `http://<observability-host>:80`
   - Username: `admin`
   - Password: In `credentials/hosts/observability/grafana_admin_password.txt`

2. **Prometheus**:
   - Configured to scrape all hosts with `monitoring-hl` or `monitoring-vps` roles
   - Access: `http://<observability-host>:9090`

### Other Services

Each service has auto-generated credentials in `credentials/hosts/<hostname>/`.
Check individual role templates for specific configuration details.

## Troubleshooting

### SSH Connection Issues

**Problem**: "Permission denied (publickey)"
```bash
# Verify key is loaded
ssh-add -l

# Add key if needed
ssh-add credentials/ssh_keys/ansible_admin.key

# Test direct connection
ssh -i credentials/ssh_keys/ansible_admin.key ansible@<host-ip>
```

### Python Not Found

**Problem**: "/usr/bin/python3: not found"
```bash
# Install Python on target host
ssh root@<host-ip>
apt update && apt install python3
```

### Privilege Escalation Failed

**Problem**: "Missing sudo password" or sudo issues
- Ensure the ansible user has passwordless sudo configured:
  ```bash
  # On the managed host
  sudo cat /etc/sudoers.d/ansible
  # Should contain: ansible ALL=(ALL) NOPASSWD:ALL
  ```
- Verify `ansible_become_method: sudo` is set in inventory
- Test sudo manually:
  ```bash
  ssh -i ./credentials/ssh_keys/ansible_admin.key ansible@<host-ip> 'sudo whoami'
  # Should return: root
  ```

### Role Not Applied

**Problem**: Service role not running
- Check `host_roles` in inventory includes the role
- Verify role name spelling matches directory in `roles/`
- Check for errors in previous tasks

### NAS Mount Failed

**Problem**: Cannot mount NAS
- Verify NAS IP in `group_vars/homelab.yml`
- Check `credentials/nas_creds.txt` has correct credentials
- Ensure NAS share exists and is accessible
- Test manual mount:
  ```bash
  sudo mount -t cifs //nas-ip/share /mnt/test -o username=user,password=pass
  ```

### Firewall Blocking Services

**Problem**: Cannot access service after deployment
- Check UFW status: `sudo ufw status`
- Verify port is allowed in role's firewall tasks
- Check if service is running: `sudo systemctl status <service>`

### Certificate Errors

**Problem**: SSL/TLS errors on nginx
- Verify certificates exist in `/etc/nginx/ssl/`
- Check certificate validity: `openssl x509 -in cert.pem -text -noout`
- Ensure NAS mount was successful during deployment

## Advanced Configuration

### Using Ansible Vault

Encrypt sensitive files:
```bash
ansible-vault encrypt group_vars/all.yml
ansible-playbook site.yml --ask-vault-pass
```

### Custom SSH Port

If using non-standard SSH port:
```yaml
# inventory/hosts.yml
ansible_port: 2222
ssh_port: 2222
```

### Limiting Execution

Run specific roles:
```bash
ansible-playbook site.yml --tags base
```

Run specific hosts:
```bash
ansible-playbook site.yml --limit homelab
ansible-playbook site.yml --limit nginx-gateway
```

### Parallel Execution

Adjust forks in `ansible.cfg`:
```ini
[defaults]
forks = 20  # Number of parallel processes
```

## Post-Setup Maintenance

### Regular Updates

Update all systems:
```bash
ansible-playbook update.yml
```

### Backup Verification

Regularly test backup restoration on non-production hosts.

### Credential Rotation

Periodically rotate:
- SSH keys
- Service passwords
- SSL certificates

### Monitoring

1. Access Grafana dashboards
2. Verify all hosts are reporting metrics
3. Set up alerting rules in Grafana

## Next Steps

1. **Customize Services**: Adjust role variables for your needs
2. **Add Monitoring Dashboards**: Import or create Grafana dashboards
3. **Set Up Backups**: Configure automated backup schedules
4. **Document Changes**: Keep notes on customizations
5. **Test DR Procedures**: Practice restoring from backups

## Getting Help

- Review role-specific `tasks/main.yml` for detailed steps
- Consult Ansible documentation: https://docs.ansible.com/ (RTFM)
- Review service-specific documentation for applications

---

**Setup Complete!** 🎉

Your infrastructure should now be managed by Ansible. Run `ansible-playbook site.yml` anytime to ensure configuration compliance.

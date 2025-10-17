# JupyterHub + Caddy Deployment (Ansible)

This project automates the deployment of a **secure, production-ready JupyterHub** instance behind **Caddy** with automatic HTTPS (via ACME / Let's Encrypt).  
It installs all dependencies, configures JupyterHub, creates a service user, and enables HTTPS using your DNS domain.  
Everything runs idempotently through modular Ansible roles.

---

## 🧭 Overview

| Component | Purpose |
|------------|----------|
| **JupyterHub** | Multi-user notebook server (Python virtual environment at `/opt/jhub-venv`) |
| **Caddy** | Reverse proxy and automatic HTTPS via Let's Encrypt |
| **PAM Authentication** | Local Linux user authentication for JupyterHub |
| **Ansible Roles** | Modular structure for reuse and clean configuration |
| **Systemd Services** | Persistent management of Caddy and JupyterHub |

---

## 🧩 Folder Structure

```
ansible/
├── inventory/
│   └── hosts
├── group_vars/
│   └── all.yml
├── site.yml
├── roles/
│   ├── common/
│   │   └── tasks/main.yml
│   ├── caddy/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/Caddyfile.j2
│   └── jupyterhub/
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       ├── templates/jupyterhub_config.py.j2
│       └── files/pam.d.jupyterhub
└── README.md
```

---

## ⚙️ Configuration

All global variables are defined in `group_vars/all.yml`:

```yaml
domain_name: "lab.example.com"     # DNS entry pointing to this machine
acme_email: "admin@example.com"    # Email for Let's Encrypt notifications

allow_all: true                    # Allow all valid Linux users to log in
# allow_all: false                  # Restrict to specific users
allowed_users: ["gerrit", "alice"]
admin_users: ["gerrit"]            # Hub admin users

jhub_user: jhub                    # System account for JupyterHub service
venv_dir: /opt/jhub-venv
hub_state_dir: /var/lib/jupyterhub
hub_cfg_dir: /etc/jupyterhub

manage_ufw: true                   # Open ports 22, 80, 443 if UFW is active
```

---

## 🏗️ How to Deploy

### 1. Prepare DNS and Firewall
- Create an **A record** for your domain (e.g., `lab.example.com`) pointing to your server’s public IP.
- Allow inbound **TCP 80** and **TCP 443** (HTTP + HTTPS).

### 2. Run the Playbook
```bash
cd ansible
sudo apt update && sudo apt install -y ansible
ansible-playbook -i inventory/hosts site.yml
```

### 3. Access JupyterHub
- Open **https://lab.example.com** in a browser.
- Log in with any valid Linux user (create with `sudo adduser <name>`).
- Admin users (`admin_users`) can access **/hub/admin**.

---

## 🔒 Security

- HTTPS certificates are automatically managed via the [ACME protocol](https://datatracker.ietf.org/doc/html/rfc8555) (Let’s Encrypt).
- `jupyterhub_cookie_secret` is generated once at `/var/lib/jupyterhub/jupyterhub_cookie_secret` for session security.
- JupyterHub runs under a **non-root** service account (`jhub`) for safety.
- All user authentication is done via **PAM** using local Linux credentials.
- Caddy automatically renews certificates — no manual cron jobs required.

---

## 🧰 Service Management

| Action | Command |
|--------|----------|
| View Caddy logs | `journalctl -u caddy -n 100 --no-pager` |
| View JupyterHub logs | `journalctl -u jupyterhub -n 100 --no-pager` |
| Restart JupyterHub | `sudo systemctl restart jupyterhub` |
| Restart Caddy | `sudo systemctl restart caddy` |
| Enable both at boot | `sudo systemctl enable jupyterhub caddy` |

---

## 🧪 Verification

After deployment, verify both internal and external routes:

```bash
# Local service check
curl -I http://127.0.0.1:8000/hub/login

# External HTTPS test
curl -vkI https://lab.example.com/hub/login
```

Expected response:
```
HTTP/1.1 200 OK
Via: 1.1 Caddy
X-JupyterHub-Version: 5.x.x
```

---

## 🧠 Technical Notes

### JupyterHub Cookie Secret
The file `/var/lib/jupyterhub/jupyterhub_cookie_secret` contains a random 32-byte key used to **sign and verify session cookies**.  
If this file changes or is deleted, all users will be logged out and new sessions will be created.

### ACME (Automatic Certificate Management Environment)
Caddy uses the ACME protocol to automatically request and renew TLS certificates from Let’s Encrypt.  
No manual certificate handling or renewal tasks are required.

### Service User
The system user `jhub` (or the name specified in `group_vars/all.yml`) runs the JupyterHub service.  
It uses `/usr/sbin/nologin` as its shell to prevent interactive logins, following least-privilege security principles.

### Handlers
Ansible **handlers** (e.g., `reload caddy`, `reload jupyterhub`) are triggered only when related configuration files change.  
This avoids unnecessary service restarts and keeps the deployment idempotent.

---

## 🧰 Extending the Playbook

You can easily extend this setup to:
- **Isolate user storage** → mount `/data/{{ username }}` for each user.
- **Integrate with HPC or Slurm** → replace the default spawner with `BatchSpawner` or `SlurmSpawner`.
- **Support multi-node environments** → modify `inventory/hosts` for multiple machines.
- **Add OAuth or LDAP authentication** → use `OAuthenticator` instead of PAM.

---

## 🪪 License

MIT License.  
Created by **Gerrit Botha**, 2025.  
Includes open-source components: [JupyterHub](https://jupyter.org/hub), [Caddy](https://caddyserver.com), [Ansible](https://ansible.com), and [Let’s Encrypt](https://letsencrypt.org).

---

## 💬 Troubleshooting

If login or proxy fails, check:

```bash
journalctl -u jupyterhub -n 50
journalctl -u caddy -n 50
```

Typical issues:
- **DNS not resolving** → fix A-record or hosts entry.  
- **Firewall blocking 80/443** → open ports or disable upstream proxy restrictions.  
- **Permission errors** → ensure `/etc/jupyterhub` and `/var/lib/jupyterhub` are owned by the service user.

---

## 🧭 Architecture Diagram

```
       ┌──────────────┐
       │   Browser    │
       │ (User Login) │
       └──────┬───────┘
              │ HTTPS (443)
              ▼
       ┌──────────────┐
       │    Caddy     │
       │ (Reverse Proxy)
       │  Auto-HTTPS  │
       └──────┬───────┘
              │ HTTP (127.0.0.1:8000)
              ▼
       ┌──────────────┐
       │ JupyterHub   │
       │ (PAM Auth,   │
       │ Spawns Lab)  │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │ JupyterLab   │
       │ (User Server)│
       └──────────────┘
```

---

✨ **After deployment:**  
You’ll have a fully functional, HTTPS-secured JupyterHub accessible at your chosen domain — automatically renewed, managed, and easy to extend through Ansible.
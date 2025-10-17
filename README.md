# JupyterHub + Caddy Deployment (Ansible)

This project automates the deployment of a **secure, production-ready JupyterHub** instance behind **Caddy** with automatic HTTPS (via ACME / Let's Encrypt).  
It installs all dependencies, configures JupyterHub, enforces web-level security, restricts access to a Linux group, and disables terminals in JupyterLab for a tighter attack surface.  
Everything runs idempotently through modular Ansible roles.

---

## 🧭 Overview

| Component | Purpose |
|------------|----------|
| **JupyterHub** | Multi-user notebook server (Python venv at `/opt/jhub-venv`) |
| **Caddy** | Reverse proxy and automatic HTTPS via Let's Encrypt |
| **PAM Authentication** | Local Linux user authentication for JupyterHub |
| **Idle Culler** | Automatically stops idle notebook servers |
| **Ansible Roles** | Modular structure for reuse and clean configuration |
| **Systemd Services** | Persistent management of Caddy, JupyterHub, and Idle Culler |

---

## ⚙️ Configuration

All global variables are defined in `group_vars/all.yml`:

```yaml
domain_name: "lab.example.com"     # DNS entry pointing to this machine
acme_email: "admin@example.com"    # Email for Let's Encrypt notifications

# JupyterHub access policy
allow_all: false                   # Restrict to specific users/groups
allowed_users: []                  # Not used when allow_all=false and group gate active
admin_users: ["gerrit"]            # Hub admin users

# Group gate (only these users can log in)
jhub_group_name: jhub
jhub_group_members: ["gerrit"]

# Core paths
jhub_user: jhub
venv_dir: /opt/jhub-venv
hub_state_dir: /var/lib/jupyterhub
hub_cfg_dir: /etc/jupyterhub

# Optional firewall management (if UFW active)
manage_ufw: true
```

---

## 🧰 Security Features

### ✅ HTTPS by Default
- Caddy provisions and renews TLS certificates via ACME (Let's Encrypt).  
- HSTS and safe response headers are set automatically.  
- All traffic between users and JupyterHub is fully encrypted.

### ✅ Reverse Proxy Isolation
- Caddy listens on ports 80/443 only and forwards to `127.0.0.1:8000`.  
- JupyterHub never binds to a public interface.

### ✅ Cookie and Session Hardening
JupyterHub is configured to:
```python
c.JupyterHub.trust_xheaders = True
c.JupyterHub.cookie_secure = True
c.JupyterHub.cookie_options = {"SameSite": "Lax", "httponly": True}
c.JupyterHub.cookie_max_age_days = 7
```
This ensures that:
- Cookies are HTTPS-only (`Secure` flag)
- They can’t be accessed via JavaScript (`HttpOnly`)
- SameSite=Lax prevents cross-site CSRF exploits

### ✅ PAM Group Restriction
Only Linux users in the `jhub` group may log in:
```python
c.PAMAuthenticator.allowed_groups = {"jhub"}
```
Define members via Ansible in `jhub_group_members`.

### ✅ Idle Culler (Automatic Logout of Idle Sessions)
The **idle culler** stops inactive notebook servers after 1 hour:
```bash
--timeout=3600 --cull-every=300 --concurrency=5
```
It runs as a separate systemd service (`jupyterhub-idle-culler`) and frees resources while reducing risk from forgotten sessions.

### ✅ Terminals Disabled
Terminals are disabled inside JupyterLab and Classic Notebook:
```python
c.Spawner.args = [
    "--ServerApp.terminals_enabled=False",
    "--NotebookApp.terminals_enabled=False"
]
```
This prevents shell access through the web UI.

### ✅ Least Privilege Service Account
JupyterHub runs as its own system user (`jhub`) with `/usr/sbin/nologin`, isolating it from root.

### ✅ VPN-Aware Deployment
The system assumes SSH access is already restricted through a secure VPN.  
No SSH hardening is applied in Ansible, keeping compatibility with your lab’s access policies.

---

## 🧱 System Architecture

```
       ┌──────────────┐
       │   Browser    │
       │ (User Login) │
       └──────┬───────┘
              │ HTTPS (443)
              ▼
       ┌──────────────┐
       │    Caddy     │
       │ Reverse Proxy│
       │ Auto-HTTPS    │
       └──────┬───────┘
              │ HTTP (127.0.0.1:8000)
              ▼
       ┌──────────────┐
       │ JupyterHub   │
       │ (PAM Auth,   │
       │ Group Gate)  │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │ JupyterLab   │
       │ (Terminals ❌│
       │ Idle Culler ✅)
       └──────────────┘
```

---

## 🏗️ How to Deploy

### 1️⃣ Prepare DNS and Firewall
- Create an **A record** for your domain (e.g. `lab.example.com`) pointing to your server’s public IP.
- Allow inbound **TCP 80** and **TCP 443** (HTTP + HTTPS).

### 2️⃣ Run the Playbook
```bash
cd ansible
sudo apt update && sudo apt install -y ansible
ansible-playbook -i inventory/hosts site.yml
```

### 3️⃣ Access the Hub
- Open **https://lab.example.com**
- Log in with a Linux user who is part of the `jhub` group
- Admins (`admin_users`) can visit `/hub/admin`

---

## 🔍 Verifying Security

**Check HTTPS headers:**
```bash
curl -I https://lab.example.com
```
You should see:
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
```

**Check JupyterHub cookie flags:**
- Open browser DevTools → Application → Cookies
- Verify `Secure` and `HttpOnly` flags are enabled.

**Check group gate:**
```bash
id gerrit  # must include 'jhub'
```

**Check idle culler:**
```bash
systemctl status jupyterhub-idle-culler
```

---

## 🧠 Technical Notes

| Component | Purpose |
|------------|----------|
| **jupyterhub_cookie_secret** | Random key for signing cookies; persisted at `/var/lib/jupyterhub/jupyterhub_cookie_secret` |
| **Idle Culler** | Terminates notebook servers after inactivity |
| **Allowed Groups** | PAM check ensures only defined group members can authenticate |
| **Spawner Args** | Disable terminal access |
| **Caddy** | Manages TLS certificates, HSTS, and proxy headers |
| **Systemd Units** | jupyterhub.service, caddy.service, jupyterhub-idle-culler.service |

---

## 🪪 License

MIT License  
Created by **Gerrit Botha**, 2025  
Includes open-source components: [JupyterHub](https://jupyter.org/hub), [Caddy](https://caddyserver.com), [Ansible](https://ansible.com), and [Let’s Encrypt](https://letsencrypt.org)

---

## 💬 Troubleshooting

| Issue | Fix |
|-------|-----|
| **Users can’t log in** | Ensure they’re in the `jhub` group: `usermod -aG jhub username` |
| **Terminals still visible** | Clear browser cache; verify `Spawner.args` are applied |
| **Idle culler not running** | `systemctl enable --now jupyterhub-idle-culler` |
| **Cookies not Secure** | Ensure you access via `https://` and `trust_xheaders=True` is set |
| **DNS or HTTPS fails** | Check that ports 80/443 are open and domain resolves to the machine |

---

✨ After deployment, you’ll have a **VPN-protected, HTTPS-secured JupyterHub** where:  
- Only authorized users can log in,  
- Idle servers self-terminate,  
- Terminals are disabled,  
- All communication is encrypted end-to-end.

# ZFS NAS Manager

A web-based administration console for an Ubuntu storage server using **ZFS**, **iSCSI**, and **Samba**. It provides a React dashboard backed by a FastAPI service that runs the system storage commands.

> [!WARNING]
> This project performs privileged storage and system-administration operations. It is intended for a trusted, private network and should not be exposed to the public internet. Review and harden authentication, authorization, input validation, CORS, secrets, and `sudoers` permissions before any production use.

## Features

- **ZFS management** — create, import, inspect, and delete pools; create and resize datasets and zvols; select compression.
- **Snapshots** — create, delete, clone, roll back, and schedule ZFS snapshots with retention cleanup.
- **iSCSI targets** — expose zvols as iSCSI targets and manage initiator ACLs.
- **Samba shares** — create and configure SMB shares, manage share permissions, restart Samba, and view audit logs.
- **User and group administration** — manage local Linux users, groups, Samba access, and delegated administrators.
- **System operations** — view disks, health, system information, network interfaces, services, and journal logs.
- **Session handling** — local-user login with JWT sessions and a browser inactivity timeout.

## Architecture

| Component | Technology | Default URL |
| --- | --- | --- |
| Web UI | React, TypeScript, Vite, Tailwind CSS | `http://localhost:5173` |
| API | Python, FastAPI, Uvicorn | `http://localhost:2435` |
| Storage services | ZFS, targetcli/iSCSI, Samba | Managed by the API |

The frontend derives the API hostname from the page URL and sends requests to port `2435`.

## Ubuntu Installation

- Ubuntu or another Linux distribution with compatible ZFS tooling
- Python 3.12+ and `venv`
- Node.js 20 and npm

Install the storage services and supporting tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y zfsutils-linux targetcli-fb open-iscsi samba samba-vfs-modules rear
sudo apt install -y net-tools curl npm python3-pip python3.12-venv
```

Install and activate Node.js 20 with NVM:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
source ~/.bashrc
nvm install 20
nvm use 20
```

## Enable Storage Services

Check that the ZFS services are enabled and active:

```bash
sudo systemctl status zfs.target
sudo systemctl status zfs-import-cache.service
sudo systemctl status zfs-import-scan.service
```

If they are not active, enable and start them:

```bash
sudo systemctl enable --now zfs.target
sudo systemctl enable zfs-import-cache.service zfs-import-scan.service
sudo systemctl start zfs-import-scan.service
```

Enable and start the iSCSI target service:

```bash
sudo systemctl status target
sudo systemctl enable --now target
```

Enable and start Samba:

```bash
sudo systemctl status smbd
sudo systemctl enable --now smbd
```

## Run Locally

Clone this repository, then open two terminals in the project directory.

### 1. Start the backend API

```bash
cd backend
python3 -m venv venv
source venv/bin/activate
python3 -m pip install --upgrade pip
pip install -r requirements.txt
python main.py
```

The API listens on `0.0.0.0:2435`. Interactive API documentation is available at `http://localhost:2435/docs`.

### 2. Start the frontend

In the second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open the URL printed by Vite (normally `http://localhost:5173`).

## Run with Systemd

Use systemd to start the backend and frontend automatically after the Ubuntu server boots. The examples below assume the repository is located at `/home/ubuntu/ZFS-NAS-Manager` and runs as the `ubuntu` user. Change both paths and the user if your installation differs.

> [!IMPORTANT]
> Do not run the backend service as `root`. Grant the service account only the minimum required `sudoers` permissions using `visudo`.

### 1. Create the backend service

Create `/etc/systemd/system/zfs-nas-backend.service`:

```ini
[Unit]
Description=ZFS NAS Manager Backend
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/ZFS-NAS-Manager/backend
ExecStart=/home/ubuntu/ZFS-NAS-Manager/backend/venv/bin/python3 main.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 2. Create the frontend service

Create `/etc/systemd/system/zfs-nas-frontend.service`:

```ini
[Unit]
Description=ZFS NAS Manager Frontend
After=network.target zfs-nas-backend.service

[Service]
Type=simple
User=ubuntu
Environment=HOME=/home/ubuntu
Environment=NVM_DIR=/home/ubuntu/.nvm
WorkingDirectory=/home/ubuntu/ZFS-NAS-Manager/frontend
ExecStart=/bin/bash -lc 'source "$NVM_DIR/nvm.sh" && nvm use 20 && npm run dev -- --host 0.0.0.0'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Install the frontend dependencies once before starting its service:

```bash
cd /home/ubuntu/ZFS-NAS-Manager/frontend
npm install
```

### 3. Enable and start both services

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now zfs-nas-backend.service zfs-nas-frontend.service
```

### 4. Check status and logs

```bash
sudo systemctl status zfs-nas-backend.service
sudo systemctl status zfs-nas-frontend.service

sudo journalctl -u zfs-nas-backend.service -f
sudo journalctl -u zfs-nas-frontend.service -f
```

Restart both services after changing the application code:

```bash
sudo systemctl restart zfs-nas-backend.service zfs-nas-frontend.service
```

## Login and Permissions

Login is based on local Linux user accounts. A user must either:

- belong to the system `sudo` group, or
- be added as a delegated administrator by an existing administrator.

The backend must be granted only the minimum command permissions necessary to operate ZFS, iSCSI, Samba, and related system services. Use `visudo` for any required `sudoers` configuration; do not grant unrestricted passwordless `sudo` to the application account.

## Production Checklist

Before deploying this application beyond local development:

- Put the frontend and API behind HTTPS and a reverse proxy.
- Restrict network access with a firewall or VPN.
- Store the JWT signing secret in a secure environment variable or secret manager; never use a hard-coded secret.
- Enforce authentication and role authorization on every API endpoint in the backend.
- Replace shell-string execution with validated arguments passed to `subprocess.run(..., shell=False)`.
- Restrict CORS to the exact frontend origin.
- Apply least-privilege `sudoers` rules and remove wildcard command permissions.
- Test all destructive operations on non-production storage first.

## Project Structure

```text
backend/
  main.py              # FastAPI routes and application startup
  zfs_manager.py       # ZFS pool, dataset, volume, and snapshot operations
  iscsi_backend.py     # iSCSI target and ACL operations
  samba_manager.py     # Samba shares, configuration, and permissions
  user_manager.py      # Linux user and group management
  snapshot_backend.py  # Snapshot scheduling and retention
frontend/
  src/pages/           # Dashboard and feature pages
  src/components/      # Application layout and UI components
  src/hooks/           # Authentication, API, toast, and idle-timeout hooks
```

## License

This repository is distributed under the terms in [LICENSE](LICENSE).

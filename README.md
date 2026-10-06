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

## Prerequisites

- Ubuntu or another Linux distribution with compatible ZFS tooling
- Python 3.10+ and `venv`
- Node.js 20+ and npm
- ZFS utilities: `zfsutils-linux`
- iSCSI target utilities: `targetcli-fb`
- Samba: `samba` and `samba-vfs-modules`

Example package installation on Ubuntu:

```bash
sudo apt update
sudo apt install -y zfsutils-linux targetcli-fb open-iscsi samba samba-vfs-modules \
  python3-pip python3-venv nodejs npm
```

Enable the required services on the storage host:

```bash
sudo systemctl enable --now zfs.target target smbd
```

## Local Development

### 1. Start the API

```bash
cd backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python main.py
```

The API listens on `0.0.0.0:2435`. Interactive API documentation is available at `http://localhost:2435/docs`.

### 2. Start the frontend

In another terminal:

```bash
cd frontend
npm install
npm run dev
```

Open the URL printed by Vite (normally `http://localhost:5173`).

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

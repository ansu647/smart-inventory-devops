# Environment Setup & Verification Guide

This document records the development environment inspection, tooling prerequisites, versions, and verification steps performed on the MacBook Air M3 workstation for the **Smart Inventory Management System** project.

---

## 1. Workstation Hardware & OS Architecture

- **Machine Model**: Apple MacBook Air (M3, Apple Silicon)
- **Architecture**: ARM64 (`aarch64` / `arm64`)
- **Kernel / OS**: Darwin macOS 26.5.2
- **Shell**: zsh

---

## 2. Tooling Prerequisites & Installed Versions

All development prerequisites were audited and verified directly on the host system:

| Tool | Required Version | Installed Version | Status | Location / Binary Path |
| :--- | :--- | :--- | :--- | :--- |
| **Homebrew** | Latest | `7.0.10` | Verified | `/opt/homebrew/bin/brew` |
| **Python** | 3.12+ | `3.12.5` | Verified | `/Library/Frameworks/Python.framework/Versions/3.12/bin/python3` |
| **pip** | 24+ | `24.2` | Verified | `.venv/lib/python3.12/site-packages/pip` |
| **Python venv**| Standard lib | Available | Verified | Built-in module |
| **Git** | 2.x | `2.55.0` | Verified | `/opt/homebrew/bin/git` |
| **Docker Desktop** | Desktop 4.x+ | Engine `29.8.2` | Verified & Running | `/Applications/Docker.app` (`/usr/local/bin/docker`) |
| **AWS CLI** | v2+ | `2.37.4` | Verified | `/opt/homebrew/bin/aws` |
| **OpenSSH** | Standard | `10.2p1` | Verified | `/usr/bin/ssh` |
| **cURL** | Standard | `8.7.1` | Verified | `/usr/bin/curl` |
| **GitHub CLI (`gh`)**| 2.x+ | `2.92.0` | Verified & Authenticated | `/opt/homebrew/bin/gh` |

> **Note on Jenkins**: As specified by project guidelines, **Jenkins LTS is deployed on AWS EC2**, not on the local MacBook M3 development workstation. The Java runtime required by Jenkins is also provisioned on EC2.

---

## 3. Dedicated Python Virtual Environment Setup

A dedicated virtual environment was provisioned to isolate development and testing dependencies:

```bash
# 1. Create dedicated virtual environment using Python 3.12
python3 -m venv .venv

# 2. Activate virtual environment
source .venv/bin/activate

# 3. Upgrade pip
pip install --upgrade pip

# 4. Install production dependencies
pip install -r requirements.txt

# 5. Install development & testing dependencies
pip install -r requirements-dev.txt
```

### Installed Python Packages
- `Flask==3.0.3` (WSGI web microframework)
- `gunicorn==22.0.0` (Production WSGI HTTP server)
- `pytest==8.2.2` (Automated testing framework)
- `pytest-flask==1.3.0` (Flask test client fixtures)
- `Jinja2==3.1.6` (Template engine)
- `Werkzeug==3.1.9` (WSGI web utility library)

---

## 4. Local Verification & Test Execution

### Automated Pytest Suite
Run the test suite inside the virtual environment:
```bash
.venv/bin/pytest -v
```
**Result**: 19 out of 19 tests passed (100% pass rate) testing application routes, database constraints, negative stock prevention, search/filtering, and the `/health` endpoint.

### Local Docker Build & Persistence Test
The container was built and validated on Apple Silicon:
```bash
# Build Docker image
docker build -t smart-inventory:test .

# Run with persistent volume
docker volume create inventory_data_local_test
docker run -d --name test-inventory -p 5055:5000 -v inventory_data_local_test:/data smart-inventory:test

# Verify health endpoint
curl -s http://127.0.0.1:5055/health
# Response: {"database":"connected","service":"smart-inventory","status":"healthy"}

# Clean up
docker stop test-inventory && docker rm test-inventory
docker volume rm inventory_data_local_test
```

---

## 5. Security & Credentials Policy
- `.gitignore` prevents inadvertent commits of virtual environments (`.venv/`), SQLite database files (`instance/*.db`), `.DS_Store`, and environment files (`.env`).
- Never commit private keys, AWS access secrets, or personal access tokens to the repository.

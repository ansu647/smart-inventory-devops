# NexStock Enterprise | Smart Cloud Inventory Management System
> An enterprise-grade, Flask-based inventory management platform deployed through an automated Jenkins CI/CD pipeline on AWS EC2 using Git and Docker with full Indian Rupee (₹) currency support.

---

## 📌 Executive Summary & Architecture

The **Smart Inventory Management System (NexStock)** is a production-grade, administrative web platform designed to monitor physical inventory, track stock movements, enforce zero-deficit inventory policies, and alert teams to low stock in real time with Indian Rupee (₹) pricing.

Built strictly to modern DevOps standards, any code change committed to GitHub automatically triggers the CI/CD pipeline in real time:
```
Developer Commit & Push (MacBook M3)
          │
          ▼
GitHub Remote Repository (`main` branch)
          │
          ├── [Instant Trigger] GitHub Push Webhook (`/github-webhook/`)
          └── [Fallback Trigger] SCM Polling (`H/2 * * * *`)
          │
          ▼
Jenkins LTS (AWS EC2 Ubuntu 24.04)
          ├── Stage 1: Checkout Source Code
          ├── Stage 2: Isolated Virtualenv & Pytest (20/20 Passing)
          ├── Stage 3: Docker Image Build (`smart-inventory:${BUILD_NUMBER}`)
          ├── Stage 4: Container Deployment (Port 80:5000, Persistent Volume `inventory_data`)
          └── Stage 5: Health Check Verification (GET /health -> HTTP 200)
          │
          ▼
AWS EC2 Live Public Endpoint (http://54.87.5.184/)
```

---

## 🛠️ Technology Stack

| Domain | Technology | Implementation Detail |
| :--- | :--- | :--- |
| **Backend** | Python 3.12+ / Flask 3.0 | Application factory, modular routing, context processors |
| **Database** | SQLite 3 | Embedded, ACID transactions, foreign keys enabled |
| **App Server** | Gunicorn 22.0 | Multi-worker WSGI HTTP server binding to `0.0.0.0:5000` |
| **Frontend** | HTML5, CSS3, Bootstrap 5.3, JS | Responsive enterprise dashboard, real-time KPI cards |
| **Testing** | Pytest 8.2 & pytest-flask | 20 automated test cases covering routes, security headers, models, & constraints |
| **Container** | Docker Engine 29.8 | Lightweight `python:3.12-slim` image, named volume persistence |
| **CI/CD** | Jenkins LTS | Declarative pipeline with SCM polling, healthchecks, & auto-rollback |
| **Cloud** | AWS EC2 (us-east-1) | Ubuntu Server 24.04 LTS on `t3.medium` instance |

---

## 🚀 Key Functional Modules

1. **Executive Dashboard**: Real-time KPI summary (total products, available stock units, low-stock count, total valuation, category breakdown, recent activity).
2. **Product Catalog Management**: Complete CRUD operations with validation rules, custom categorisation, and pricing controls.
3. **Stock Management (In / Out)**:
   - **Stock IN**: Restocks inventory with supplier/order audit notes.
   - **Stock OUT**: Dispatches items with automatic check preventing negative inventory balances.
   - **Transaction Log**: Full audit trail of stock transitions (`previous_stock` &rarr; `new_stock`).
4. **Low-Stock Alerts**: Automatic detection when `quantity <= min_stock`. Highlights depleted items and calculates deficit quantities.
5. **Search & Multidimensional Filtering**: Instant substring search, category dropdown filter, and bidirectional sorting by name, price, stock, and date.
6. **Health Verification Endpoint**: `GET /health` returns JSON `{"status": "healthy", "service": "smart-inventory", "database": "connected"}` with HTTP 200.

---

## 🔒 Production Security & Cleanliness Audit

A comprehensive security and repository cleanliness audit has been conducted to ensure the project meets strict enterprise deployment criteria:

1. **Zero Secret Leaks**:
   - No AWS access keys, secret keys, or session tokens are committed to source control.
   - SSH private keys (`.pem`) are strictly stored outside the repository in `~/.ssh/smart-inventory-key.pem`.
   - Database credentials and session signing secrets are dynamically generated or passed via environment variables.
2. **Watertight `.gitignore` & `.dockerignore`**:
   - Python virtual environments (`.venv`), bytecode (`__pycache__`, `*.pyc`), test caches (`.pytest_cache`), local databases (`instance/*.db`), environment files (`.env`), and OS metadata (`.DS_Store`) are completely excluded from git and Docker builds.
3. **Database Security**:
   - All SQL queries use parameterized queries (`?`) to prevent SQL injection vulnerabilities.
   - Foreign key constraints are enforced (`PRAGMA foreign_keys = ON;`).
4. **HTTP Security Headers**:
   - Production headers (`X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `X-XSS-Protection: 1; mode=block`) are injected on every response.
5. **Isolated Container Execution**:
   - Application runs containerized behind Gunicorn WSGI server with designated non-root volumes.
   - Only the public application URL (`http://54.87.5.184/`) is exposed on standard HTTP port 80.

---

## 💻 Local Development Setup

To run and test the application locally on a development machine:

```bash
# 1. Clone repository
git clone https://github.com/ansu647/smart-inventory-devops.git
cd smart-inventory-devops

# 2. Create Python 3.12 virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 3. Install dependencies
pip install -r requirements-dev.txt

# 4. Run automated unit tests
pytest -v

# 5. Start development server
python app.py
# Application accessible at http://127.0.0.1:5000
```

---

## 🐳 Local Docker Execution

Test the production container locally with persistent storage:

```bash
# Build Docker image
docker build -t smart-inventory:latest .

# Create named volume for SQLite persistence
docker volume create inventory_data

# Run container mapping host port 80 (or 5000)
docker run -d \
  --name smart-inventory-app \
  -p 80:5000 \
  -v inventory_data:/data \
  smart-inventory:latest

# Verify health endpoint
curl -i http://localhost/health
```

---

## 📑 Project Documentation Links

- [Workstation Environment Setup (ENVIRONMENT_SETUP.md)](ENVIRONMENT_SETUP.md)
- [AWS EC2 & Jenkins Pipeline Guide (DEPLOYMENT_GUIDE.md)](DEPLOYMENT_GUIDE.md)
- [Comprehensive Academic Report (PROJECT_REPORT.md)](PROJECT_REPORT.md)

---

## 🌐 Quick Access & Verified Live Links

| Resource | Public Access Link | Protocol / Port | Real-Time Sync Status |
| :--- | :--- | :--- | :--- |
| **🚀 Production Live Application** | **[http://54.87.5.184/](http://54.87.5.184/)** | HTTP : 80 | **Active & Live** |
| **🩺 Healthcheck & Probe Endpoint** | **[http://54.87.5.184/health](http://54.87.5.184/health)** | HTTP : 80 | **HTTP 200 (Healthy)** |
| **⚙️ Jenkins CI/CD Dashboard** | **[http://54.87.5.184:8080/job/Smart-Inventory-CICD/](http://54.87.5.184:8080/job/Smart-Inventory-CICD/)** | HTTP : 8080 | **Active (Private Admin Access)** |
| **🔄 Real-Time GitHub Webhook** | `http://54.87.5.184:8080/github-webhook/` | POST / 8080 | **Connected (Triggers build on push)** |
| **📦 GitHub Source Repository** | **[https://github.com/ansu647/smart-inventory-devops](https://github.com/ansu647/smart-inventory-devops)** | HTTPS / 443 | **Tracked `main` branch** |

---

## 📄 License
This project is open-source under the MIT License. Developed for academic and demonstration purposes.


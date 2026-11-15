# Smart Inventory Management System
> A Flask-based inventory management application deployed through an automated Jenkins CI/CD pipeline on AWS EC2 using Git and Docker.

---

## LIVE APPLICATION

- **Application URL**: [OPEN SMART INVENTORY APPLICATION](http://54.90.251.219/)
- **Status**: Live, Deployed & Verified on AWS EC2
- **Repository**: [View Source Code](https://github.com/ansu647/smart-inventory-devops)
- **CI/CD Automation**: [Jenkins Dashboard](http://54.90.251.219:8080/job/Smart-Inventory-CICD/) *(Admin: `admin`)*

---

## 📌 Executive Summary & Architecture

The **Smart Inventory Management System** is a production-ready, administrative web platform designed to monitor physical inventory, track stock movements, enforce zero-deficit inventory policies, and alert teams to low stock in real time.

Built to modern DevOps standards, the application undergoes automated testing and deployment whenever changes are pushed to GitHub:
```
Developer Commit & Push (MacBook M3)
          │
          ▼
GitHub Remote Repository (`main` branch)
          │  SCM Polling (H/2 * * * *)
          ▼
Jenkins LTS (AWS EC2 Ubuntu 24.04)
          ├── Stage 1: Checkout Source Code
          ├── Stage 2: Isolated Virtualenv & Pytest (19/19 Passing)
          ├── Stage 3: Docker Image Build (`smart-inventory:${BUILD_NUMBER}`)
          ├── Stage 4: Container Deployment (Port 80:5000, Persistent Volume)
          └── Stage 5: Health Check Verification (GET /health -> HTTP 200)
          │
          ▼
AWS EC2 Live Endpoint (http://54.90.251.219/)
```

---

## 🛠️ Technology Stack

| Domain | Technology | Implementation Detail |
| :--- | :--- | :--- |
| **Backend** | Python 3.12+ / Flask 3.0 | Application factory, modular routing, context processors |
| **Database** | SQLite 3 | Embedded, ACID transactions, foreign keys enabled |
| **App Server** | Gunicorn 22.0 | Multi-worker WSGI HTTP server binding to `0.0.0.0:5000` |
| **Frontend** | HTML5, CSS3, Bootstrap 5.3, JS | Responsive enterprise dashboard, real-time KPI cards |
| **Testing** | Pytest 8.2 & pytest-flask | 19 automated test cases covering routes, models, & constraints |
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

## 🎓 Faculty Demonstration Guide

To present this DevOps project to faculty, follow this step-by-step walkthrough:

1. **Live Application Navigation**:
   - Open the live URL: `http://54.90.251.219/` in any external browser.
   - Demonstrate the responsive sidebar, top navigation, and live system status pill.
2. **Review Dashboard Statistics**:
   - Point out real-time aggregated metrics: Total Products, Available Units, Low Stock Alerts, and Inventory Valuation.
3. **Add a Product**:
   - Navigate to **Add Product** (`/products/add`).
   - Enter a sample product (e.g., `MacBook Air M3`, Category: `Electronics`, Price: `$1099`, Quantity: `10`, Min Stock: `3`).
   - Click **Save Product** and demonstrate instant persistence in the catalog table.
4. **Demonstrate Stock Out & Negative Quantity Prevention**:
   - Go to **Stock In / Out** (`/stock`).
   - Select the newly created product. Select **Stock OUT**, input `999` units, and click **Confirm Stock Out**.
   - Show that the system cleanly blocks the transaction with an alert: `Stock cannot be negative`.
   - Dispatch a valid amount (e.g. `8` units). Show that available stock reduces to `2` units.
5. **Show Low Stock Alert Trigger**:
   - Since available stock (`2`) is below the minimum threshold (`3`), navigate to **Low Stock** (`/low-stock`).
   - Show the dynamic warning badge, deficit calculation (`+1 needed`), and the status indicator.
6. **Show GitHub Repository**:
   - Open `https://github.com/ansu647/smart-inventory-devops`.
   - Show clean repository structure, `Dockerfile`, `Jenkinsfile`, and passing test suite.
7. **Trigger Automated CI/CD Pipeline via Git Push**:
   - On the workstation terminal, make a slight visual update (e.g., in a template).
   - Run `git commit -am "chore: trigger CI/CD pipeline update"` and `git push origin main`.
8. **Demonstrate Jenkins Pipeline Execution**:
   - Open Jenkins (`http://54.90.251.219:8080/job/Smart-Inventory-CICD/`).
   - Show Jenkins detecting the push via SCM polling (`H/2 * * * *`).
   - Show all 5 pipeline stages completing with green checkmarks:
     - Stage 1: Checkout Source
     - Stage 2: Pytest Suite Execution (19 passed)
     - Stage 3: Docker Build
     - Stage 4: Container Deployment
     - Stage 5: Health Check Verification (`/health` HTTP 200)
9. **Verify Zero-Downtime Live Update**:
   - Refresh the public URL `http://54.90.251.219/`. Show the newly deployed version.
10. **Demonstrate SQLite Database Persistence**:
    - On the EC2 terminal, run `docker restart smart-inventory-app` (or remove and recreate the container).
    - Refresh the browser: show that all products, transactions, and stock history remain completely intact due to the Docker named volume mount (`inventory_data:/data`).

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

## 📄 License
This project is open-source under the MIT License. Developed for academic and demonstration purposes.


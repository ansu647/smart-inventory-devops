# Academic Project Report
## Smart Inventory Management System with Automated CI/CD Deployment Using AWS Cloud, Git, Jenkins, and Docker

---

## 1. Abstract
In modern software engineering, bridging application development with cloud operations through automated Continuous Integration and Continuous Deployment (CI/CD) pipelines is essential for reliable, rapid, and scalable releases. This project presents the design, development, containerization, and automated cloud deployment of a **Smart Inventory Management System**. The application is developed using Python 3.12, Flask, and an ACID-compliant SQLite relational database with Docker named volume persistence. The DevOps lifecycle is powered by Git for distributed version control, GitHub for repository hosting, Jenkins LTS on an AWS EC2 Ubuntu instance for automated build, testing, and deployment orchestration, and Docker for containerized application runtime. An end-to-end automated pipeline ensures that every code push to GitHub triggers automated testing, container builds, zero-downtime redeployment, and automated health checks, making the live application instantly accessible via a public AWS EC2 endpoint.

---

## 2. Introduction
Traditional inventory tracking frequently suffers from manual data entry errors, lack of real-time stock visibility, and brittle, manual deployment workflows. Concurrently, manual deployments often cause inconsistencies between staging and production environments ("works on my machine" syndrome). 

By uniting lightweight microframework design with industry-standard DevOps automation, this project demonstrates a modern software engineering lifecycle suitable for production-grade operational workflows.

---

## 3. Problem Statement
Organizations handling physical assets require an intuitive administrative interface to track inventory levels, monitor low-stock thresholds, and log stock inflows and dispatches without data corruption. Furthermore, development teams require an automated delivery mechanism so that every bug fix or feature enhancement can be thoroughly tested, built into a reproducible container image, and deployed to cloud infrastructure without human intervention or data loss.

---

## 4. Project Objectives
1. **Application Layer**: Develop a responsive, full-featured administrative web application using Python 3.12, Flask, HTML5, CSS3, Bootstrap 5, and SQLite.
2. **Data Integrity & Persistence**: Design an ACID-compliant schema with foreign key constraints, parameterized queries, and persistent Docker volume mounts to ensure zero data loss during application updates.
3. **Automated CI/CD Pipeline**: Implement an end-to-end automated Jenkins pipeline triggered via GitHub SCM polling.
4. **Automated Verification**: Build an automated Pytest test suite covering route handling, database CRUD operations, business logic constraints, and service health endpoints.
5. **Containerization & Cloud Deployment**: Package the application with Gunicorn into a slim Docker container deployed on AWS EC2, exposing the system over standard HTTP port 80.

---

## 5. Technology Stack & Architectural Decisions

| Layer / Role | Technology Selected | Justification |
| :--- | :--- | :--- |
| **Backend Framework** | Python 3.12 + Flask 3.0.3 | Minimalistic, fast execution, expressive routing, low overhead |
| **Database** | SQLite 3 | Embedded, zero-configuration, ACID compliant, persistent across container restarts via volume mounts |
| **WSGI Server** | Gunicorn 22.0.0 | Production-grade multi-worker WSGI HTTP server |
| **Frontend** | HTML5, CSS3, Bootstrap 5.3, JS | Clean, responsive enterprise UI; fast load times; mobile friendly |
| **Version Control** | Git & GitHub | Distributed version control, branch management, CI/CD trigger source |
| **CI/CD Automation** | Jenkins LTS | Industry-standard automation server for pipeline orchestration |
| **Containerization** | Docker Engine | Immutable artifacts, consistent runtime across M3 and AWS EC2 |
| **Cloud Hosting** | AWS EC2 (Ubuntu 24.04 LTS) | High availability, scalable cloud compute, configurable security groups |
| **Automated Testing** | Pytest 8.2 + pytest-flask | Comprehensive unit, route, and database test coverage |

---

## 6. System Architecture

```
+-----------------------------------------------------------------------------------+
|                            DEVELOPER ENVIRONMENT (M3 MAC)                         |
|  - Python 3.12 Virtual Environment (.venv)                                        |
|  - Local Pytest Execution (20/20 Passing)                                         |
|  - Git Commit & Push                                                              |
+------------------------------------------+----------------------------------------+
                                           | git push origin main
                                           v
+-----------------------------------------------------------------------------------+
|                               GITHUB REPOSITORY                                   |
|  - URL: https://github.com/ansu647/smart-inventory-devops                        |
|  - Source Code, Dockerfile, Jenkinsfile, Automated Tests                          |
+------------------------------------------+----------------------------------------+
                                           | [Webhook POST] /github-webhook/
                                           | [SCM Polling] H/2 * * * *
                                           v
+-----------------------------------------------------------------------------------+
|                             AWS EC2 CLOUD HOST (UBUNTU)                           |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  |                             JENKINS CI/CD SERVER                            |  |
|  |  Stage 1: Checkout Source Code from GitHub                                  |  |
|  |  Stage 2: Setup Test Environment & Run 20 Pytests                           |  |
|  |  Stage 3: Build Docker Image (smart-inventory:${BUILD_NUMBER})              |  |
|  |  Stage 4: Deploy Container with Persistent Volume Mount (`inventory_data`)   |  |
|  |  Stage 5: Health Check Verification (GET /health -> HTTP 200)                |  |
|  +-------------------------------------+---------------------------------------+  |
|                                        | Deploy / Run                             |
|                                        v                                          |
|  +-----------------------------------------------------------------------------+  |
|  |                           DOCKER RUNTIME CONTAINER                          |  |
|  |  - Container Name: smart-inventory-app                                      |  |
|  |  - Port Mapping: 80:5000 (Gunicorn WSGI)                                     |  |
|  |  - Volume Mount: inventory_data -> /data/inventory.db                       |  |
|  +-------------------------------------+---------------------------------------+  |
|                                        |                                          |
+----------------------------------------+------------------------------------------+
                                         | HTTP Port 80 (Public Traffic)
                                         v
+-----------------------------------------------------------------------------------+
|                               END USER / BROWSER                                  |
|  Public Live URL: http://54.87.5.184/                                            |
+-----------------------------------------------------------------------------------+
```

---

## 7. Functional Module Descriptions

### Module 1: Executive Dashboard
- Aggregates high-level metrics in real time from SQLite: total unique SKUs, total units on hand, total valuation, and low-stock alert count.
- Visualizes category breakdowns and recent stock movement events.

### Module 2: Product Catalog Management
- Enables full CRUD operations (Create, Read, Update, Delete) for products.
- Product fields include ID, Name, Category, Unit Price, Stock Quantity, Minimum Alert Threshold, and Created Timestamp.
- Parameterized SQL execution prevents SQL injection vulnerabilities.

### Module 3: Stock Movement Operations (Stock In / Stock Out)
- Supports inventory replenishment (`Stock IN`) and fulfillment dispatch (`Stock OUT`).
- Enforces strict validation: transaction amounts must be positive integers, and stock levels cannot fall below zero.
- Maintains a detailed transaction audit trail in `stock_history`.

### Module 4: Low Stock Alert Engine
- Evaluates stock availability against individual SKU threshold configurations (`quantity <= min_stock`).
- Categorizes inventory status into **Optimal**, **Critical Low**, and **Out of Stock**.
- Provides single-click restock navigation.

### Module 5: Search & Multidimensional Filtering
- Dynamic filtering by search substring across name and category.
- Dropdown category filtration.
- Bidirectional sorting by product name, unit price, stock count, and creation date.

---

## 8. Database Design & Schema

The relational schema is implemented in SQLite with foreign keys and index optimizations:

```sql
-- Products Table
CREATE TABLE products (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    category TEXT NOT NULL,
    price REAL NOT NULL CHECK(price >= 0),
    quantity INTEGER NOT NULL DEFAULT 0 CHECK(quantity >= 0),
    min_stock INTEGER NOT NULL DEFAULT 5 CHECK(min_stock >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Stock Transaction History Table
CREATE TABLE stock_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id INTEGER NOT NULL,
    change_type TEXT NOT NULL CHECK(change_type IN ('IN', 'OUT', 'INITIAL', 'ADJUSTMENT')),
    quantity INTEGER NOT NULL,
    previous_stock INTEGER NOT NULL,
    new_stock INTEGER NOT NULL,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);

-- Indexes for Query Performance
CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_quantity ON products(quantity);
CREATE INDEX idx_stock_history_product_id ON stock_history(product_id);
```

### Data Persistence Architecture
In containerized environments, container storage is ephemeral. To guarantee data preservation across container updates, the production SQLite database is mounted to a Docker named volume:
- Host Volume: `inventory_data`
- Container Mount Point: `/data`
- Database File: `/data/inventory.db`

---

## 9. Testing & Verification

Automated testing is integrated directly into the CI/CD pipeline using **Pytest**. Tests run against isolated, temporary SQLite instances to eliminate side-effects.

### Test Results Summary
- Total Tests: **20**
- Passing: **20 (100%)**
- Execution Duration: **0.24 seconds**

| Test Module | Test Case | Target Feature | Result |
| :--- | :--- | :--- | :--- |
| `test_database.py` | `test_init_db` | Table and index generation | Passed |
| `test_database.py` | `test_create_and_get_product` | Insertion and retrieval | Passed |
| `test_database.py` | `test_product_validation` | Validation constraints | Passed |
| `test_database.py` | `test_update_product` | SKU modification | Passed |
| `test_database.py` | `test_delete_product` | Cascade deletion | Passed |
| `test_database.py` | `test_stock_in` | Inflow quantity adjustment | Passed |
| `test_database.py` | `test_stock_out` | Outflow quantity adjustment | Passed |
| `test_database.py` | `test_negative_quantity_prevention` | Negative stock rollback | Passed |
| `test_database.py` | `test_low_stock_detection` | Threshold alert logic | Passed |
| `test_database.py` | `test_search_and_filter` | Query and sorting logic | Passed |
| `test_app.py` | `test_health_check` | HTTP `/health` response | Passed |
| `test_app.py` | `test_security_headers` | Production HTTP security headers | Passed |
| `test_app.py` | `test_dashboard_route` | UI metrics rendering | Passed |
| `test_app.py` | `test_products_list_and_search` | HTTP query parameters | Passed |
| `test_app.py` | `test_add_product_route` | Form POST processing | Passed |
| `test_app.py` | `test_edit_product_route` | Update POST processing | Passed |
| `test_app.py` | `test_delete_product_route` | Delete POST processing | Passed |
| `test_app.py` | `test_stock_in_and_out_route` | Stock movements via web | Passed |
| `test_app.py` | `test_negative_stock_rejection_route` | Form-level validation flash | Passed |
| `test_app.py` | `test_low_stock_route` | Alert page view | Passed |

---

## 10. Conclusion & Future Scope
The **Smart Inventory Management System** successfully unifies full-stack web application development with modern DevOps automation. By utilizing AWS EC2, Git, Jenkins, and Docker, the project achieves:
- Automated continuous delivery triggered by source code commits.
- Zero manual server configuration during routine deployments.
- Guaranteed database persistence across container lifecycles.
- A publicly accessible, presentation-ready application URL.

### Future Scope
- Integration of role-based authentication (Admin, Manager, Auditor).
- Barcode and QR code scanning via mobile camera interface.
- Automated email / Slack webhook alerts for depleted inventory.
- Real-time stock notifications via WebSockets.


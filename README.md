# 🏗️ Hello World — Three-Tier AWS Architecture

> A production-pattern web application demonstrating clean separation of concerns across three independent tiers: a browser-facing NGINX layer, a PHP business logic layer, and a managed MySQL data layer — all deployed on AWS with load balancing and auto-scaling at every tier.

![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?style=flat-square&logo=amazonaws&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-Backend-777BB4?style=flat-square&logo=php&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-Database-4479A1?style=flat-square&logo=mysql&logoColor=white)
![NGINX](https://img.shields.io/badge/NGINX-Web%20Server-009639?style=flat-square&logo=nginx&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

---

## 📑 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [AWS Infrastructure Components](#-aws-infrastructure-components)
- [Traffic Flow](#-traffic-flow)
- [Directory Structure](#-directory-structure)
- [Local Setup](#-local-setup)
- [Deployment Steps](#-deployment-steps)
- [Development Guide](#-development-guide)
- [Security Considerations](#-security-considerations)

---

## 🧱 Architecture Overview

![AWS Three-Tier Architecture Diagram](aws-three-tier-architecture.png)

The application follows the classic **three-tier architecture**, separating the presentation, application, and data layers for independent scalability and maintainability.

| Tier | Layer | Technology | Hosting |
|------|-------|------------|---------|
| **Tier 1** | Presentation | HTML · CSS · JavaScript | NGINX on EC2 (public subnets) |
| **Tier 2** | Application | PHP · Apache | EC2 (private app subnets) |
| **Tier 3** | Data | MySQL | Amazon RDS (isolated DB subnets) |

### What each tier does

**Frontend (Tier 1)**
Serves static assets via NGINX. All user interactions are handled here; the frontend calls the backend using the browser's Fetch API. Sits behind a public-facing Application Load Balancer.

**Backend API (Tier 2)**
Processes business logic and exposes REST-style PHP endpoints for reading and writing messages. Sits behind an **internal** ALB — it is never directly reachable from the internet.

**Database (Tier 3)**
A managed RDS MySQL instance deployed in isolated private subnets. Only reachable from the App tier on port `3306`. Handles all persistence and data durability.

---

## ☁️ AWS Infrastructure Components

| Component | Type | Purpose |
|-----------|------|---------|
| **Web ALB** | Application Load Balancer (public) | Distributes inbound HTTP traffic across NGINX instances |
| **NGINX Servers** | EC2 + Auto Scaling Group | Serves the frontend; scales horizontally based on demand |
| **App ALB** | Application Load Balancer (internal) | Routes traffic from NGINX to PHP application servers |
| **PHP Servers** | EC2 + Auto Scaling Group | Runs the API layer; private, not internet-facing |
| **RDS MySQL** | Managed Database (Multi-AZ optional) | Persistent data storage in isolated DB subnets |
| **VPC** | Virtual Private Cloud | Isolated network housing all tiers across 3 Availability Zones |
| **Internet Gateway** | IGW | Enables inbound/outbound internet traffic for public subnets |
| **NAT Gateway** | NAT GW | Allows private EC2 instances to reach the internet (e.g. for `yum`/`apt`) without being publicly reachable |

---

## 🔄 Traffic Flow

```
Browser
   │
   ▼
Web ALB  (public, port 80)
   │
   ▼
NGINX EC2  (Auto Scaling Group — Web Private subnets)
   │  Fetch API call to /api/*
   ▼
App ALB  (internal, port 80)
   │
   ▼
PHP EC2  (Auto Scaling Group — App Private subnets)
   │  PDO prepared statements
   ▼
RDS MySQL  (DB Private subnets, port 3306)
```

Each hop is secured by a dedicated Security Group allowing traffic **only from the upstream tier**.

---

## 📁 Directory Structure

```
three-tier-architecture-aws/
│
├── frontend/
│   ├── index.html            # Main HTML entry point
│   └── styles.css            # Global stylesheet
│
├── backend/
│   └── api/
│       ├── get_messages.php  # GET  /api/messages  — retrieve all messages
│       ├── save_message.php  # POST /api/messages  — persist a new message
│       └── db_connection.php # PDO connection helper (reads DB config)
│
├── database/
│   └── database_setup.sql    # Schema definition + initial seed data
│
└── infrastructure/
    ├── frontend_server.md    # NGINX userdata / manual setup notes
    ├── backend_server.md     # Apache + PHP setup notes
    └── nginx_config          # NGINX server block configuration file
```

---

## 💻 Local Setup

### Prerequisites

- A local web server with PHP support — [XAMPP](https://www.apachefriends.org/), [WAMP](https://www.wampserver.com/), or [MAMP](https://www.mamp.info/)
- MySQL (local instance or Docker: `docker run -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 mysql:8`)

### Steps

1. **Clone the repository** into your web server's document root (e.g. `htdocs/` or `www/`):
   ```bash
   git clone https://github.com/<your-org>/three-tier-architecture-aws.git
   ```

2. **Import the database schema** and seed data:
   ```bash
   mysql -u root -p < database/database_setup.sql
   ```

3. **Configure the DB connection** — edit `backend/api/db_connection.php`:
   ```php
   $host = 'localhost';
   $dbname = 'hellodb';       // match your local DB name
   $username = 'root';
   $password = 'yourpassword';
   ```

4. **Update the API base URL** in `frontend/index.html` to point to your local PHP server:
   ```javascript
   const API_BASE = 'http://localhost/three-tier-architecture-aws/backend/api';
   ```

5. **Open in your browser:**
   ```
   http://localhost/three-tier-architecture-aws/frontend/index.html
   ```

---

## 🚀 Deployment Steps

### Phase 1 — Networking Foundation

1. **Create a VPC** (e.g. `10.0.0.0/16`)
2. **Create 12 subnets** across three Availability Zones:
   - `Web-Public-1a/1b/1c` — internet-facing ALB and NGINX
   - `Web-Private-1a/1b/1c` — reserved for private web tier use
   - `App-Private-1a/1b/1c` — PHP application servers
   - `DB-Private-1a/1b/1c` — database tier (fully isolated)
3. **Create route tables** — one per subnet group; associate each with its subnets

### Phase 2 — Internet & NAT Gateways

4. **Create an Internet Gateway (IGW)** and attach it to the VPC
5. **Create a NAT Gateway** in one of the Web Public subnets (requires an Elastic IP)
6. **Update route tables:**
   - Public route tables → `0.0.0.0/0` via **IGW**
   - Private route tables → `0.0.0.0/0` via **NAT Gateway**

### Phase 3 — Security Groups (Least Privilege)

7. Create five Security Groups with tightly scoped ingress rules:

   | Security Group | Inbound Rule | Source |
   |----------------|-------------|--------|
   | `sg-frontend-alb` | HTTP (80) | `0.0.0.0/0` |
   | `sg-frontend-servers` | HTTP (80) | `sg-frontend-alb` |
   | `sg-backend-alb` | HTTP (80) | `sg-frontend-servers` |
   | `sg-backend-servers` | HTTP (80) | `sg-backend-alb` |
   | `sg-database` | MySQL (3306) | `sg-backend-servers` |

### Phase 4 — Database

8. **Create a DB subnet group** named `database-subnet-group` spanning all three `DB-Private` subnets
9. **Launch an RDS MySQL instance** into this subnet group with `sg-database` applied
10. **Run the database setup script** by SSH-ing into a backend server:
    ```bash
    mysql -h <rds-endpoint> -u <user> -p < database/database_setup.sql
    ```

### Phase 5 — Load Balancers

11. **Create the Frontend ALB** (internet-facing) with a target group pointing to port 80 on NGINX instances
12. **Create the Backend ALB** (internal) with a target group pointing to port 80 on PHP instances

### Phase 6 — Custom AMIs

13. **Build the Frontend AMI:**
    ```bash
    sudo yum install -y nginx git
    sudo git clone https://github.com/<your-org>/three-tier-architecture-aws.git /usr/share/nginx/html/app
    # Apply nginx_config, then start and enable nginx
    sudo systemctl enable --now nginx
    ```
    → Stop the instance and create an AMI from it.

14. **Build the Backend AMI:**
    ```bash
    sudo yum install -y php php-mysqlnd httpd git
    sudo git clone https://github.com/<your-org>/three-tier-architecture-aws.git /var/www/html/app
    # Update db_connection.php with RDS endpoint
    sudo systemctl enable --now httpd
    ```
    → Stop the instance and create an AMI from it.

### Phase 7 — Launch Templates & Auto Scaling

15. **Create a Launch Template** for Frontend Servers using the Frontend AMI + `sg-frontend-servers`
16. **Create a Launch Template** for Backend Servers using the Backend AMI + `sg-backend-servers`
17. **Create an Auto Scaling Group** for Frontend, targeting Web Private subnets and the Frontend ALB target group
18. **Create an Auto Scaling Group** for Backend, targeting App Private subnets and the Backend ALB target group

---

## 🛠️ Development Guide

### Frontend Changes

1. Modify HTML/CSS/JS files in the `frontend/` directory
2. The Fetch API base URL is the only environment-specific value — keep it configurable
3. Test locally before deploying
4. On AWS: push to Git and pull latest in userdata, or rebuild the Frontend AMI

### Backend Changes

1. Edit PHP files in `backend/api/` — three endpoints: `get_messages`, `save_message`, `db_connection`
2. PDO with prepared statements is used throughout — **always maintain this pattern** for new queries
3. Test locally against MySQL before deploying
4. On AWS: push to Git and pull latest in userdata, or rebuild the Backend AMI

### Adding a New API Endpoint

1. Create `backend/api/your_endpoint.php`
2. Require `db_connection.php` for database access
3. Return JSON with appropriate `Content-Type: application/json` header
4. Call the endpoint from `frontend/index.html` via Fetch

---

## 🔒 Security Considerations

> ⚠️ This is a **demo application**. The following hardening steps are required before any production exposure.

| Area | Current State | Production Requirement |
|------|--------------|----------------------|
| **Input Validation** | None | Validate and sanitize all user-supplied input server-side |
| **Authentication** | None | Add session-based or token-based (JWT) auth before exposing endpoints |
| **HTTPS / TLS** | HTTP only | Attach an ACM certificate to both ALBs; redirect HTTP → HTTPS |
| **CORS Policy** | Open | Restrict `Access-Control-Allow-Origin` to your frontend domain only |
| **SQL Injection** | PDO prepared statements ✅ | Maintain this pattern; never concatenate user input into SQL |
| **Secrets Management** | Hardcoded in PHP | Move DB credentials to **AWS Secrets Manager** or **Parameter Store** |

---

## 📄 License

Released under the [MIT License](LICENSE).

---

## 🙏 Acknowledgements

This sample application was created as a demonstration of AWS three-tier architecture principles.

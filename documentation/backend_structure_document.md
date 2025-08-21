# Backend Structure Document

This document outlines the backend architecture, database setup, APIs, hosting, infrastructure, security, and monitoring for the **b2b-hunter** platform. It is written in straightforward language so that anyone can understand how the backend is structured and operates.

## 1. Backend Architecture

### Overall Design
- The backend is split into two main services:
  - **Core API**: Built with Node.js and Express, this service handles user authentication, search requests, data exports, dashboard data, and integrations.
  - **Enrichment Microservice**: Built with Python and FastAPI, this service manages web crawling, data enrichment (company profiles, social links), and contact verification (SMTP checks).
- Both services follow a **layered architecture**:
  1. **Routing Layer** handles incoming HTTP requests.
  2. **Controller Layer** contains business logic (e.g., validating filters, triggering enrichment).
  3. **Service Layer** orchestrates calls to databases, caches, and external APIs.
  4. **Data Access Layer** abstracts database queries and Elasticsearch calls.

### Scalability, Maintainability & Performance
- **Containerization**: Each service runs in its own Docker container, ensuring consistent environments.
- **Orchestration**: Kubernetes (on AWS EKS) automatically scales services up or down based on load.
- **Separation of Concerns**: Splitting search/API and enrichment into separate services allows independent scaling and easier code maintenance.
- **Caching**: Redis stores frequent search results and session data to reduce database load and improve response times.
- **Specialized Search**: Elasticsearch powers advanced filtering so most queries return under 2 seconds.

## 2. Database Management

### Technologies Used
- **PostgreSQL (SQL)**: Primary relational store for users, roles, company records, search histories, audit logs, and export metadata.
- **Redis (In-Memory Cache)**: Caches session tokens, recent search results, and rate-limit counters.
- **Elasticsearch (Search Engine)**: Indexes company data for fast, faceted search (industry, size, location, revenue).
- **AWS S3 (Object Storage)**: Stores raw crawl files, large export files (CSV/JSON), and backup snapshots.

### Data Management Practices
- **Schema Migrations**: Handled via a migration tool (e.g., Flyway or TypeORM migrations) to ensure versioned, trackable changes.
- **Backups & Retention**: Automated daily backups of PostgreSQL via AWS RDS snapshots; S3 lifecycle rules purge raw crawl data older than configured thresholds.
- **Index Rebuilds**: Periodic re-indexing of Elasticsearch when bulk data changes.
- **Data Purging**: Automated GDPR/CCPA compliance routines remove user-personal data on request or after retention periods.

## 3. Database Schema

Below is a human-readable overview of our main SQL tables, followed by their definition in PostgreSQL syntax.

### Human-Readable Schema
- **Users**: stores user accounts, emails, hashed passwords, roles, and 2FA settings.
- **Roles**: defines permission levels (e.g., admin, standard user).
- **Companies**: holds basic company info (name, industry, size, location, revenue estimates).
- **EnrichedProfiles**: additional data for each company (headquarters address, founding date, social links).
- **Contacts**: extracted email/phone records tied to companies, with confidence scores.
- **SearchHistory**: records each user’s search queries, filters used, and result counts.
- **Exports**: tracks data exports (type, fields selected, user, timestamp).
- **Integrations**: configuration for CRM webhooks and API credentials.
- **AuditLogs**: captures critical events (logins, exports, role changes) with user, action, timestamp.

### PostgreSQL Schema (Example)
```sql
-- Users and Roles
CREATE TABLE roles (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL  -- e.g., 'admin', 'user'
);

CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role_id INT REFERENCES roles(id),
  is_email_verified BOOLEAN DEFAULT FALSE,
  two_fa_secret VARCHAR(255),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Company and Enrichment
CREATE TABLE companies (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  industry VARCHAR(100),
  size_range VARCHAR(50),
  location JSONB,        -- e.g., { city: 'NYC', region: 'NY', country: 'USA' }
  revenue_range VARCHAR(50),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE enriched_profiles (
  company_id INT PRIMARY KEY REFERENCES companies(id),
  headquarters_address TEXT,
  founding_date DATE,
  employee_trends JSONB,  -- e.g., [ { year: 2021, count: 50 }, ... ]
  social_links JSONB      -- e.g., { linkedin: 'url', twitter: 'url' }
);

-- Contacts and Verification
CREATE TABLE contacts (
  id SERIAL PRIMARY KEY,
  company_id INT REFERENCES companies(id),
  type VARCHAR(10),       -- 'email' or 'phone'
  value VARCHAR(255),
  confidence_score DECIMAL(5,2),
  source VARCHAR(100),    -- e.g., 'web_crawl', 'linkedin'
  verified BOOLEAN DEFAULT FALSE,
  last_checked_at TIMESTAMP WITH TIME ZONE
);

-- User Activity
CREATE TABLE search_history (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  filters JSONB,
  result_count INT,
  executed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE exports (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  format VARCHAR(10),      -- 'csv' or 'json'
  fields_selected JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  file_s3_key VARCHAR(255)
);

CREATE TABLE integrations (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  service VARCHAR(50),     -- e.g., 'salesforce', 'hubspot'
  config JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Audit Logs
CREATE TABLE audit_logs (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  action VARCHAR(100),     -- e.g., 'login', 'export', 'role_change'
  details JSONB,
  logged_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```  

## 4. API Design and Endpoints

We follow a **RESTful** approach with JSON payloads and secure endpoints. All endpoints require HTTPS, and most require a valid JWT in the Authorization header.

### Authentication & Authorization
- **POST /api/auth/signup**: Create a new user (email, password). Sends verification email.
- **POST /api/auth/login**: Validate credentials, return JWT.
- **GET /api/auth/verify?token=...**: Verify email address.
- **POST /api/auth/forgot-password**: Send password reset link.
- **POST /api/auth/reset-password**: Update password.

### Company Search & Profiles
- **GET /api/companies/search**: Accepts filter parameters, returns paginated company IDs and basic info (proxy to Elasticsearch).
- **GET /api/companies/{companyId}**: Retrieve company details, including enriched profile and the latest contact summary.

### Enrichment & Contacts
- **POST /api/companies/{companyId}/enrich**: Trigger or re-run enrichment (fast internal call to the FastAPI service).
- **POST /api/companies/{companyId}/contacts/verify**: Re-verify selected contacts and return updated scores.

### Exports & Integrations
- **POST /api/exports**: Create an export job (format, fields, company IDs). Returns S3 download link when ready.
- **GET /api/exports/{exportId}**: Check status or download the file.
- **GET /api/integrations**: List configured CRM connections.
- **POST /api/integrations/{id}/push**: Push selected company data to external CRM via webhook/API.

### Dashboard & Admin
- **GET /api/dashboard/metrics**: Return summary stats (search volume, enrichment rates, verification success).
- **GET /api/admin/users** (admin only): List all users.
- **PUT /api/admin/users/{userId}/role** (admin only): Change user role.
- **GET /api/admin/audit-logs** (admin only): Retrieve audit events with optional filters.

## 5. Hosting Solutions

- **Cloud Provider**: AWS (Amazon Web Services) is used for all infrastructure.
- **Container Orchestration**: Kubernetes on EKS manages Docker containers for both services, handling auto-scaling, rolling updates, and self-healing.
- **Relational Database**: PostgreSQL hosted on AWS RDS with multi-AZ deployment for high availability and automatic backups.
- **Object Storage**: AWS S3 stores raw crawl files, export artifacts, and backups.

**Benefits**:
- High availability and automated failover with RDS and EKS.
- Cost-effectiveness via right-sized EC2 node groups and on-demand scaling.
- Tight integration between AWS services simplifies networking, IAM, and monitoring.

## 6. Infrastructure Components

- **Load Balancer**: AWS Application Load Balancer (ALB) distributes incoming HTTP traffic across Kubernetes pods.
- **Caching Layer**: Redis cluster (ElastiCache) caches sessions, search results, and API rate limits.
- **Search Engine**: Elasticsearch cluster (self-managed on EC2 or AWS Elasticsearch Service).
- **CDN**: AWS CloudFront serves static assets (certificates, JavaScript bundles) close to users.
- **Message Broker / Queue** (Optional): A small RabbitMQ or Redis Streams queue handles enrichment job dispatch.
- **CI/CD Pipeline**: GitHub Actions builds, tests, and deploys containers to EKS on each merge to main.
- **Secrets Management**: AWS Secrets Manager stores database credentials, API keys, and 2FA secrets.

## 7. Security Measures

- **Authentication & Authorization**:
  - JWT tokens secure stateless API access.
  - Role-Based Access Control (RBAC) enforces user vs. admin permissions.
- **Data Encryption**:
  - HTTPS/TLS for all client-server traffic.
  - AES-256 encryption at rest for RDS, S3, and Redis.
- **OWASP Protections**:
  - Input validation and sanitization on all endpoints.
  - Rate limiting per IP and per user to guard against brute-force and DOS.
  - Secure HTTP headers via Helmet middleware.
- **Audit Logging**:
  - Every critical action (login, data export, role change) is recorded in `audit_logs`.
- **Compliance**:
  - GDPR/CCPA workflows for data subject requests, consent logs, and automated data deletion.

## 8. Monitoring and Maintenance

- **Metrics & Alerting**:
  - Prometheus scrapes Kubernetes, Node.js, and FastAPI metrics.
  - Grafana dashboards visualize CPU, memory, request latency, and error rates.
  - Alertmanager sends email or Slack alerts on high latency, increased error rates, or Pod failures.
- **Logging**:
  - All services log to stdout/stderr, which are collected by Fluentd or Logstash and stored in Elasticsearch (ELK stack).
  - Kibana provides a UI for searching and filtering logs.
- **Health Checks & Auto-Healing**:
  - Kubernetes liveness and readiness probes restart unhealthy containers automatically.
- **Regular Maintenance**:
  - Scheduled RDS maintenance windows for minor version upgrades.
  - Quarterly security reviews and dependency updates.
  - Periodic disaster recovery drills restoring backups to a staging environment.

## 9. Conclusion and Overall Backend Summary

The **b2b-hunter** backend is built for speed, reliability, and scale. By combining a Node.js/Express core API with a Python/FastAPI enrichment microservice, a powerful SQL and search data layer, and a robust AWS-based infrastructure, we ensure:

- **Fast Searches**: Elasticsearch + Redis caching delivers results under 2 seconds.
- **Reliable Enrichment**: Independent microservice scales crawling and verification jobs.
- **Secure Operations**: Encryption, RBAC, audit logs, and compliance workflows protect user data.
- **High Availability**: Kubernetes on EKS, multi-AZ RDS, and automated failover keep the platform online.
- **Easy Maintenance**: CI/CD pipelines, health checks, and monitoring tools keep everything up to date and in good health.

This setup aligns with b2b-hunter’s goals of delivering accurate B2B lead data quickly and securely, empowering sales and marketing teams to focus on outreach rather than data wrangling.
# b2b-hunter Security Guidelines

This document outlines the security principles, controls, and best practices tailored to the b2b-hunter platform. By following these guidelines, development, operations, and QA teams will ensure that b2b-hunter remains robust, resilient, and compliant with data-protection regulations.

---

## 1. Security by Design
- Embed security reviews in every sprint: threat modeling, design reviews, and code audits.  
- Maintain a security backlog alongside feature backlog to track vulnerabilities and risk mitigations.  
- Adopt an iterative approach: integrate automated security checks (SAST, DAST, dependency scans) into CI/CD pipelines.

## 2. Authentication & Access Control
### 2.1 User Authentication
- **Password Policies:** Enforce minimum 12-character passwords with uppercase, lowercase, digits, and special characters.  
- **Hashing & Salting:** Use bcrypt or Argon2 with unique salts for each user record.  
- **Email Verification:** Require email confirmation before granting access to protected routes.

### 2.2 Session Management & JWT
- **Secure Tokens:** Sign JWTs with a strong secret or asymmetric keys (RS256).  
- **Expiration & Rotation:** Set short `exp` (e.g., 15 min) and refresh tokens with longer lifetimes (e.g., 7 days).  
- **Revocation:** Maintain a token blacklist in Redis for immediate logout and session invalidation.

### 2.3 Role-Based Access Control (RBAC)
- Define roles (`user`, `admin`) in the database.  
- Enforce server-side permission checks on every API endpoint.  
- Use middleware (Node/Express & FastAPI) to verify roles before granting access.

### 2.4 Multi-Factor Authentication (MFA)
- Offer TOTP via authenticator apps.  
- Store shared secrets encrypted in the database.  
- Enforce MFA for admin users and sensitive operations (e.g., changing subscription plans).

## 3. Input Validation & Output Encoding
- **Server-Side Validation:** Validate all JSON payloads with schemas (Joi for Node.js, Pydantic for FastAPI).  
- **Query Parameters:** Sanitize and type-cast search filters to prevent injection (e.g., numeric ranges, allowed string patterns).  
- **NoSQL/SQL Injection:** Use parameterized queries (pg-p (node-pg) for PostgreSQL) and Elasticsearch query DSL bindings.
- **Output Encoding:** Escape data rendered on Next.js pages. Use React’s built-in escaping and sanitize any HTML (e.g., user-provided notes).

## 4. Data Protection & Privacy
### 4.1 Encryption
- **In Transit:** Enforce HTTPS with TLS 1.2+ and HSTS headers (`Strict-Transport-Security`).  
- **At Rest:** Enable AES-256 encryption for RDS/PostgreSQL storage and S3 buckets (bucket-level encryption).

### 4.2 Secrets Management
- Store API keys and database credentials in AWS Secrets Manager or Parameter Store, never in plain environment variables.  
- Automatically rotate secrets and apply IAM policies with least privilege.

### 4.3 Privacy Compliance (GDPR/CCPA)
- Log user consent with timestamps.  
- Provide data-export and deletion endpoints (`/api/v1/users/:id/export`, `/api/v1/users/:id/delete`) that purge PII.  
- Mask or hash sensitive fields in logs (e.g., email, payment tokens).

## 5. API & Service Security
- **HTTPS Enforcement:** Redirect HTTP to HTTPS at load balancer (ALB) level.  
- **Rate Limiting & Throttling:** Use API Gateway or Express middleware (e.g., `express-rate-limit`) to cap requests per IP or user.  
- **CORS Policy:** Allow only approved origins (e.g., `https://app.b2b-hunter.com`).  
- **Versioning:** Prefix routes with `/api/v1/` to manage breaking changes.
- **Error Handling:** Return generic error messages (e.g., `400 Bad Request`) without stack traces. Log details internally.

## 6. Web Application Security Hygiene
- **CSRF Protection:** Use anti-CSRF tokens for all state-changing forms and API endpoints (e.g., `csrf` middleware in Express).  
- **Security Headers:** Deploy Helmet in Express and Next.js:
  - `Content-Security-Policy` (restrict scripts, frames).
  - `X-Content-Type-Options: nosniff`.
  - `X-Frame-Options: DENY`.
  - `Referrer-Policy: strict-origin-when-cross-origin`.
  - `Strict-Transport-Security`.
- **Secure Cookies:** Set `HttpOnly`, `Secure`, and `SameSite=Strict` on session or refresh tokens.

## 7. Secure File Uploads
- Validate file type, size, and content (e.g., whitelist on S3 uploads).  
- Scan uploads for malware via AWS Lambda + antivirus service.  
- Store files with randomized keys outside the public webroot.

## 8. Infrastructure & Configuration Management
- **Hardened AMIs & Containers:** Use minimal OS images and scan Docker images for vulnerabilities (Trivy).  
- **Network Segmentation:** Place databases and caches in private subnets, only accessible from application tiers.  
- **IAM Policies:** Grant ECS/EKS service roles only the needed permissions (least privilege).  
- **Logging & Monitoring:** Centralize logs in ELK, monitor metrics with Prometheus and alarms in CloudWatch for anomalous activity.
- **Disaster Recovery:** Automated backups for RDS (daily), S3 versioning, and cross-region replication.

## 9. Dependency Management
- Use `package-lock.json` and `Pipfile.lock` to pin dependencies.  
- Run periodic automated scans (Dependabot, Snyk) for known CVEs.  
- Audit new libraries manually before adoption; prefer well-maintained, popular packages.

## 10. CI/CD & DevOps Security
- **GitHub Actions Hardening:** Require code reviews, branch protection rules, and 2FA for repo access.  
- **Secrets in CI:** Store tokens in GitHub Secrets; never echo them in logs.  
- **Scan & Test:** On every PR, run linting, unit/integration tests, vulnerability scans, and container image checks.
- **Deployment Approvals:** Enforce manual approval gates before deploying to production.

---

Adhering to these guidelines ensures that b2b-hunter protects user data, maintains regulatory compliance, and resists common attack vectors. Security is everyone’s responsibility—please raise any concerns or deviations for immediate review.
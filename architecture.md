# Yappy WordPress Site Architecture

**Last Updated:** 2025-06-09  
**Author:** Derrin Chong

---

## 1. Overview

This document outlines the architecture of the Yappy WordPress site, hosted on AWS Lightsail. The site is designed for high availability and performance with CDN caching, HTTPS support across multiple domains, and ease of administrative maintenance through the Lightsail console.

---

## 2. Objectives

- Host a single-instance WordPress site.
- Serve identical content across multiple domains:
  - Primary: `yappygroup.com`
  - Secondary: `yappy.com.au`
  - Internal: `yappy.holomuatech.com`
- Enforce HTTPS with a single multi-SAN certificate.
- Use Lightsail Distribution (CDN) for content caching.
- Minimize administrative overhead through UI-based configuration, not Terraform.

---

## 3. Architecture Diagram

```text
+----------------------------+
|        Web Clients         |
| (yappygroup.com, etc.)     |
+-------------+--------------+
              |
              v
   +-----------------------+
   |   DNS (Route 53 or    |
   |   external registrar) |
   +----------+------------+
              |
              v
   +---------------------------+
   |  Lightsail Distribution   |
   |  (CDN + HTTPS w/ cert)    |
   +----------+----------------+
              |
              v
   +---------------------------+
   |   Lightsail WordPress     |
   |  (Bitnami single-site VM) |
   +---------------------------+
```

---

## 4. Components

### 4.1 AWS Lightsail WordPress Instance

- **Type:** Bitnami WordPress (Single Site)
- **OS:** Linux (Debian-based Bitnami image)
- **Services:**
  - Apache with PHP
  - MySQL/MariaDB (local)
- **Region:** (Your selected AWS Region)

### 4.2 Lightsail Distribution

- **Purpose:** Acts as a CDN layer in front of the WordPress instance.
- **Origin:** Public IP of the Lightsail instance.
- **Caching Behavior:** Default caching rules with WordPress-specific tuning possible.
- **Domains:**
  - Attached multi-SAN SSL certificate includes all three domains.
- **Rewrite Behavior:**
  - The origin (WordPress) uses `yappy.holomuatech.com` as `SITEURL`, causing browser redirection to that domain unless overridden.

### 4.3 DNS Configuration

- **DNS Records:**
  - `CNAME` records for all domains point to the Lightsail Distribution endpoint (`*.cloudfront.net`)
  - Alternatively, `A` records if using alias routing with Route 53.
- **Domain Ownership:** Managed via AWS Route 53 or external DNS provider.

### 4.4 SSL/TLS

- **Certificate Type:** Lightsail-issued certificate with multiple SANs:
  - `yappygroup.com`
  - `yappy.com.au`
  - `yappy.holomuatech.com`
- **Attached To:** Lightsail Distribution
- **Validation:** Handled via DNS validation at the time of certificate creation.

---

## 5. WordPress Configuration

### 5.1 Site URLs

- **WordPress Address (URL):** `https://yappy.holomuatech.com`
- **Site Address (URL):** `https://yappy.holomuatech.com`
- These fields are immutable in the UI due to Bitnami settings; overridden via `wp-config.php`.

### 5.2 Domain Aliases

- Secondary domains redirect or resolve to `yappy.holomuatech.com` due to the primary `SITEURL` setting.
- WordPress does not natively support multiple domains without a plugin; redirection is performed by browser due to canonical URL logic.

### 5.3 Plugins

- **Import Plugin:** All-in-One WP Migration (used for `.wpress` file imports)
- Other plugins restored as part of the `.wpress` archive.

---

## 6. Security Considerations

- HTTPS enforced across all domains.
- Admin access protected via Bitnami default credentials (reset during provisioning).
- SSH access to Lightsail instance is restricted to known IPs or via Lightsail Console.

---

## 7. Administrative Notes

- **Backups:** Configure Lightsail automatic snapshots.
- **Scaling:** Lightsail instance is single-node. Vertical scaling is available but horizontal scaling is limited unless migrating to EC2 + ALB.
- **Maintenance:** Prefer UI over CLI or Terraform for simplicity. Certs, domains, and distributions are manageable in the Lightsail Console.

---

## 8. Future Improvements

- Add canonical meta tags to prevent SEO penalties for duplicate domains.
- Consider a reverse proxy in front (e.g., CloudFront + Lambda@Edge) to enforce consistent domain usage.
- Evaluate caching plugin behavior for better CDN integration.

---

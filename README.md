# .agents-whmcs - WHMCS DevKit Agent

**Purpose:** Autonomous agent configuration for WHMCS module development.

**Owner:** Pho Tue SoftWare And Technology Solutions Joint Stock Company (Tax Code: 0318222903)
**Version:** 1.0 | **Updated:** 2026-05-28

**Legal Entity:** CÔNG TY CỔ PHẦN GIẢI PHÁP CÔNG NGHỆ VÀ PHẦN MỀM PHỔ TUỆ  
**D-U-N-S Number:** 557339920  
**Address:** 128 Binh My Street, Binh My Commune, Ho Chi Minh City

**License:** Conditional non-commercial license. Commercial use, resale, SaaS/service use, redistribution, relicensing, marketplace publication, and business exploitation are prohibited without prior written permission. See `LICENSE.md`.

**Trademark Notice:** WHMCS and related WHMCS names, marks, logos, and brand identifiers belong to their respective owner(s), including WHMCS Limited or applicable rights holder(s). This project is independent and is not affiliated with, endorsed by, sponsored by, certified by, or officially connected with WHMCS unless separately agreed in writing. All third-party trademarks belong to their respective owners.

**Legal Documents:** See `NOTICE`, `LICENSE.md`, `COMMERCIAL-LICENSE.md`, `SUPPORT.md`, and `CODE_OF_CONDUCT.md`.

---

## Agent Overview

This is an independent agent system dedicated exclusively to WHMCS module development. The agent focuses ONLY on reading, coding, and supplementing skills/workflows/devkits - no references to external core or sample directories.

---

## Directory Structure

```
.agents-whmcs/
├── CLAUDE.md                    ← Main technical reference
├── README.md                   ← This file
├── devkits/                     ← Complete module templates (60)
├── docs/                       ← Reference documentation (84)
├── workflows/                  ← Step-by-step development guides (84)
└── skills/                     ← Specialized development skills (148)
```

---

## Quick Reference

| Category | Count | Description |
|----------|-------|-------------|
| DevKits | 60 | Complete module templates |
| Skills | 148 | Specialized development guides |
| Workflows | 84 | Step-by-step processes |
| Docs | 84 | Reference documentation |
| **Total** | **376** | WHMCS development resources |

---

## Quick Start

### Step 1: Identify Module Type

| Type | Path | Description |
|------|------|-------------|
| Provisioning | `devkits/provisioning-module/` | VPS, Cloud, Dedicated server |
| Gateway | `devkits/gateway-module/` | Payment collection |
| Registrar | `devkits/registrar-module/` | Domain registration/management |
| Addon | `devkits/addon-module/` | Admin tools, client area pages |
| Notification | `devkits/notification-module/` | SMS, Push, Chat notifications |

### Step 2: Read Technical Reference

**Always read CLAUDE.md first** for:
- WHMCS coding standards
- Module function signatures
- Return value patterns
- Security requirements
- Database patterns (Capsule)

### Step 3: Reference Relevant Skill

Skills provide deep-dive patterns, templates, and checklists for specific development tasks.

---

## Development Flow

```
1. Identify Module Type
       ↓
2. Read CLAUDE.md (mandatory)
       ↓
3. Read corresponding devkit
       ↓
4. Reference relevant workflow/guide
       ↓
5. Check specific skill for patterns
       ↓
6. Build module code
       ↓
7. Validate and test
```

---

## Available DevKits (70)

### Core Modules
1. **Provisioning Module** - Server/VPS/cloud provisioning
2. **Gateway Module** - Payment gateway (5 types)
3. **Registrar Module** - Domain registrar
4. **Addon Module** - Admin/client tools
5. **Notification Module** - Notifications provider

### Advanced Modules
6. **Hooks Module** - Custom hook development
7. **Widget Module** - Dashboard widgets
8. **Reporting Module** - Reports generation
9. **SMTP Module** - Email delivery
10. **Import Module** - Data import tools
11. **Export Module** - Data export
12. **Sync Module** - Synchronization
13. **SSO Module** - Single sign-on
14. **API Module** - REST API endpoints
15. **CDN Module** - CDN integration
16. **Live Chat Module** - Chat integration
17. **Knowledge Base Module** - KB system
18. **SMS Module** - SMS gateway
19. **Email Module** - Custom email
20. **Fraud Module** - Fraud detection
21. **Affiliate Module** - Affiliate tracking
22. **Support Module** - Helpdesk system
23. **Reviews Module** - Product reviews
24. **Backup Module** - Backup system
25. **Subscription Management** - Subscription billing cycles
26. **Customer Portal** - Custom client dashboard
27. **Order Form Builder** - Customizable order forms
28. **Product Configurator** - Product configuration wizard
29. **Domain Checker** - Domain availability checker
30. **Automation Cron** - Cron job automation
31. **Webhook Handler** - Webhook integration
32. **Two-Factor Auth** - Two-factor authentication
33. **IP Whitelist** - Admin IP whitelist
34. **Rate Limiter** - API rate limiting
35. **Email Automation** - Email automation sequences
36. **Ticket Automation** - Support ticket automation
37. **Client Migration** - Client data migration
38. **Invoice Reminder** - Invoice reminder scheduler
39. **Service Upgrade** - Service upgrade/downgrade
40. **Usage Billing** - Usage-based billing tracker
41. **Multi-Tenant** - Multi-tenant architecture
42. **Affiliate Tracking** - Advanced affiliate tracking
43. **Marketplace** - Digital marketplace
44. **API Key Manager** - API key management

---

## Available Skills (128)

### Core Development
- `whmcs-core-reader` - WHMCS module patterns reading
- `whmcs-validator` - Module validation
- `whmcs-server-builder` - Provisioning modules
- `whmcs-gateway-builder` - Payment gateways
- `whmcs-registrar-builder` - Domain registrars
- `whmcs-addon-builder` - Addon modules
- `whmcs-notification-builder` - Notifications

### Development Tools
- `whmcs-api-integration` - API integration patterns
- `whmcs-api-documentation` - API documentation
- `whmcs-hooks-development` - Hook system
- `whmcs-ajax-patterns` - AJAX in WHMCS

### Client & Admin UI
- `whmcs-clientarea-builder` - Client area pages
- `whmcs-admin-ui-builder` - Admin interfaces
- `whmcs-template-styling` - Smarty templates
- `whmcs-widget-builder` - Dashboard widgets

### Data & Storage
- `whmcs-database-design` - Table design
- `whmcs-migration-guide` - Database migrations
- `whmcs-configuration-management` - Settings management

### Operations
- `whmcs-cron-automation` - Cron jobs
- `whmcs-logging` - Logging patterns
- `whmcs-error-handling` - Error handling
- `whmcs-backup-restore` - Backup/restore

### Security & Quality
- `whmcs-security-hardening` - Security hardening
- `whmcs-testing-qa` - Testing patterns
- `whmcs-performance-optimization` - Performance tips

### Integrations
- `whmcs-vietnamese-payment-builder` - VNPay, MoMo, ZaloPay
- `whmcs-ssl-certificate-module` - SSL provisioning
- `whmcs-domain-sync` - Domain synchronization
- `whmcs-einvoice-integration` - E-invoice integration
- `whmcs-webhook-handler` - Webhooks

### Business Features
- `whmcs-subscription-billing` - Subscription billing
- `whmcs-affiliate-module` - Affiliate tracking
- `whmcs-reporting` - Reporting/analytics
- `whmcs-fraud-detection` - Fraud detection

### Support & Communication
- `whmcs-sms-notification-builder` - SMS notifications
- `whmcs-email-template-builder` - Email templates
- `whmcs-support-ticket-module` - Ticket management

### Specialized Modules
- `whmcs-license-module-builder` - License provisioning
- `whmcs-cloudflare-module` - Cloudflare DNS
- `whmcs-reseller-module` - Reseller hosting
- `whmcs-inventory-manager` - Inventory management
- `whmcs-product-configurator` - Product builder
- `whmcs-remote-import` - Data import
- `whmcs-service-billing` - Billing management
- `whmcs-cost-calculator` - Cost calculators

---

## Keywords

Use these keywords when working with this agent:

- "Tạo module WHMCS" - Create WHMCS module
- "Bổ sung chức năng" - Add functionality  
- "Module mới" - New module
- "Debug module" - Debug module
- "Validate module" - Validate module

---

## Agent Rules

1. **ONLY reference files within .agents-whmcs/**
2. **Do NOT mention external directories** (Core_exapm_whmcs/, module_done_whmcs/, etc.)
3. **Focus on skills, workflows, docs, and devkits**
4. **Always read CLAUDE.md before coding**
5. **Return error string for server modules, array for registrar, exception for notification**

---

## Contact

**Legal Entity:** CÔNG TY CỔ PHẦN GIẢI PHÁP CÔNG NGHỆ VÀ PHẦN MỀM PHỔ TUỆ  
**English Name:** Pho Tue SoftWare And Technology Solutions Joint Stock Company  
**Tax Code:** 0318222903  
**D-U-N-S Number:** 557339920  
**Address:** 128 Binh My Street, Binh My Commune, Ho Chi Minh City

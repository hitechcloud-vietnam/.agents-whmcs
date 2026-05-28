# .agents-whmcs - WHMCS DevKit Agent

**Purpose:** Autonomous agent configuration for WHMCS module development.

**Owner:** Pho Tue SoftWare And Technology Solutions JSC (MST: 0318222203)
**Version:** 1.0 | **Updated:** 2026-05-28

---

## Agent Overview

This is an independent agent system dedicated exclusively to WHMCS module development. The agent focuses ONLY on reading, coding, and supplementing skills/workflows/devkits - no references to external core or sample directories.

---

## Directory Structure

```
.agents-whmcs/
├── CLAUDE.md                    ← Main technical reference
├── README.md                   ← This file
├── devkits/                     ← Complete module templates
│   ├── provisioning-module/    ← Server module template
│   ├── gateway-module/         ← Payment gateway template
│   ├── registrar-module/       ← Domain registrar template
│   ├── addon-module/           ← Addon module template
│   └── notification-module/    ← Notification provider template
├── docs/                       ← Reference documentation
│   └── module-groups.md        ← Module type overview
├── workflows/                  ← Step-by-step development guides
│   ├── server-provisioning-module.md
│   ├── payment-gateway-module.md
│   ├── registrar-module.md
│   ├── addon-module.md
│   ├── notification-provider.md
│   ├── whmcs-docker-deployment.md
│   ├── whmcs-marketplace-submission.md
│   └── whmcs-provisioning-automation.md
└── skills/                     ← 56 specialized development skills
    ├── whmcs-server-builder/
    ├── whmcs-gateway-builder/
    ├── whmcs-registrar-builder/
    ├── whmcs-addon-builder/
    ├── whmcs-notification-builder/
    └── ... (51 more skills)
```

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

## Available DevKits (5)

### 1. Provisioning Module
Complete server module template with:
- All lifecycle functions (Create, Suspend, Terminate, etc.)
- API Client skeleton
- Client area template
- Checklist

### 2. Gateway Module
Payment gateway templates (5 types):
- Standard redirect
- Merchant capture
- Tokenization
- Remote input (iframe)
- Callback handler

### 3. Registrar Module
Domain registrar template with:
- All domain functions
- EPP code handling
- DNSSEC management
- DNS management
- Sync functionality

### 4. Addon Module
Addon module template with:
- Database setup (activate/deactivate)
- Upgrade/migration patterns
- Admin output with CSRF
- Client area template
- Sidebar hooks

### 5. Notification Module
Notification provider template with:
- Provider class with DescriptionTrait
- API Client skeleton
- Test connection pattern
- Send notification logic
- whmcs.json metadata

---

## Available Skills (56)

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

**Owner:** Pho Tue SoftWare And Technology Solutions JSC (MST: 0318222203)

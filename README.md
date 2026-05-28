# .agents-whmcs - WHMCS Agent Configuration

**Purpose:** WHMCS module development configurations and guides for HiTechCloud DevKits.

**Owner:** Pho Tue SoftWare And Technology Solutions JSC (MST: 0318222203)

---

## Quick Start

### Read First
1. [CLAUDE.md](CLAUDE.md) - Technical Reference (mandatory)
2. [docs/module-groups.md](docs/module-groups.md) - Module type overview

### Choose Your Module Type

| Type | Workflow | When to Use |
|------|----------|-------------|
| Server/Provisioning | [workflows/server-provisioning-module.md](workflows/server-provisioning-module.md) | VPS, Cloud, Dedicated hosting |
| Payment Gateway | [workflows/payment-gateway-module.md](workflows/payment-gateway-module.md) | Payment collection |
| Domain Registrar | [workflows/registrar-module.md](workflows/registrar-module.md) | Domain registration/transfer |
| Addon Module | [workflows/addon-module.md](workflows/addon-module.md) | Admin tools, client area |
| Notification | [workflows/notification-provider.md](workflows/notification-provider.md) | SMS, Push notifications |

### Available Skills

| Skill | Purpose |
|-------|---------|
| [skills/whmcs-core-reader/](skills/whmcs-core-reader/SKILL.md) | Reading WHMCS sample modules |
| [skills/whmcs-validator/](skills/whmcs-validator/SKILL.md) | Validating WHMCS modules |

---

## Directory Structure

```
.agents-whmcs/
├── CLAUDE.md                           ← Main technical reference
├── docs/
│   └── module-groups.md               ← Module type overview
├── workflows/
│   ├── server-provisioning-module.md  ← Provisioning module guide
│   ├── payment-gateway-module.md       ← Payment gateway guide
│   ├── registrar-module.md             ← Domain registrar guide
│   ├── addon-module.md                 ← Addon module guide
│   └── notification-provider.md        ← Notification provider guide
├── skills/
│   ├── whmcs-core-reader/             ← How to read WHMCS core
│   └── whmcs-validator/               ← Module validation guide
└── README.md                          ← This file
```

---

## Module Development Flow

```
1. Identify Module Type
   ↓
2. Read CLAUDE.md
   ↓
3. Read corresponding workflow
   ↓
4. Read sample modules in Core_exapm_whmcs/
   ↓
5. Create module in module_dev_whmcs/
   ↓
6. Validate with whmcs-validator skill
   ↓
7. Test in WHMCS installation
```

---

## Reference Locations

### Read Only (Do not modify)
```
Core_exapm_whmcs/                 ← WHMCS official samples
module_done_whmcs/                ← Completed WHMCS modules
```

### Development Output
```
module_dev_whmcs/                 ← Write WHMCS modules here
```

---

## Module Types Summary

### Server/Provisioning (modules/servers/)
Automated hosting service provisioning
- Create, Suspend, Unsuspend, Terminate
- Password change, Package change

### Payment Gateway (modules/gateways/)
Payment collection
- Standard redirect
- Merchant capture
- Tokenization
- Remote input (iframe)
- Remote bank

### Registrar (modules/registrars/)
Domain management
- Register, Transfer, Renew
- Nameservers, Contacts
- Lock, EPP codes

### Addon (modules/addons/)
Admin tools and client pages
- Database tables (mod_ prefix)
- Admin output with CSRF
- Client area

### Notification (modules/notifications/)
SMS/Push notifications
- use DescriptionTrait (REQUIRED)
- throw Exception on errors

---

## Key WHMCS Conventions

### Naming
- File: `{module}.php` or `{module}/{module}.php`
- Functions: `{module}_FunctionName()`
- Tables: `mod_{module}_table_name`

### Return Values
- Server: `'success'` or error string
- Registrar: `['success' => true]` or `['error' => '...']`
- Notification: throw Exception (NOT return false)

### Security
- CSRF: `check_token()` on all POST
- Tables: Prefix with `mod_`
- Input: Sanitize all user data

---

## Common Issues

| Issue | Solution |
|-------|----------|
| Module not showing | Check file naming, use DescriptionTrait for notification |
| Activation fails | Verify table names have `mod_` prefix |
| Function not called | Check function naming matches file name |
| Callback fails | Verify signature validation, IP whitelist |

---

**Version:** 1.0 | **Updated:** 2026-05-28
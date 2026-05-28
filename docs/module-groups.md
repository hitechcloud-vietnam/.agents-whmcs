# WHMCS Module Groups Reference
# Version: 1.0 | Created: 2026-05-28

## Overview

This document categorizes WHMCS modules into groups based on functionality and provides quick reference for development.

---

## 1. Server/Provisioning Modules (modules/servers/)

**Purpose:** Automate provisioning of hosting services (VPS, cloud, dedicated servers)

**Example Modules in module_done_whmcs/:**
- `cloudzone/` - Cloud hosting provisioning
- `virtualizor/` - Virtualizor VPS management
- `virtualizor_cloud/` - Cloud VPS provisioning
- `virtualizor_cloud_account/` - Cloud account management

**Key Functions:**
- `CreateAccount` - Provision new service
- `SuspendAccount` - Suspend service
- `UnsuspendAccount` - Reactivate service
- `TerminateAccount` - Delete service
- `ChangePassword` - Update password
- `ChangePackage` - Upgrade/downgrade

---

## 2. Payment Gateways (modules/gateways/)

### 2.1 Standard Gateway
**Purpose:** Redirect to external payment page

**Example:** `alepay/` - AlePay redirect gateway

**Key Functions:**
- `config` - Gateway settings
- `link` - Return HTML form to redirect
- `refund` - Process refunds (optional)

### 2.2 Merchant Gateway
**Purpose:** Capture card on-site, then redirect or process

**Example:** `sample-merchant-gateway/`

**Key Functions:**
- `capture_form` - Return card capture form
- `capture` - Process payment
- `refund` - Process refunds

### 2.3 Tokenization Gateway
**Purpose:** Tokenize card for future charges

**Example:** `sample-tokenisation-gateway-module/`

**Key Functions:**
- `capture_form` - Get card token
- `capture_token` - Store token
- `capture` - Charge using token

### 2.4 Remote Input Gateway
**Purpose:** iframe hosted form (PCI compliant)

**Example:** `sample-remote-input-gateway/`

**Key Functions:**
- `remote_input` - Return iframe parameters

### 2.5 Remote Bank Gateway
**Purpose:** Bank simulation gateway

**Example:** `sample-remote-bank-gateway-module/`, `vnpay_v2/`, `payos_v2/`, `sepayv2/`

**Key Functions:**
- `remote_bank` - Bank configuration
- Callback handler with IPN

---

## 3. Registrars (modules/registrars/)

**Purpose:** Domain registration, transfer, management

**Example Modules:**
- `inet/` - INET registrar

**Key Functions:**
- `RegisterDomain` - New registration
- `TransferDomain` - Transfer request
- `RenewDomain` - Renewal
- `GetNameservers` / `SaveNameservers`
- `GetContactDetails` / `SaveContactDetails`
- `GetRegistrarLock` / `SaveRegistrarLock`
- `GetEPPCodes` - Auth code for transfer
- `Sync` - Sync domain status

---

## 4. Addon Modules (modules/addons/)

**Purpose:** Admin tools, client area pages, automation

**Example Modules in module_dev_whmcs/:**
- `ddns_admin_whitelist/` - Admin whitelist tool
- `misa_invoice/` - Misa e-invoice integration
- `viettel_sinvoice/` - Viettel S-Invoice integration

**Key Functions:**
- `config` - Module configuration
- `activate` - Create tables (mod_ prefix)
- `deactivate` - Drop tables
- `upgrade` - Version migrations
- `output` - Admin area output
- `clientarea` - Client-facing page

---

## 5. Notification Providers (modules/notifications/)

**Purpose:** Send SMS, push notifications, chat messages

**Example Modules:**
- `ZaloZbs/` - Zalo notification (in dev)

**Key Functions:**
- `moduleConfiguration` - Module settings
- `testConnection` - Verify credentials (throw Exception)
- `notificationSettings` - Per-rule settings
- `send` - Send notification (throw Exception)

---

## 6. Hooks (hooks.php)

**Purpose:** React to WHMCS events

**Common Hooks:**
- `ClientAdd` - New client registration
- `AfterModuleCreate` - Service created
- `AfterModuleSuspend` - Service suspended
- `InvoicePaid` - Payment received
- `TicketOpen` - New support ticket
- `DailyCronJob` - Daily maintenance

---

## Module Type Quick Reference

| Type | Location | Return Type | Key Pattern |
|------|----------|-------------|-------------|
| Server | `modules/servers/{name}/{name}.php` | String `'success'` or error | Provisioning lifecycle |
| Gateway | `modules/gateways/{name}.php` | String (HTML) or Array | Payment flow |
| Registrar | `modules/registrars/{name}/{name}.php` | Array `['success' => bool]` | Domain operations |
| Addon | `modules/addons/{name}/{name}.php` | Array or void | Admin/Client UI |
| Notification | `modules/notifications/{Name}/{Name}.php` | void (throw Exception) | Notification delivery |

---

## Development Order by Type

### Server/Provisioning
1. Read `sample-provisioning-module/`
2. Create `lib/ApiClient.php`
3. Implement all lifecycle functions
4. Add `MetaData` and `ConfigOptions`
5. Test connection function

### Payment Gateway
1. Determine gateway type (standard/merchant/token/remote)
2. Read corresponding sample
3. Implement `config` and `link`
4. Create callback handler with signature validation
5. Add refund support if needed

### Registrar
1. Get EPP or REST API documentation
2. Read `sample-registrar-module/`
3. Implement all domain functions
4. Add `getConfigArray`
5. Test with domain registration

### Addon
1. Read `sample-addon-module/`
2. Plan database tables (prefix with `mod_`)
3. Implement `config`, `activate`, `deactivate`
4. Add `output` with CSRF protection
5. Add client area if needed

### Notification
1. Read `sample-notification-module/`
2. Implement `use DescriptionTrait` (REQUIRED)
3. Add `moduleConfiguration`
4. Implement `testConnection` and `send` (throw exceptions)
5. Create logo (80x80px)

---

Last updated: 2026-05-28
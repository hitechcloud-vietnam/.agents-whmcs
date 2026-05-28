# WHMCS Hooks Reference
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Complete list of WHMCS hooks with examples.

## Client Hooks

| Hook | Parameters | Description |
|------|-------------|-------------|
| `ClientAdd` | `userid`, `firstname`, `lastname`, `email` | New client registered |
| `ClientEdit` | `userid`, `params` | Client profile updated |
| `ClientDelete` | `userid` | Client deleted |
| `ClientLogin` | `userid` | Client logged in |
| `PreLoginClient` | `username` | Before login validation |
| `ClientAreaFooterOutput` | - | Footer HTML output |

## Invoice Hooks

| Hook | Parameters | Description |
|------|-------------|-------------|
| `InvoiceCreation` | `invoiceid`, `userid` | Invoice created |
| `InvoicePaid` | `invoiceid`, `userid`, `amount` | Invoice payment received |
| `InvoiceCancelled` | `invoiceid` | Invoice cancelled |
| `InvoiceRefunded` | `invoiceid` | Invoice refunded |
| `InvoicePrePayment` | `invoiceid` | Before payment processing |

## Service Hooks

| Hook | Parameters | Description |
|------|-------------|-------------|
| `AfterModuleCreate` | `serviceid`, `userid` | Service created |
| `AfterModuleSuspend` | `serviceid`, `userid` | Service suspended |
| `AfterModuleUnsuspend` | `serviceid`, `userid` | Service reactivated |
| `AfterModuleTerminate` | `serviceid`, `userid` | Service terminated |
| `AfterModuleChangePackage` | `serviceid`, `params` | Package changed |
| `AfterModuleChangePassword` | `serviceid`, `password` | Password changed |
| `PreServiceDelete` | `userid`, `serviceid` | Before service deletion |

## Domain Hooks

| Hook | Parameters | Description |
|------|-------------|-------------|
| `DomainTransferCompleted` | `domainid` | Transfer completed |
| `DomainTransferFailed` | `domainid` | Transfer failed |
| `DomainDeletion` | `domainid` | Domain deleted |
| `DomainRegister` | `domainid` | Domain registered |
| `DomainRenew` | `domainid` | Domain renewed |

## Order Hooks

| Hook | Parameters | Description |
|------|-------------|-------------|
| `OrderAccepted` | `orderid` | Order accepted |
| `OrderPaid` | `orderid`, `userid` | Order payment received |
| `OrderCancelled` | `orderid` | Order cancelled |
| `OrderFraud` | `orderid` | Order flagged as fraud |
| `OrderProductValidation` | `params` | Product validation |

## Ticket Hooks

| Hook | Parameters | Description |
|------|-------------|-------------|
| `TicketOpen` | `ticketid`, `userid`, `deptid` | New ticket created |
| `TicketReply` | `ticketid`, `userid` | Ticket replied |
| `TicketClose` | `ticketid` | Ticket ticket closed |
| `TicketAddNote` | `ticketid` | Note added to ticket |

## Cron Hooks

| Hook | Parameters | Description |
|------|-------------|-------------|
| `DailyCronJob` | - | Daily cron execution |
| `HourlyCronJob` | - | Hourly cron execution |
| `AutoTaxProvision` | - | Auto tax processing |

## Usage Examples

### Send Email on Service Create
```php
add_hook('AfterModuleCreate', 1, function($vars) {
    $service = Capsule::table('tblhosting')->where('id', $vars['serviceid'])->first();

    send_email([
        'type' => 'Product',
        'id' => $service->packageid,
        'customvars' => [
            'service_id' => $vars['serviceid'],
            'domain' => $service->domain,
        ],
    ]);
});
```

### Log All Logins
```php
add_hook('ClientLogin', 1, function($vars) {
    Capsule::table('mod_audit_logs')->insert([
        'user_id' => $vars['userid'],
        'action' => 'login',
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});
```

---

**Related Skills:**
- whmcs-hooks-development
- whmcs-email-template-builder

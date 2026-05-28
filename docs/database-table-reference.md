# WHMCS WHMCS WHMCS WHMCS WHMCS WHMCS...
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Complete WHMCS database table reference for module development.

## Core Tables

### Clients & Users
| Table | Description |
|-------|-------------|
| `tblclients` | Client accounts |
| `tblusers` | User accounts (WHMCS 8+) |
| `tbladdresses` | Client addresses |
| `tblcontacts` | Client contacts |

### Products & Services
| Table | Description |
|-------|-------------|
| `tblproducts` | Product definitions |
| `tblhosting` | Service/Hosting accounts |
| `tblhostingaddons` | Service addons |
| `tblpricing` | Product pricing |
| `tblproductconfiglinks` | Config option links |

### Invoices & Billing
| Table | Description |
|-------|-------------|
| `tblinvoices` | Invoices |
| `tblinvoiceitems` | Invoice line items |
| `tblinvoiceitems` | Invoice items |
| `tbltransfers` | Payment transactions |
| `tbltax` | Tax rules |

### Domains
| Table | Description |
|-------|-------------|
| `tbldomains` | Domain registrations |
| `tbldomainpricing` | Domain pricing |
| `tbl域名contacts` | Registrar contacts |

### Support
| Table | Description |
|-------|-------------|
| `tbltickets` | Support tickets |
| `tblticketmessages` | Ticket messages |
| `tblticketdepartments` | Departments |
| `tblticketpriorities` | Priorities |

### Orders
| Table | Description |
|-------|-------------|
| `tblorders` | Order headers |
| `tblorderitems` | Order line items |
| `tblorderstatuses` | Order statuses |

## Module Table Naming

```
mod_{module}_{purpose}
Examples:
├── mod_example_settings     # Configuration
├── mod_example_data        # Main data
├── mod_example_logs        # Activity logs
├── mod_example_cache       # Cached data
└── mod_example_queue       # Job queue
```

## Common Queries

### Active Services by Client
```php
Capsule::table('tblhosting')
    ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
    ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
    ->where('tblhosting.userid', $userId)
    ->where('tblhosting.domainstatus', 'Active')
    ->select('tblhosting.*', 'tblproducts.name as product_name')
    ->get();
```

### Overdue Invoices
```php
Capsule::table('tblinvoices')
    ->where('status', 'Unpaid')
    ->where('duedate', '<', date('Y-m-d'))
    ->get();
```

### Recent Orders
```php
Capsule::table('tblorders')
    ->where('status', 'Pending')
    ->where('created_at', '>=', date('Y-m-d', strtotime('-24 hours')))
    ->get();
```

---

**Related Skills:**
- whmcs-database-design
- whmcs-core-reader

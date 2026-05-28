# WHMCS Client Management Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building client management modules.

## When to Use

- Client profiles
- Client dashboards
- Client management tools

## Client Dashboard Patterns

```php
<?php
class ClientManager {
    public function getDashboardData(int $userId): array {
        return [
            'services' => $this->getActiveServices($userId),
            'invoices' => $this->getRecentInvoices($userId),
            'tickets' => $this->getOpenTickets($userId),
            'usage' => $this->getResourceUsage($userId),
            'spending' => $this->getPendingActions($userId),
        ];
    }

    public function getServiceSummary(int $userId): array {
        return Capsule::table('tblhosting')
            ->selectRaw("
                COUNT(*) as total,
                SUM(CASE WHEN domainstatus = 'Active' THEN 1 ELSE 0 END) as active,
                SUM(CASE WHEN domainstatus = 'Suspended' THEN 1 ELSE 0 END) as suspended,
                SUM(CASE WHEN domainstatus = 'Terminated' THEN 1 ELSE 0 END) as terminated
            ")
            ->where('userid', $userId)
            ->first();
    }

    public function getBillingSummary(int $userId): array {
        $paid = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->where('status', 'Paid')
            ->whereMonth('datepaid', date('m'))
            ->sum('total');

        $pending = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->whereIn('status', ['Unpaid', 'Overdue'])
            ->sum('total');

        return [
            'paid_this_month' => $paid,
            'pending_balance' => $pending,
        ];
    }

    public function getSecurityStatus(int $userId): array {
        $twofa = Capsule::table('tbltblclient_security')
            ->where('uid', $userId)
            ->where('setting', 'twofaenabled')
            ->value('value');

        return [
            'two_factor_enabled' => $twofa === '1',
            'password_updated' => $this->getLastPasswordUpdate($userId),
            'recent_logins' => $this->getRecentLogins($userId),
        ];
    }
}
```

### Client Area Header
```php
function {module}_clientarea(array $vars): array {
    $dashboard = new ClientManager();
    $data = $dashboard->getDashboardData($_SESSION['uid']);

    return [
        'pagetitle' => 'My Account',
        'templatefile' => 'templates/client_dashboard',
        'vars' => $data,
        'requirelogin' => true,
    ];
}
```

---

**Related Skills:**
- whmcs-clientarea-builder
- whmcs-admin-ui-builder

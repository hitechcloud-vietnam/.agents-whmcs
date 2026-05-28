# WHMCS Service Level Agreement Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building SLA management modules.

## When to Use

- Support SLA tracking
- Uptime monitoring
- Service credits

## SLA Module Patterns

```php
<?php
class SLAManager {
    private array $defaultSLAs = [
        'bronze' => [
            'response_time' => 24 * 60,     // minutes
            'resolution_time' => 72 * 60,
            'uptime' => 99.0,
            'credits' => 5,               // % credit per hour downtime
        ],
        'silver' => [
            'response_time' => 8 * 60,
            'resolution_time' => 24 * 60,
            'uptime' => 99.5,
            'credits' => 10,
        ],
        'gold' => [
            'response_time' => 4 * 60,
            'resolution_time' => 8 * 60,
            'uptime' => 99.9,
            'credits' => 25,
        ],
    ];

    public function getSLAForService(int $serviceId): array {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        $slaName = $service->sla_tier ?? 'bronze';

        return $this->defaultSLAs[$slaName] ?? $this->defaultSLAs['bronze'];
    }

    public function checkResponseTime(int $ticketId): array {
        $ticket = Capsule::table('tbltickets')->where('id', $ticketId)->first();
        $sla = $this->getSLAForService($ticket->serviceid);
        $openedAt = strtotime($ticket->created);
        $respondedAt = strtotime($this->getFirstReplyTime($ticketId));
        $responseMinutes = ($respondedAt - $openedAt) / 60;

        return [
            'within_sla' => $responseMinutes <= $sla['response_time'],
            'response_minutes' => $responseMinutes,
            'sla_minutes' => $sla['response_time'],
            'breach' => $responseMinutes > $sla['response_time'],
        ];
    }

    public function calculateCredits(int $serviceId, float $downtimeHours): array {
        $sla = $this->getSLAForService($serviceId);
        $uptimeRequired = $sla['uptime'];

        $creditPercent = $sla['credits'] * min($downtimeHours, 24);
        $monthlyFee = Capsule::table('tblhosting')->where('id', $serviceId)->value('amount');

        return [
            'credit_percent' => $creditPercent,
            'credit_amount' => ($monthlyFee * $creditPercent) / 100,
        ];
    }
}
```

### Uptime Calculation
```php
public function calculateUptime(int $serviceId, string $period = 'monthly'): float {
    $startDate = match ($period) {
        'daily' => date('Y-m-d', strtotime('-1 day')),
        'weekly' => date('Y-m-d', strtotime('-1 week')),
        'monthly' => date('Y-m-d', strtotime('-1 month')),
        default => date('Y-m-d', strtotime('-1 month')),
    };

    $incidents = Capsule::table('mod_sla_incidents')
        ->where('service_id', $serviceId)
        ->where('created_at', '>=', $startDate)
        ->get();

    $totalMinutes = match ($period) {
        'daily' => 24 * 60,
        'weekly' => 7 * 24 * 60,
        'monthly' => 30 * 24 * 60,
        default => 30 * 24 * 60,
    };

    $downtimeMinutes = array_sum(array_map(fn($i) => $i->duration_minutes, $incidents));

    return round((($totalMinutes - $downtimeMinutes) / $totalMinutes) * 100, 2);
}
```

---

**Related Skills:**
- whmcs-support-ticket-module
- whmcs-reporting

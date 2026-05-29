# WHMCS Support Ticket Report Workflow

## Overview
This workflow generates support ticket analytics and performance reports.

## Prerequisites
- WHMCS with ticket system enabled
- Admin access for reports
- Optional: Survey module for CSAT

## Step-by-Step Process

### Step 1: Create Ticket Report Generator
```php
<?php
// /includes/reports/TicketReportGenerator.php

class TicketReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getTicketSummary($startDate, $endDate),
            'by_status' => $this->getByStatus($startDate, $endDate),
            'by_department' => $this->getByDepartment($startDate, $endDate),
            'by_priority' => $this->getByPriority($startDate, $endDate),
            'response_time' => $this->getResponseTimeMetrics($startDate, $endDate),
            'resolution_time' => $this->getResolutionTimeMetrics($startDate, $endDate),
            'csat' => $this->getCustomerSatisfaction($startDate, $endDate),
            'staff_performance' => $this->getStaffPerformance($startDate, $endDate)
        ];
    }
```

### Step 2: Implement Report Methods
```php
    private function getTicketSummary(string $startDate, string $endDate): array
    {
        return [
            'total_tickets' => Capsule::table('tbltickets')
                ->whereBetween('date', [$startDate, $endDate])
                ->count(),
            'open_tickets' => Capsule::table('tbltickets')
                ->whereIn('status', ['Open', 'Answered'])
                ->count(),
            'closed_today' => Capsule::table('tbltickets')
                ->whereDate('lastactivity', date('Y-m-d'))
                ->count(),
            'avg_response_time' => $this->getAvgResponseTime($startDate, $endDate),
            'resolution_rate' => $this->getResolutionRate($startDate, $endDate)
        ];
    }

    private function getByStatus(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                status,
                COUNT(*) as count,
                AVG(TIMESTAMPDIFF(HOUR, date, COALESCE(lastreply, date))) as avg_hours_open
            FROM tbltickets
            WHERE date BETWEEN ? AND ?
            GROUP BY status
        ", [$startDate, $endDate]);
    }

    private function getByDepartment(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                d.name as department,
                COUNT(t.id) as tickets,
                AVG(TIMESTAMPDIFF(HOUR, t.date, COALESCE(t.lastreply, t.date))) as avg_response_hours,
                SUM(CASE WHEN t.status = 'Open' THEN 1 ELSE 0 END) as open_tickets
            FROM tbldepartments d
            LEFT JOIN tbltickets t ON d.id = t.department
            WHERE t.date BETWEEN ? AND ?
            GROUP BY d.id
            ORDER BY tickets DESC
        ", [$startDate, $endDate]);
    }

    private function getByPriority(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                urgency as priority,
                COUNT(*) as count,
                AVG(TIMESTAMPDIFF(HOUR, date, lastactivity)) as avg_duration_hours
            FROM tbltickets
            WHERE date BETWEEN ? AND ?
            GROUP BY urgency
        ", [$startDate, $endDate]);
    }
```

### Step 3: Implement Time Metrics
```php
    private function getAvgResponseTime(string $startDate, string $endDate): float
    {
        $result = Capsule::select("
            SELECT AVG(TIMESTAMPDIFF(MINUTE, date, first_reply)) as avg_minutes
            FROM (
                SELECT
                    t.id,
                    t.date,
                    MIN(r.date) as first_reply
                FROM tbltickets t
                LEFT JOIN tblticketreplies r ON t.id = r.ticketid
                WHERE t.date BETWEEN ? AND ?
                GROUP BY t.id
            ) as tickets_with_reply
        ", [$startDate, $endDate]);

        return round(($result[0]->avg_minutes ?? 0) / 60, 1);
    }

    private function getResolutionTimeMetrics(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                urgency,
                COUNT(*) as total,
                AVG(TIMESTAMPDIFF(HOUR, date, lastactivity)) as avg_hours,
                MIN(TIMESTAMPDIFF(HOUR, date, lastactivity)) as min_hours,
                MAX(TIMESTAMPDIFF(HOUR, date, lastactivity)) as max_hours
            FROM tbltickets
            WHERE date BETWEEN ? AND ?
            AND status IN ('Closed', 'Resolved')
            GROUP BY urgency
        ", [$startDate, $endDate]);
    }
```

### Step 4: Customer Satisfaction
```php
    private function getCustomerSatisfaction(string $startDate, string $endDate): array
    {
        $surveys = Capsule::select("
            SELECT
                AVG(rating) as avg_rating,
                COUNT(*) as responses,
                SUM(CASE WHEN rating >= 4 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as positive_rate
            FROM mod_ticket_surveys
            WHERE created_at BETWEEN ? AND ?
        ", [$startDate, $endDate]);

        return [
            'avg_rating' => round($surveys[0]->avg_rating ?? 0, 2),
            'responses' => $surveys[0]->responses ?? 0,
            'positive_rate' => round($surveys[0]->positive_rate ?? 0, 1)
        ];
    }
```

### Step 5: Staff Performance
```php
    private function getStaffPerformance(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                r.adminid as staff_id,
                a.username,
                COUNT(DISTINCT r.ticketid) as tickets_handled,
                COUNT(r.id) as replies_given,
                AVG(TIMESTAMPDIFF(MINUTE, t.date, r.date)) as avg_first_reply_minutes,
                SUM(CASE WHEN t.status IN ('Closed', 'Resolved') THEN 1 ELSE 0 END) as resolved
            FROM tblticketreplies r
            JOIN tbladmins a ON r.adminid = a.id
            JOIN tbltickets t ON r.ticketid = t.id
            WHERE r.date BETWEEN ? AND ?
            GROUP BY r.adminid
            ORDER BY tickets_handled DESC
        ", [$startDate, $endDate]);
    }

    private function getResolutionRate(string $startDate, string $endDate): float
    {
        $total = Capsule::table('tbltickets')
            ->whereBetween('date', [$startDate, $endDate])
            ->count();
        $resolved = Capsule::table('tbltickets')
            ->whereBetween('date', [$startDate, $endDate])
            ->whereIn('status', ['Closed', 'Resolved'])
            ->count();

        return $total > 0 ? round(($resolved / $total) * 100, 1) : 0;
    }
}
```

### Step 6: Create Scheduled Report Hook
```php
<?php
// /includes/hooks/ticket_report_hooks.php

add_hook('DailyCronJob', 1, function($vars) {
    $reportGenerator = new TicketReportGenerator();

    $report = $reportGenerator->generate([
        'start_date' => date('Y-m-d', strtotime('-7 days')),
        'end_date' => date('Y-m-d')
    ]);

    // Store report
    Capsule::table('mod_reports')->insert([
        'report_type' => 'ticket_daily',
        'data' => json_encode($report),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    // Alert on SLA violations
    if ($report['summary']['avg_response_time'] > 4) {
        sendAdminEmail('Ticket Response Time Warning', [
            'avg_response_hours' => $report['summary']['avg_response_time']
        ]);
    }

    return $report;
});

add_hook('MonthlyCronJob', 1, function($vars) {
    $reportGenerator = new TicketReportGenerator();

    $report = $reportGenerator->generate([
        'start_date' => date('Y-m-01'),
        'end_date' => date('Y-m-t')
    ]);

    sendAdminEmail('Monthly Ticket Report', [
        'summary' => $report['summary'],
        'staff_performance' => $report['staff_performance'],
        'csat' => $report['csat']
    ]);

    return $report;
});
```

## Report Metrics Reference

| Metric | Description | Target |
|--------|-------------|--------|
| Total Tickets | Number of tickets in period | - |
| Open Tickets | Currently open tickets | Minimize |
| Avg Response Time | Hours to first reply | < 4 hours |
| Resolution Time | Hours to close | < 24 hours |
| Resolution Rate | % tickets closed | > 90% |
| CSAT Score | Customer satisfaction | > 4.0 |

## Related Workflows
- [WHMCS CSAT Report](./whmcs-csat-report.md)
- [WHMCS Report Automation](./whmcs-report-automation.md)
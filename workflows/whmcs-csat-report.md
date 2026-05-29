# WHMCS CSAT Report Workflow

## Overview
This workflow generates Customer Satisfaction reports and metrics.

## Prerequisites
- WHMCS with CSAT survey module
- Admin access for survey reports
- Survey responses collected

## Step-by-Step Process

### Step 1: Create CSAT Report Generator
```php
<?php
// /includes/reports/CSATReportGenerator.php

class CSATReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getCSATSummary($startDate, $endDate),
            'by_channel' => $this->getCSATByChannel($startDate, $endDate),
            'trends' => $this->getCSATTrends(),
            'recent_surveys' => $this->getRecentSurveys($startDate, $endDate),
            'dissatisfied_analysis' => $this->getDissatisfiedAnalysis($startDate, $endDate)
        ];
    }

    private function getCSATSummary(string $startDate, string $endDate): array
    {
        $data = Capsule::select("
            SELECT
                COUNT(*) as total_responses,
                AVG(rating) as avg_rating,
                MIN(rating) as min_rating,
                MAX(rating) as max_rating,
                SUM(CASE WHEN rating >= 4 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as positive_rate,
                SUM(CASE WHEN rating <= 2 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as negative_rate
            FROM mod_csat_surveys
            WHERE created_at BETWEEN ? AND ?
        ", [$startDate, $endDate]);

        return (array)$data[0];
    }

    private function getCSATByChannel(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                source as channel,
                COUNT(*) as responses,
                AVG(rating) as avg_rating,
                SUM(CASE WHEN rating >= 4 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as satisfaction_rate
            FROM mod_csat_surveys
            WHERE created_at BETWEEN ? AND ?
            GROUP BY source
            ORDER BY satisfaction_rate DESC
        ", [$startDate, $endDate]);
    }

    private function getCSATTrends(): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(created_at, '%Y-%m') as month,
                COUNT(*) as responses,
                AVG(rating) as avg_rating,
                SUM(CASE WHEN rating >= 4 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as satisfaction_rate
            FROM mod_csat_surveys
            WHERE created_at >= DATE_SUB(NOW(), INTERVAL 12 MONTH)
            GROUP BY month
            ORDER BY month
        ");
    }

    private function getRecentSurveys(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                s.id,
                s.rating,
                s.feedback,
                s.created_at,
                c.firstname,
                c.email,
                t.subject as ticket_subject
            FROM mod_csat_surveys s
            JOIN tblclients c ON s.client_id = c.id
            LEFT JOIN tbltickets t ON s.ticket_id = t.id
            WHERE s.created_at BETWEEN ? AND ?
            ORDER BY s.created_at DESC
            LIMIT 50
        ", [$startDate, $endDate]);
    }

    private function getDissatisfiedAnalysis(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                feedback,
                rating,
                c.firstname,
                c.email,
                s.created_at
            FROM mod_csat_surveys s
            JOIN tblclients c ON s.client_id = c.id
            WHERE s.created_at BETWEEN ? AND ?
            AND s.rating <= 2
            AND s.feedback IS NOT NULL
            ORDER BY s.created_at DESC
        ", [$startDate, $endDate]);
    }
}
```

## CSAT Rating Scale

| Rating | Satisfaction Level |
|--------|-------------------|
| 5 | Very Satisfied |
| 4 | Satisfied |
| 3 | Neutral |
| 2 | Dissatisfied |
| 1 | Very Dissatisfied |

## Related Workflows
- [WHMCS NPS Report](./whmcs-nps-report.md)
- [WHMCS Ticket Report](./whmcs-ticket-report.md)
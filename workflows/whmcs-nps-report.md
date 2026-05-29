# WHMCS NPS Report Workflow

## Overview
This workflow generates Net Promoter Score survey and analysis reports.

## Prerequisites
- WHMCS with NPS survey module
- Admin access for survey reports
- Survey responses collected

## Step-by-Step Process

### Step 1: Create NPS Report Generator
```php
<?php
// /includes/reports/NPSReportGenerator.php

class NPSReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'nps_score' => $this->calculateNPS($startDate, $endDate),
            'breakdown' => $this->getScoreBreakdown($startDate, $endDate),
            'by_segment' => $this->getNPSBySegment($startDate, $endDate),
            'trends' => $this->getNPSTrends(),
            'feedback_analysis' => $this->analyzeFeedback($startDate, $endDate)
        ];
    }

    private function calculateNPS(string $startDate, string $endDate): array
    {
        $responses = Capsule::select("
            SELECT
                score,
                COUNT(*) as count
            FROM mod_nps_surveys
            WHERE created_at BETWEEN ? AND ?
            GROUP BY score
        ", [$startDate, $endDate]);

        $promoters = 0;
        $passives = 0;
        $detractors = 0;
        $total = 0;

        foreach ($responses as $response) {
            $total += $response->count;
            if ($response->score >= 9) {
                $promoters += $response->count;
            } elseif ($response->score >= 7) {
                $passives += $response->count;
            } else {
                $detractors += $response->count;
            }
        }

        $promoterPercent = $total > 0 ? ($promoters / $total) * 100 : 0;
        $detractorPercent = $total > 0 ? ($detractors / $total) * 100 : 0;
        $nps = $promoterPercent - $detractorPercent;

        return [
            'nps' => round($nps, 1),
            'promoters' => $promoters,
            'passives' => $passives,
            'detractors' => $detractors,
            'total_responses' => $total,
            'promoter_percent' => round($promoterPercent, 1),
            'detractor_percent' => round($detractorPercent, 1)
        ];
    }

    private function getScoreBreakdown(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                score,
                COUNT(*) as responses,
                AVG(sentiment_score) as avg_sentiment
            FROM mod_nps_surveys
            WHERE created_at BETWEEN ? AND ?
            GROUP BY score
            ORDER BY score
        ", [$startDate, $endDate]);
    }

    private function getNPSBySegment(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                c.groupid,
                g.groupname as segment,
                AVG(n.score) as avg_score,
                COUNT(n.id) as responses,
                SUM(CASE WHEN n.score >= 9 THEN 1 WHEN n.score < 7 THEN -1 ELSE 0 END) * 100.0 / NULLIF(COUNT(n.id), 0) as nps
            FROM mod_nps_surveys n
            JOIN tblclients c ON n.client_id = c.id
            LEFT JOIN tblclientgroups g ON c.groupid = g.id
            WHERE n.created_at BETWEEN ? AND ?
            GROUP BY c.groupid
            ORDER BY nps DESC
        ", [$startDate, $endDate]);
    }

    private function getNPSTrends(): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(created_at, '%Y-%m') as month,
                COUNT(*) as responses,
                AVG(score) as avg_score,
                SUM(CASE WHEN score >= 9 THEN 1 WHEN score < 7 THEN -1 ELSE 0 END) * 100.0 / NULLIF(COUNT(*), 0) as nps
            FROM mod_nps_surveys
            WHERE created_at >= DATE_SUB(NOW(), INTERVAL 12 MONTH)
            GROUP BY month
            ORDER BY month
        ");
    }

    private function analyzeFeedback(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                feedback,
                score,
                sentiment_score,
                created_at,
                c.firstname,
                c.email
            FROM mod_nps_surveys n
            JOIN tblclients c ON n.client_id = c.id
            WHERE n.created_at BETWEEN ? AND ?
            AND n.feedback IS NOT NULL
            AND n.feedback != ''
            ORDER BY sentiment_score ASC
            LIMIT 20
        ", [$startDate, $endDate]);
    }
}
```

## NPS Score Interpretation

| NPS Score | Classification |
|-----------|----------------|
| 70+ | Excellent |
| 50-69 | Great |
| 30-49 | Good |
| 0-29 | Needs Improvement |
| -100 to 0 | Critical |

## Related Workflows
- [WHMCS CSAT Report](./whmcs-csat-report.md)
- [WHMCS Ticket Report](./whmcs-ticket-report.md)
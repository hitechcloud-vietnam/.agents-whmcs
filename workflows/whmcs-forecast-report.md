# WHMCS Forecast Report Workflow

## Overview
This workflow generates revenue and growth forecast reports.

## Prerequisites
- WHMCS with historical data
- Admin access for analytics
- Forecasting capabilities

## Step-by-Step Process

### Step 1: Create Forecast Report Generator
```php
<?php
// /includes/reports/ForecastReportGenerator.php

class ForecastReportGenerator
{
    public function generate(array $params = []): array
    {
        $forecastMonths = $params['months'] ?? 12;

        return [
            'revenue_forecast' => $this->getRevenueForecast($forecastMonths),
            'mrr_forecast' => $this->getMRRForecast($forecastMonths),
            'customer_forecast' => $this->getCustomerForecast($forecastMonths),
            'churn_forecast' => $this->getChurnForecast($forecastMonths),
            'scenario_analysis' => $this->getScenarioAnalysis()
        ];
    }

    private function getRevenueForecast(int $months): array
    {
        // Get historical data
        $historical = Capsule::select("
            SELECT
                DATE_FORMAT(datepaid, '%Y-%m') as month,
                SUM(total) as revenue
            FROM tblinvoices
            WHERE status = 'Paid'
            AND datepaid >= DATE_SUB(NOW(), INTERVAL 12 MONTH)
            GROUP BY DATE_FORMAT(datepaid, '%Y-%m')
            ORDER BY month
        ");

        // Calculate average growth rate
        $growthRates = [];
        for ($i = 1; $i < count($historical); $i++) {
            $prev = $historical[$i - 1]->revenue ?? 0;
            $curr = $historical[$i]->revenue ?? 0;
            if ($prev > 0) {
                $growthRates[] = ($curr - $prev) / $prev;
            }
        }

        $avgGrowthRate = count($growthRates) > 0 ? array_sum($growthRates) / count($growthRates) : 0;

        // Generate forecast
        $lastRevenue = end($historical)->revenue ?? 0;
        $forecast = [];

        for ($i = 1; $i <= $months; $i++) {
            $projectedRevenue = $lastRevenue * pow(1 + $avgGrowthRate, $i);
            $forecast[] = [
                'month' => date('Y-m', strtotime("+{$i} months")),
                'projected_revenue' => round($projectedRevenue, 2),
                'confidence' => $this->getConfidenceLevel($i)
            ];
        }

        return [
            'historical' => $historical,
            'forecast' => $forecast,
            'avg_growth_rate' => round($avgGrowthRate * 100, 2)
        ];
    }

    private function getMRRForecast(int $months): array
    {
        $currentMRR = Capsule::select("
            SELECT SUM(
                CASE billingcycle
                    WHEN 'Monthly' THEN amount
                    WHEN 'Quarterly' THEN amount / 3
                    WHEN 'Semi-Annual' THEN amount / 6
                    WHEN 'Annual' THEN amount / 12
                    ELSE amount
                END
            ) as mrr
            FROM tblhosting
            WHERE domainstatus = 'Active'
        ")[0]->mrr ?? 0;

        $monthlyGrowthRate = 0.05; // 5% default

        $forecast = [];
        for ($i = 1; $i <= $months; $i++) {
            $projectedMRR = $currentMRR * pow(1 + $monthlyGrowthRate, $i);
            $forecast[] = [
                'month' => date('Y-m', strtotime("+{$i} months")),
                'projected_mrr' => round($projectedMRR, 2),
                'projected_arr' => round($projectedMRR * 12, 2)
            ];
        }

        return [
            'current_mrr' => round($currentMRR, 2),
            'forecast' => $forecast
        ];
    }

    private function getCustomerForecast(int $months): array
    {
        $currentCustomers = Capsule::table('tblclients')->count();
        $monthlyGrowthRate = 0.03; // 3% default

        $forecast = [];
        for ($i = 1; $i <= $months; $i++) {
            $projectedCustomers = $currentCustomers * pow(1 + $monthlyGrowthRate, $i);
            $forecast[] = [
                'month' => date('Y-m', strtotime("+{$i} months")),
                'projected_customers' => round($projectedCustomers)
            ];
        }

        return [
            'current_customers' => $currentCustomers,
            'forecast' => $forecast
        ];
    }

    private function getChurnForecast(int $months): array
    {
        $avgChurnRate = 0.05; // 5% monthly default

        return Capsule::select("
            SELECT
                DATE_FORMAT(termination_date, '%Y-%m') as month,
                COUNT(*) as churned
            FROM tblhosting
            WHERE termination_date >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
            GROUP BY month
        ");
    }

    private function getScenarioAnalysis(): array
    {
        $currentMRR = Capsule::select("
            SELECT SUM(CASE billingcycle
                WHEN 'Monthly' THEN amount
                WHEN 'Quarterly' THEN amount / 3
                WHEN 'Semi-Annual' THEN amount / 6
                WHEN 'Annual' THEN amount / 12
                ELSE amount
            END) as mrr FROM tblhosting WHERE domainstatus = 'Active'
        ")[0]->mrr ?? 0;

        return [
            'conservative' => [
                'growth_rate' => 3,
                'projected_mrr_12m' => round($currentMRR * pow(1.03, 12), 2)
            ],
            'moderate' => [
                'growth_rate' => 5,
                'projected_mrr_12m' => round($currentMRR * pow(1.05, 12), 2)
            ],
            'aggressive' => [
                'growth_rate' => 10,
                'projected_mrr_12m' => round($currentMRR * pow(1.10, 12), 2)
            ]
        ];
    }

    private function getConfidenceLevel(int $monthsAhead): float
    {
        // Confidence decreases over time
        return max(0.5, 1 - ($monthsAhead * 0.05));
    }
}
```

## Related Workflows
- [WHMCS Trend Report](./whmcs-trend-report.md)
- [WHMCS ARR Report](./whmcs-arr-report.md)
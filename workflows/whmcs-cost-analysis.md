# WHMCS Cost Analysis Workflow
# Version: 1.0 | Created: 2026-05-28

## Purpose

Comprehensive guide to analyzing and optimizing WHMCS operational costs including server costs, licensing, payment processing fees, support costs, and infrastructure expenses.

## Prerequisites

- WHMCS installation with billing data
- Access to hosting provider invoices
- Payment gateway fee records
- Support ticket metrics
- Historical revenue data

## Workflow Steps

### Step 1: Collect Cost Data

Set up comprehensive cost tracking:

```php
// includes/hooks/cost_tracking.php

use WHMCS\Database\Capsule;

/**
 * Track all operational costs
 */
function trackOperationalCosts(): array
{
    return [
        'infrastructure' => getInfrastructureCosts(),
        'licensing'     => getLicensingCosts(),
        'payment_fees'  => getPaymentProcessingFees(),
        'support'      => getSupportCosts(),
        'addons'        => getAddonCosts(),
        'marketing'     => getMarketingCosts(),
    ];
}

function getInfrastructureCosts(): array
{
    $serverCosts = [];

    // Get server hosting costs
    $servers = Capsule::table('tblservers')->get();

    foreach ($servers as $server) {
        $monthlyCost = Capsule::table('mod_server_costs')
            ->where('server_id', $server->id)
            ->where('billing_cycle', 'monthly')
            ->orderBy('id', 'DESC')
            ->first();

        $serverCosts[] = [
            'server_id'   => $server->id,
            'name'         => $server->name,
            'monthly_cost' => $monthlyCost->monthly_amount ?? 0,
            'cost_type'    => $monthlyCost->cost_type ?? 'fixed',
        ];
    }

    return $serverCosts;
}

function getPaymentProcessingFees(): array
{
    $fees = [];

    // Calculate payment gateway fees from transactions
    $gateways = Capsule::table('tblpaymentgateways')->get();

    foreach ($gateways as $gateway) {
        $transactionFees = Capsule::table('tblaccounts')
            ->where('gateway', $gateway->GetValue())
            ->where('date', '>=', date('Y-m-01'))
            ->sum('fees');

        $transactionVolume = Capsule::table('tblaccounts')
            ->where('gateway', $gateway->GetValue())
            ->where('date', '>=', date('Y-m-01'))
            ->sum('amount');

        $fees[] = [
            'gateway'        => $gateway->GetValue(),
            'monthly_fees'   => abs($transactionFees),
            'monthly_volume' => $transactionVolume,
            'fee_percentage' => $transactionVolume > 0
                ? (abs($transactionFees) / $transactionVolume) * 100
                : 0,
        ];
    }

    return $fees;
}

function getSupportCosts(): array
{
    $staff = Capsule::table('tblstaff')->get();

    $costs = [];
    foreach ($staff as $member) {
        $ticketsResolved = Capsule::table('tbltickets')
            ->where('adminid', $member->id)
            ->where('status', 'Closed')
            ->where('lastreply', '>=', date('Y-m-01'))
            ->count();

        $avgResponseTime = Capsule::getOne(
            "SELECT AVG(TIMESTAMPDIFF(SECOND, date, lastreply)) as avg_seconds
             FROM tbltickets WHERE adminid = ? AND lastreply >= ?",
            [$member->id, date('Y-m-01')]
        );

        $costs[] = [
            'staff_id'        => $member->id,
            'name'            => $member->firstname . ' ' . $member->lastname,
            'tickets_resolved'=> $ticketsResolved,
            'avg_response_sec'=> round($avgResponseTime->avg_seconds ?? 0),
        ];
    }

    return $costs;
}
```

### Step 2: Calculate Cost Per Customer Segment

Analyze costs by customer segment:

```php
// modules/addons/cost_analysis/cost_analysis.php

class CostPerCustomerCalculator
{
    /**
     * Calculate total cost per customer
     */
    public function calculateCostPerCustomer(int $userId): array
    {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();

        return [
            'direct_costs'   => $this->getDirectCosts($userId),
            'allocated_costs'=> $this->getAllocatedCosts($userId),
            'support_costs'  => $this->getSupportCostsForClient($userId),
            'total_cost'     => $this->getTotalCustomerCost($userId),
            'revenue'       => $this->getClientRevenue($userId),
            'margin'        => $this->calculateMargin($userId),
        ];
    }

    /**
     * Get direct costs (directly attributable to customer)
     */
    private function getDirectCosts(int $userId): array
    {
        $services = Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->where('domainstatus', 'Active')
            ->get();

        $directCosts = [
            'server_cost'    => 0,
            'license_cost'   => 0,
            'domain_cost'    => 0,
        ];

        foreach ($services as $service) {
            $product = Capsule::table('tblproducts')->where('id', $service->packageid)->first();
            $server = Capsule::table('tblservers')->where('id', $service->server)->first();

            // Server allocation cost
            $serverCost = Capsule::table('mod_server_costs')
                ->where('server_id', $server->id)
                ->first();

            $directCosts['server_cost'] += ($serverCost->monthly_amount ?? 0) *
                ($service->qty ?? 1);

            // Product cost
            $directCosts['license_cost'] += ($product->cost ?? 0) *
                ($service->qty ?? 1);
        }

        // Domain costs
        $domains = Capsule::table('tbldomains')
            ->where('userid', $userId)
            ->where('status', 'Active')
            ->get();

        foreach ($domains as $domain) {
            $tldPricing = Capsule::table('tblpricing')
                ->where('type', 'domain')
                ->first();
            $directCosts['domain_cost'] += $tldPricing-> register ?? 0;
        }

        return $directCosts;
    }

    /**
     * Get allocated overhead costs
     */
    private function getAllocatedCosts(int $userId): array
    {
        $totalCustomers = Capsule::table('tblclients')->count();
        $totalOverhead = $this->getTotalMonthlyOverhead();

        $clientAllocation = $totalOverhead / $totalCustomers;

        return [
            'admin_overhead'   => $clientAllocation * 0.4,
            'infrastructure'   => $clientAllocation * 0.3,
            'marketing'        => $clientAllocation * 0.2,
            'miscellaneous'    => $clientAllocation * 0.1,
        ];
    }

    /**
     * Analyze cost by customer segment
     */
    public function analyzeBySegment(): array
    {
        $segments = [
            'enterprise' => ['min_revenue' => 10000000], // > 10M VND/month
            'business'  => ['min_revenue' => 2000000, 'max_revenue' => 10000000],
            'starter'   => ['max_revenue' => 2000000],
        ];

        $results = [];

        foreach ($segments as $segmentName => $criteria) {
            $query = Capsule::table('tblclients');

            if (isset($criteria['min_revenue'])) {
                $query->where('amountpaid', '>=', $criteria['min_revenue']);
            }
            if (isset($criteria['max_revenue'])) {
                $query->where('amountpaid', '<=', $criteria['max_revenue']);
            }

            $clients = $query->get();
            $segmentCosts = [];

            foreach ($clients as $client) {
                $segmentCosts[] = $this->calculateCostPerCustomer($client->id);
            }

            $results[$segmentName] = [
                'client_count' => count($clients),
                'avg_cost'     => array_sum(array_column($segmentCosts, 'total_cost')) / count($segmentCosts),
                'avg_margin'  => array_sum(array_column($segmentCosts, 'margin')) / count($segmentCosts),
            ];
        }

        return $results;
    }

    private function getTotalMonthlyOverhead(): float
    {
        // Calculate total overhead (staff, infrastructure, licenses)
        $staffCosts = Capsule::table('tblstaff')
            ->sum('monthly_salary');

        $infrastructureCosts = Capsule::table('mod_server_costs')
            ->where('billing_cycle', 'monthly')
            ->sum('monthly_amount');

        return $staffCosts + $infrastructureCosts;
    }

    private function calculateMargin(int $userId): float
    {
        $totalCost = $this->getTotalCustomerCost($userId);
        $revenue = $this->getClientRevenue($userId);

        if ($revenue <= 0) {
            return 0;
        }

        return (($revenue - $totalCost) / $revenue) * 100;
    }
}
```

### Step 3: Create Cost Reporting Dashboard

Build reporting functionality:

```php
// modules/addons/cost_analysis/views/dashboard.tpl

<div class="cost-analysis-dashboard">
    <h2>Cost Analysis Dashboard</h2>

    <div class="panel-container">
        <div class="cost-summary-cards">
            <?php foreach ($summaryCards as $card): ?>
            <div class="cost-card <?= $card['class'] ?>">
                <h3><?= $card['label'] ?></h3>
                <p class="cost-value"><?= formatCurrency($card['value']) ?></p>
                <p class="trend <?= $card['trend'] ?>">
                    <?= $card['trend_icon'] ?> <?= abs($card['change']) ?>%
                </p>
            </div>
            <?php endforeach; ?>
        </div>

        <div class="cost-breakdown-chart">
            <h3>Cost Breakdown</h3>
            <canvas id="costBreakdownChart"></canvas>
        </div>

        <div class="margin-analysis">
            <h3>Profit Margin by Segment</h3>
            <table class="margin-table">
                <thead>
                    <tr>
                        <th>Segment</th>
                        <th>Clients</th>
                        <th>Avg Revenue</th>
                        <th>Avg Cost</th>
                        <th>Margin</th>
                    </tr>
                </thead>
                <tbody>
                    <?php foreach ($margins as $segment => $data): ?>
                    <tr>
                        <td><?= ucfirst($segment) ?></td>
                        <td><?= $data['client_count'] ?></td>
                        <td><?= formatCurrency($data['avg_revenue']) ?></td>
                        <td><?= formatCurrency($data['avg_cost']) ?></td>
                        <td class="<?= $data['margin'] > 20 ? 'positive' : 'negative' ?>">
                            <?= number_format($data['margin'], 1) ?>%
                        </td>
                    </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        </div>
    </div>
</div>
```

### Step 4: Implement Cost Optimization Recommendations

Generate actionable recommendations:

```php
class CostOptimizationEngine
{
    private array $thresholds = [
        'low_margin'        => 15,  // percent
        'high_gateway_fee'  => 3.5, // percent
        'underutilized'    => 30,  // percent server usage
    ];

    /**
     * Generate optimization recommendations
     */
    public function generateRecommendations(): array
    {
        return [
            'pricing'     => $this->analyzePricingStrategy(),
            'gateway'     => $this->analyzePaymentGateways(),
            'infrastructure' => $this->analyzeInfrastructure(),
            'support'     => $this->analyzeSupportEfficiency(),
        ];
    }

    /**
     * Analyze pricing recommendations
     */
    private function analyzePricingStrategy(): array
    {
        $recommendations = [];
        $lowMarginClients = Capsule::table('tblclients')
            ->where('amountpaid', '>', 0)
            ->get()
            ->filter(function($client) {
                $calculator = new CostPerCustomerCalculator();
                $margin = $calculator->calculateCostPerCustomer($client->id)['margin'];
                return $margin < $this->thresholds['low_margin'];
            });

        if ($lowMarginClients->count() > 0) {
            $recommendations[] = [
                'priority'   => 'high',
                'type'       => 'pricing',
                'title'      => 'Review Low-Margin Accounts',
                'description'=> $lowMarginClients->count() . ' clients have margins below '
                    . $this->thresholds['low_margin'] . '%',
                'action'     => 'Consider price adjustments or service optimization',
            ];
        }

        return $recommendations;
    }

    /**
     * Analyze payment gateway costs
     */
    private function analyzePaymentGateways(): array
    {
        $recommendations = [];
        $calculator = new CostPerCustomerCalculator();
        $fees = $calculator->getPaymentProcessingFees();

        foreach ($fees as $fee) {
            if ($fee['fee_percentage'] > $this->thresholds['high_gateway_fee']) {
                $recommendations[] = [
                    'priority'   => 'medium',
                    'type'      => 'gateway',
                    'title'     => 'High ' . ucfirst($fee['gateway']) . ' Fees',
                    'description'=> 'Current fee: ' . number_format($fee['fee_percentage'], 2)
                        . '%. Consider renegotiating or switching providers.',
                    'potential_savings' => $fee['monthly_fees'] * 0.3,
                ];
            }
        }

        return $recommendations;
    }

    /**
     * Generate ROI report
     */
    public function generateROIReport(array $dateRange): array
    {
        $startDate = $dateRange['start'] ?? date('Y-m-01');
        $endDate = $dateRange['end'] ?? date('Y-m-d');

        $revenue = Capsule::table('tblaccounts')
            ->where('date', '>=', $startDate)
            ->where('date', '<=', $endDate)
            ->sum('amount');

        $costs = 0;
        foreach ($this->getTotalMonthlyOverhead() as $costCategory) {
            $costs += $costCategory;
        }

        return [
            'period'         => ['start' => $startDate, 'end' => $endDate],
            'total_revenue'  => $revenue,
            'total_costs'    => $costs,
            'net_profit'     => $revenue - $costs,
            'margin'         => ($revenue > 0) ? (($revenue - $costs) / $revenue) * 100 : 0,
            'cost_breakdown' => $this->getCostBreakdown($startDate, $endDate),
        ];
    }
}
```

### Step 5: Set Up Automated Cost Reports

Schedule automated cost analysis:

```php
// includes/hooks/cost_reporting.php

add_hook('DailyCronJob', 1, function($vars) {
    $reportGenerator = new CostReportGenerator();
    $reportGenerator->generateDailySnapshot();
});

add_hook('WeeklyCronJob', 1, function($vars) {
    $reportGenerator = new CostReportGenerator();
    $reportGenerator->sendWeeklyReport();
});

class CostReportGenerator
{
    public function generateDailySnapshot(): void
    {
        $calculator = new CostPerCustomerCalculator();
        $snapshot = [
            'date'              => date('Y-m-d'),
            'total_revenue'     => $this->getTotalRevenue(date('Y-m-d')),
            'total_costs'       => $calculator->getTotalMonthlyOverhead(),
            'active_clients'    => Capsule::table('tblclients')->where('status', 'Active')->count(),
            'new_signups'       => Capsule::table('tblclients')
                ->where('datecreated', date('Y-m-d'))
                ->count(),
            'churned'           => Capsule::table('tblhosting')
                ->where('domainstatus', 'Terminated')
                ->where('termination_date', date('Y-m-d'))
                ->count(),
        ];

        Capsule::table('mod_cost_snapshots')->insert($snapshot);
    }

    public function sendWeeklyReport(): void
    {
        $report = $this->generateWeeklyAggregates();
        $optimization = new CostOptimizationEngine();

        $emailContent = $this->buildReportEmail($report, $optimization->generateRecommendations());

        sendEmail([
            'to'      => 'admin@yourcompany.com',
            'subject' => 'Weekly Cost Analysis Report - ' . date('Y-m-d'),
            'body'    => $emailContent,
        ]);
    }

    private function buildReportEmail(array $report, array $recommendations): string
    {
        $html = '<h1>Weekly Cost Analysis Report</h1>';
        $html .= '<p>Period: ' . $report['period_start'] . ' to ' . $report['period_end'] . '</p>';

        $html .= '<h2>Summary</h2>';
        $html .= '<ul>';
        $html .= '<li>Revenue: ' . formatCurrency($report['revenue']) . '</li>';
        $html .= '<li>Costs: ' . formatCurrency($report['costs']) . '</li>';
        $html .= '<li>Net Profit: ' . formatCurrency($report['revenue'] - $report['costs']) . '</li>';
        $html .= '<li>Margin: ' . number_format($report['margin'], 1) . '%</li>';
        $html .= '</ul>';

        if (!empty($recommendations)) {
            $html .= '<h2>Recommendations</h2>';
            $html .= '<ul>';
            foreach ($recommendations as $rec) {
                $html .= '<li><strong>' . $rec['title'] . '</strong>: ' . $rec['description'] . '</li>';
            }
            $html .= '</ul>';
        }

        return $html;
    }
}
```

---

## Best Practices

1. **Track all cost categories** - Include hidden costs like admin time and setup effort
2. **Regular cost audits** - Review costs monthly to identify trends early
3. **Benchmark against industry** - Compare your costs with similar providers
4. **Automate data collection** - Reduce manual effort with automated tracking
5. **Analyze per customer** - Understand individual client profitability
6. **Monitor payment fees** - Negotiate better rates as volume grows
7. **Review server utilization** - Right-size infrastructure to actual usage
8. **Consider bulk pricing** - Negotiate volume discounts where applicable
9. **Plan for scalability** - Factor growth into cost optimization decisions
10. **Document all assumptions** - Maintain clear methodology for calculations

---

## Verification Checklist

- [ ] Cost data collection hooks operational
- [ ] Payment gateway fees calculated correctly
- [ ] Server costs assigned to appropriate services
- [ ] Support costs allocated reasonably
- [ ] Cost per customer calculation accurate
- [ ] Margin analysis by segment complete
- [ ] Dashboard displays real-time data
- [ ] Weekly reports generated and sent
- [ ] Optimization recommendations actionable
- [ ] Historical cost data retained for trend analysis

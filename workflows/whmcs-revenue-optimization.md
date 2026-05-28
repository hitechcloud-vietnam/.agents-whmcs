# WHMCS Revenue Optimization Workflow

## Purpose

Implement comprehensive revenue optimization strategies for WHMCS to maximize recurring revenue, identify growth opportunities, and improve billing efficiency. This workflow covers pricing optimization, upselling, cross-selling, and revenue analytics.

## Prerequisites

- WHMCS v8.0+
- Admin access to WHMCS
- Understanding of business metrics
- Historical transaction data

## Workflow Steps

### Step 1: Revenue Analysis Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                   Revenue Optimization Framework                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Revenue Components:                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │   MRR       │  │    ARR      │  │    ARPU     │            │
│  │  (Monthly)  │  │  (Annual)   │  │  (Average)  │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                  │
│  Growth Levers:                                                  │
│  ┌─────────────────┐  ┌─────────────────┐                     │
│  │   Expansion     │  │   New Business  │                     │
│  │  (Upsell/Cross) │  │   (Acquisition) │                     │
│  └─────────────────┘  └─────────────────┘                     │
│  ┌─────────────────┐  ┌─────────────────┐                     │
│  │   Contraction   │  │    Churn        │                     │
│  │  (Downgrades)   │  │   (Revenue Loss)│                     │
│  └─────────────────┘  └─────────────────┘                     │
│                                                                  │
│  Net Revenue:                                                    │
│  New + Expansion - Contraction - Churn = Net Revenue Growth     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Revenue Analytics Implementation

```php
<?php
// /var/www/html/whmcs/includes/helpers/RevenueAnalytics.php

namespace WHMCS\Helpers;

use Illuminate\Database\Capsule\Manager as Capsule;

class RevenueAnalytics
{
    /**
     * Calculate Monthly Recurring Revenue (MRR)
     */
    public static function calculateMRR(): float
    {
        $activeServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->get();
        
        $monthlyTotal = 0;
        
        foreach ($activeServices as $service) {
            // Get billing cycle amount
            $amount = self::getServiceMonthlyAmount($service);
            $monthlyTotal += $amount;
        }
        
        return round($monthlyTotal, 2);
    }
    
    /**
     * Calculate Annual Recurring Revenue (ARR)
     */
    public static function calculateARR(): float
    {
        return self::calculateMRR() * 12;
    }
    
    /**
     * Get average revenue per user
     */
    public static function calculateARPU(): float
    {
        $totalRevenue = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->where('datepaid', '>=', date('Y-m-01'))
            ->sum('total');
        
        $activeClients = Capsule::table('tblclients')
            ->where('status', 'Active')
            ->count();
        
        return $activeClients > 0 ? round($totalRevenue / $activeClients, 2) : 0;
    }
    
    /**
     * Calculate revenue breakdown by product
     */
    public static function getRevenueByProduct(): array
    {
        return Capsule::select("
            SELECT 
                p.name as product_name,
                COUNT(h.id) as subscriptions,
                SUM(CASE 
                    WHEN h.billingcycle = 1 THEN h.amount 
                    WHEN h.billingcycle = 2 THEN h.amount / 3
                    WHEN h.billingcycle = 3 THEN h.amount / 6
                    WHEN h.billingcycle = 4 THEN h.amount / 12
                    ELSE h.amount
                END) as mrr
            FROM tblhosting h
            INNER JOIN tblproducts p ON h.packageid = p.id
            WHERE h.domainstatus = 'Active'
            GROUP BY p.id, p.name
            ORDER BY mrr DESC
        ");
    }
    
    /**
     * Calculate expansion revenue
     */
    public static function calculateExpansionRevenue(string $period = '30 days'): array
    {
        return Capsule::select("
            SELECT 
                SUM(
                    CASE 
                        WHEN billingcycle = 1 THEN (new_amount - old_amount)
                        WHEN billingcycle = 2 THEN (new_amount - old_amount) / 3
                        WHEN billingcycle = 3 THEN (new_amount - old_amount) / 6
                        WHEN billingcycle = 4 THEN (new_amount - old_amount) / 12
                        ELSE 0
                    END
                ) as expansion_mrr,
                COUNT(*) as upgrade_count
            FROM (
                SELECT 
                    h.id,
                    h.billingcycle,
                    COALESCE(old.amount, 0) as old_amount,
                    h.amount as new_amount
                FROM tblhosting h
                LEFT JOIN tblhosting_history old ON h.id = old.hosting_id 
                    AND old.change_type = 'upgrade'
                    AND old.created_at >= DATE_SUB(NOW(), INTERVAL ?)
                WHERE h.domainstatus = 'Active'
                AND h.amount > COALESCE(old.amount, 0)
            ) as upgrades
        ", [$period]);
    }
    
    /**
     * Calculate churned revenue
     */
    public static function calculateChurnedRevenue(string $period = '30 days'): float
    {
        $result = Capsule::select("
            SELECT 
                SUM(CASE 
                    WHEN billingcycle = 1 THEN amount
                    WHEN billingcycle = 2 THEN amount / 3
                    WHEN billingcycle = 3 THEN amount / 6
                    WHEN billingcycle = 4 THEN amount / 12
                    ELSE amount
                END) as churned_mrr
            FROM tblhosting
            WHERE domainstatus IN ('Suspended', 'Terminated')
            AND terminated_date >= DATE_SUB(NOW(), INTERVAL ?)
        ", [$period]);
        
        return $result[0]->churned_mrr ?? 0;
    }
    
    /**
     * Get monthly revenue trend
     */
    public static function getRevenueTrend(int $months = 12): array
    {
        $trend = [];
        
        for ($i = $months - 1; $i >= 0; $i--) {
            $monthStart = date('Y-m-01', strtotime("-{$i} months"));
            $monthEnd = date('Y-m-t', strtotime("-{$i} months"));
            
            $revenue = Capsule::table('tblinvoices')
                ->where('status', 'Paid')
                ->whereBetween('datepaid', [$monthStart, $monthEnd])
                ->sum('total');
            
            $newMRR = Capsule::table('tblhosting')
                ->where('domainstatus', 'Active')
                ->where('regdate', '>=', $monthStart)
                ->where('regdate', '<=', $monthEnd)
                ->sum('amount');
            
            $churnedMRR = Capsule::table('tblhosting')
                ->whereIn('domainstatus', ['Suspended', 'Terminated'])
                ->where('terminated_date', '>=', $monthStart)
                ->where('terminated_date', '<=', $monthEnd)
                ->sum('amount');
            
            $trend[] = [
                'month' => date('Y-m', strtotime("-{$i} months")),
                'revenue' => round($revenue, 2),
                'new_mrr' => round($newMRR, 2),
                'churned_mrr' => round($churnedMRR, 2),
                'net_mrr_change' => round($newMRR - $churnedMRR, 2)
            ];
        }
        
        return $trend;
    }
    
    /**
     * Calculate service monthly amount from billing cycle
     */
    private static function getServiceMonthlyAmount(object $service): float
    {
        $amount = (float) $service->amount;
        
        switch ($service->billingcycle) {
            case 1: // Monthly
                return $amount;
            case 2: // Quarterly
                return $amount / 3;
            case 3: // Semi-Annual
                return $amount / 6;
            case 4: // Annual
                return $amount / 12;
            case 5: // Biennial
                return $amount / 24;
            case 6: // Triennial
                return $amount / 36;
            default:
                return $amount;
        }
    }
}
```

### Step 3: Upsell and Cross-Sell Engine

```php
<?php
// /var/www/html/whmcs/includes/helpers/UpsellEngine.php

namespace WHMCS\Helpers;

class UpsellEngine
{
    /**
     * Get upsell recommendations for client
     */
    public static function getRecommendations(int $clientId): array
    {
        $client = \WHMCS\User\Client::find($clientId);
        $services = $client->services()->where('domainstatus', 'Active')->get();
        $recommendations = [];
        
        foreach ($services as $service) {
            $recommendations = array_merge(
                $recommendations,
                self::getProductUpsells($service),
                self::getAddonsUpsells($service)
            );
        }
        
        // Sort by revenue potential
        usort($recommendations, function($a, $b) {
            return $b['potential_revenue'] - $a['potential_revenue'];
        });
        
        return $recommendations;
    }
    
    /**
     * Get upsell opportunities based on current product
     */
    private static function getProductUpsells(object $service): array
    {
        $productId = $service->packageid;
        
        // Find higher tier products in same category
        $upsells = Capsule::table('tblproducts')
            ->where('gid', function($query) use ($productId) {
                $query->select('gid')
                    ->from('tblproducts')
                    ->where('id', $productId);
            })
            ->where('id', '!=', $productId)
            ->where('hidden', '0')
            ->get();
        
        $recommendations = [];
        foreach ($upsells as $upsell) {
            $recommendations[] = [
                'type' => 'upgrade',
                'from_product' => $service->product->name ?? 'Current Plan',
                'to_product' => $upsell->name,
                'to_product_id' => $upsell->id,
                'price_difference' => $upsell->pricing()->first()->monthly() - $service->amount,
                'potential_revenue' => ($upsell->pricing()->first()->monthly() - $service->amount) * 12,
                'message' => "Upgrade to {$upsell->name} for more features"
            ];
        }
        
        return $recommendations;
    }
    
    /**
     * Get addon recommendations for service
     */
    private static function getAddonsUpsells(object $service): array
    {
        // Get addons not yet purchased for this service
        $purchasedAddons = Capsule::table('tblhostingaddons')
            ->where('hostingid', $service->id)
            ->pluck('addonid')
            ->toArray();
        
        $availableAddons = Capsule::table('tbladdons')
            ->where('hidden', '0')
            ->whereNotIn('id', $purchasedAddons)
            ->get();
        
        $recommendations = [];
        foreach ($availableAddons as $addon) {
            $recommendations[] = [
                'type' => 'addon',
                'addon_name' => $addon->name,
                'addon_id' => $addon->id,
                'service_id' => $service->id,
                'monthly_price' => $addon->monthly,
                'potential_revenue' => $addon->monthly * 12,
                'message' => "Add {$addon->name} to your {$service->domain} service"
            ];
        }
        
        return $recommendations;
    }
    
    /**
     * Track upsell conversion
     */
    public static function trackConversion(int $clientId, string $recommendationType, int $productId): void
    {
        Capsule::table('tblupsell_tracking')->insert([
            'client_id' => $clientId,
            'recommendation_type' => $recommendationType,
            'product_id' => $productId,
            'converted' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Step 4: Pricing Optimization

```php
<?php
// /var/www/html/whmcs/includes/helpers/PricingOptimizer.php

namespace WHMCS\Helpers;

class PricingOptimizer
{
    /**
     * Analyze pricing effectiveness
     */
    public static function analyzePricing(string $period = '90 days'): array
    {
        return [
            'price_elasticity' => self::calculatePriceElasticity($period),
            'conversion_by_price' => self::analyzeConversionByPrice($period),
            'revenue_per_product' => self::analyzeRevenuePerProduct($period),
            'discount_effectiveness' => self::analyzeDiscountEffectiveness($period),
        ];
    }
    
    /**
     * Calculate price elasticity
     */
    private static function calculatePriceElasticity(string $period): array
    {
        // Compare similar products with different pricing
        $products = Capsule::select("
            SELECT 
                p.id,
                p.name,
                p.qty,
                AVG(pr.monthly) as avg_price,
                SUM(CASE WHEN o.status = 'Active' THEN 1 ELSE 0 END) as conversions
            FROM tblproducts p
            LEFT JOIN tblpricing pr ON pr.relid = p.id AND pr.type = 'product'
            LEFT JOIN tblorders o ON o.products LIKE CONCAT('%', p.id, '%')
            WHERE p.hidden = '0'
            AND p.created >= DATE_SUB(NOW(), INTERVAL ?)
            GROUP BY p.id
            HAVING conversions > 0
        ", [$period]);
        
        // Calculate elasticity based on price vs conversion rate
        return $products;
    }
    
    /**
     * Analyze conversion rates by price point
     */
    private static function analyzeConversionByPrice(string $period): array
    {
        return Capsule::select("
            SELECT 
                CASE 
                    WHEN pr.monthly < 10 THEN 'under_10'
                    WHEN pr.monthly < 25 THEN '10_to_25'
                    WHEN pr.monthly < 50 THEN '25_to_50'
                    WHEN pr.monthly < 100 THEN '50_to_100'
                    ELSE 'over_100'
                END as price_range,
                COUNT(DISTINCT o.id) as orders,
                COUNT(DISTINCT v.id) as views,
                COUNT(DISTINCT o.id) * 100.0 / NULLIF(COUNT(DISTINCT v.id), 0) as conversion_rate
            FROM tblorders o
            INNER JOIN tblorders_products op ON o.id = op.order_id
            INNER JOIN tblproducts p ON op.product_id = p.id
            INNER JOIN tblpricing pr ON pr.relid = p.id AND pr.type = 'product'
            LEFT JOIN (
                SELECT COUNT(*) as id, product_id FROM tblcart_items GROUP BY product_id
            ) v ON p.id = v.product_id
            WHERE o.date >= DATE_SUB(NOW(), INTERVAL ?)
            GROUP BY price_range
            ORDER BY price_range
        ", [$period]);
    }
    
    /**
     * Get optimal price recommendations
     */
    public static function getOptimalPriceRecommendation(int $productId): array
    {
        $product = \WHMCS\Product\Product::find($productId);
        $currentPrice = $product->pricing()->first()->monthly();
        
        // Calculate based on competition and market
        $marketData = self::getMarketPricing($product->gid);
        
        $recommendations = [
            'current_price' => $currentPrice,
            'competitor_avg' => $marketData['average'],
            'price_floor' => $marketData['floor'],
            'price_ceiling' => $marketData['ceiling'],
            'suggested_price' => $currentPrice * 1.05, // 5% increase suggestion
            'confidence' => 'medium'
        ];
        
        return $recommendations;
    }
    
    /**
     * Get market pricing data
     */
    private static function getMarketPricing(int $groupId): array
    {
        $products = Capsule::table('tblproducts')
            ->where('gid', $groupId)
            ->where('hidden', '0')
            ->get();
        
        $prices = $products->map(function($p) {
            return $p->pricing()->first()->monthly() ?? 0;
        })->filter()->toArray();
        
        return [
            'average' => count($prices) > 0 ? array_sum($prices) / count($prices) : 0,
            'floor' => count($prices) > 0 ? min($prices) : 0,
            'ceiling' => count($prices) > 0 ? max($prices) : 0,
            'count' => count($prices)
        ];
    }
}
```

### Step 5: Revenue Dashboard Widget

```php
<?php
// /var/www/html/whmcs/modules/widgets/RevenueDashboard.php

namespace WHMCS\Module\Widget;

class RevenueDashboard extends \WHMCS\Module\AbstractWidget
{
    protected $title = 'Revenue Overview';
    protected $author = 'System';
    protected $type = 'dashboard';
    
    public function getData(): array
    {
        $analytics = new \WHMCS\Helpers\RevenueAnalytics();
        
        return [
            'mrr' => $analytics->calculateMRR(),
            'arr' => $analytics->calculateARR(),
            'arpu' => $analytics->calculateARPU(),
            'churn_rate' => $this->calculateChurnRate(),
            'growth_rate' => $this->calculateGrowthRate(),
            'revenue_by_product' => $analytics->getRevenueByProduct(),
            'trend' => $analytics->getRevenueTrend(6)
        ];
    }
    
    private function calculateChurnRate(): float
    {
        $startCustomers = Capsule::table('tblclients')
            ->where('createdat', '<', date('Y-m-01', strtotime('-1 month')))
            ->count();
        
        $churned = Capsule::table('tblclients')
            ->where('status', 'Closed')
            ->where('updated_at', '>=', date('Y-m-01', strtotime('-1 month')))
            ->count();
        
        return $startCustomers > 0 ? round(($churned / $startCustomers) * 100, 2) : 0;
    }
    
    private function calculateGrowthRate(): float
    {
        $thisMonth = \WHMCS\Helpers\RevenueAnalytics::calculateMRR();
        $lastMonth = Capsule::select("
            SELECT SUM(amount) as mrr
            FROM tblhosting
            WHERE domainstatus = 'Active'
            AND regdate <= DATE_SUB(NOW(), INTERVAL 1 MONTH)
        ")[0]->mrr ?? 0;
        
        return $lastMonth > 0 ? round((($thisMonth - $lastMonth) / $lastMonth) * 100, 2) : 0;
    }
    
    public function generateOutput($data): string
    {
        return <<<HTML
<div class="revenue-dashboard">
    <div class="metrics-row">
        <div class="metric">
            <span class="value">\${$data['mrr']}</span>
            <span class="label">Monthly Revenue</span>
        </div>
        <div class="metric">
            <span class="value">\${$data['arr']}</span>
            <span class="label">Annual Revenue</span>
        </div>
        <div class="metric">
            <span class="value">\${$data['arpu']}</span>
            <span class="label">Avg Per User</span>
        </div>
    </div>
    <div class="growth-row">
        <span class="growth-rate {$data['growth_rate'] >= 0 ? 'positive' : 'negative'}">
            {$data['growth_rate']}% growth
        </span>
        <span class="churn-rate">{$data['churn_rate']}% churn</span>
    </div>
</div>
HTML;
    }
}
```

## Revenue Metrics Summary

| Metric | Formula | Target |
|--------|---------|--------|
| MRR | Sum of monthly recurring revenue | Monitor trends |
| ARR | MRR * 12 | Annual planning |
| ARPU | Total Revenue / Active Clients | Industry benchmark |
| Churn Rate | (Churned MRR / Start MRR) * 100 | <5% monthly |
| Growth Rate | ((Current MRR - Previous MRR) / Previous MRR) * 100 | >10% monthly |
| LTV | ARPU / Churn Rate | Target 3x CAC |

## Best Practices

1. **Track Metrics**: Monitor MRR, ARR, ARPU, and churn rate weekly
2. **Reduce Churn**: Even small churn improvements multiply over time
3. **Upsell Existing**: Cheaper than acquiring new customers
4. **Price Testing**: A/B test pricing changes
5. **Segment Analysis**: Different pricing for different segments
6. **Forecast Revenue**: Use historical data for projections

## Common Pitfalls

- **Ignoring Churn**: Small churn compounds negatively
- **Underpricing**: Not testing price elasticity
- **No Upsell Path**: Missing natural upgrade opportunities
- **Complex Pricing**: Confusion reduces conversion
- **One-Time Focus**: Prioritizing one-time over recurring revenue

## Verification Checklist

- [ ] Revenue analytics implemented
- [ ] MRR/ARR tracking configured
- [ ] Upsell engine deployed
- [ ] Pricing optimization analyzed
- [ ] Revenue dashboard created
- [ ] Growth forecasting in place
- [ ] Churn tracking configured

## Related Documentation

- [WHMCS Client Segmentation](whmcs-client-segmentation.md)
- [WHMCS Churn Reduction](whmcs-churn-reduction.md)
- [WHMCS Customer Retention](whmcs-customer-retention.md)
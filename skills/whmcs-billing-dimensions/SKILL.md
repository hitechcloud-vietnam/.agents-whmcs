---
name: whmcs-billing-dimensions
description: Billing dimensions for WHMCS cloud services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Billing Dimensions Skill

## Overview
This skill provides patterns and implementations for managing multi-dimensional billing in WHMCS, including resource-based pricing, usage tracking, cost allocation, and invoice generation.

## Implementation Patterns

### Billing Dimensions Manager
```php
<?php
/**
 * WHMCS Billing Dimensions
 * Manages multi-dimensional billing for cloud services
 */

namespace WHMCS\Module\Server\Billing;

class BillingDimensionsManager {
    private $db;
    private $pricingEngine;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->pricingEngine = new PricingEngine();
    }

    /**
     * Define billing dimension
     */
    public function createDimension(array $params): array {
        $dimensionId = 'dim_' . bin2hex(random_bytes(12));

        $dimension = [
            'id' => $dimensionId,
            'name' => $params['name'],
            'type' => $params['type'], // compute, storage, network, api
            'unit' => $params['unit'], // hours, gb, requests, mbps
            'pricing_model' => $params['pricing_model'] ?? 'fixed', // fixed, tiered, usage
            'base_price' => $params['base_price'] ?? 0,
            'currency' => $params['currency'] ?? 'USD',
            'description' => $params['description'] ?? '',
            'active' => true,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_billing_dimensions', $dimension);

        // Create pricing tiers if tiered pricing
        if ($params['pricing_model'] === 'tiered' && !empty($params['tiers'])) {
            $this->createPricingTiers($dimensionId, $params['tiers']);
        }

        return [
            'success' => true,
            'dimension_id' => $dimensionId,
            'name' => $params['name']
        ];
    }

    /**
     * Record usage for dimension
     */
    public function recordUsage(array $params): array {
        $usageId = 'usg_' . bin2hex(random_bytes(12));

        $usage = [
            'id' => $usageId,
            'service_id' => $params['service_id'],
            'dimension_id' => $params['dimension_id'],
            'quantity' => $params['quantity'],
            'unit' => $params['unit'],
            'period_start' => $params['period_start'],
            'period_end' => $params['period_end'],
            'recorded_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_billing_usage', $usage);

        // Calculate cost
        $cost = $this->pricingEngine->calculateCost(
            $params['dimension_id'],
            $params['quantity']
        );

        return [
            'success' => true,
            'usage_id' => $usageId,
            'quantity' => $params['quantity'],
            'estimated_cost' => $cost
        ];
    }

    /**
     * Generate billing report
     */
    public function generateReport(int $serviceId, array $params): array {
        $from = $params['from'] ?? date('Y-m-01');
        $to = $params['to'] ?? date('Y-m-t');

        $usage = $this->db->select(
            "SELECT dimension_id, SUM(quantity) as total_quantity
             FROM mod_billing_usage
             WHERE service_id = ? AND period_start >= ? AND period_end <= ?
             GROUP BY dimension_id",
            [$serviceId, $from, $to]
        );

        $report = [
            'service_id' => $serviceId,
            'period' => ['from' => $from, 'to' => $to],
            'dimensions' => [],
            'total_cost' => 0
        ];

        foreach ($usage as $usageData) {
            $dimension = $this->getDimension($usageData->dimension_id);
            $cost = $this->pricingEngine->calculateCost(
                $usageData->dimension_id,
                $usageData->total_quantity
            );

            $report['dimensions'][] = [
                'dimension_id' => $usageData->dimension_id,
                'name' => $dimension['name'],
                'type' => $dimension['type'],
                'quantity' => $usageData->total_quantity,
                'unit' => $dimension['unit'],
                'cost' => $cost
            ];

            $report['total_cost'] += $cost;
        }

        return $report;
    }

    /**
     * Create cost allocation
     */
    public function createCostAllocation(array $params): array {
        $allocationId = 'alloc_' . bin2hex(random_bytes(12));

        $allocation = [
            'id' => $allocationId,
            'service_id' => $params['service_id'],
            'project_id' => $params['project_id'] ?? null,
            'dimension_id' => $params['dimension_id'],
            'percentage' => $params['percentage'] ?? 100,
            'tags' => json_encode($params['tags'] ?? []),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_cost_allocations', $allocation);

        return [
            'success' => true,
            'allocation_id' => $allocationId
        ];
    }

    /**
     * Generate invoice line items
     */
    public function generateInvoiceItems(int $serviceId, \DateTime $billingDate): array {
        $periodStart = $billingDate->modify('first day of previous month')->format('Y-m-d');
        $periodEnd = $billingDate->modify('last day of previous month')->format('Y-m-d');

        $usage = $this->db->select(
            "SELECT dimension_id, SUM(quantity) as total_quantity
             FROM mod_billing_usage
             WHERE service_id = ? AND period_start >= ? AND period_end <= ?
             GROUP BY dimension_id",
            [$serviceId, $periodStart, $periodEnd]
        );

        $items = [];

        foreach ($usage as $usageData) {
            $dimension = $this->getDimension($usageData->dimension_id);
            $cost = $this->pricingEngine->calculateCost(
                $usageData->dimension_id,
                $usageData->total_quantity
            );

            $items[] = [
                'description' => "{$dimension['name']} - {$usageData->total_quantity} {$dimension['unit']}",
                'quantity' => 1,
                'unit_price' => $cost,
                'tax' => $cost * 0.1,
                'total' => $cost * 1.1
            ];
        }

        return [
            'service_id' => $serviceId,
            'billing_period' => ['start' => $periodStart, 'end' => $periodEnd],
            'items' => $items,
            'subtotal' => array_sum(array_column($items, 'unit_price')),
            'tax_total' => array_sum(array_column($items, 'tax')),
            'total' => array_sum(array_column($items, 'total'))
        ];
    }

    /**
     * Set custom pricing for customer
     */
    public function setCustomPricing(int $serviceId, string $dimensionId, array $pricing): array {
        $this->db->delete('mod_custom_pricing', [
            'service_id' => $serviceId,
            'dimension_id' => $dimensionId
        ]);

        $this->db->insert('mod_custom_pricing', [
            'service_id' => $serviceId,
            'dimension_id' => $dimensionId,
            'price_per_unit' => $pricing['price_per_unit'] ?? null,
            'discount_percent' => $pricing['discount_percent'] ?? 0,
            'minimum_units' => $pricing['minimum_units'] ?? 0,
            'maximum_units' => $pricing['maximum_units'] ?? null,
            'effective_from' => $pricing['effective_from'] ?? date('Y-m-d'),
            'effective_to' => $pricing['effective_to'] ?? null
        ]);

        return [
            'success' => true,
            'service_id' => $serviceId,
            'dimension_id' => $dimensionId
        ];
    }

    /**
     * Get spending forecast
     */
    public function getSpendingForecast(int $serviceId, int $days = 30): array {
        $currentUsage = $this->db->select(
            "SELECT dimension_id, SUM(quantity) as total_quantity
             FROM mod_billing_usage
             WHERE service_id = ? AND recorded_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
             GROUP BY dimension_id",
            [$serviceId]
        );

        $forecast = [];

        foreach ($currentUsage as $usage) {
            $dimension = $this->getDimension($usage->dimension_id);
            $dailyRate = $usage->total_quantity / 7;
            $projectedUsage = $dailyRate * $days;
            $projectedCost = $this->pricingEngine->calculateCost(
                $usage->dimension_id,
                $projectedUsage
            );

            $forecast[] = [
                'dimension_id' => $usage->dimension_id,
                'name' => $dimension['name'],
                'current_daily_avg' => round($dailyRate, 2),
                'projected_monthly_usage' => round($projectedUsage, 2),
                'projected_monthly_cost' => round($projectedCost, 2)
            ];
        }

        $totalProjected = array_sum(array_column($forecast, 'projected_monthly_cost'));

        return [
            'service_id' => $serviceId,
            'forecast_days' => $days,
            'dimensions' => $forecast,
            'total_projected_cost' => round($totalProjected, 2)
        ];
    }
}

/**
 * Pricing Engine
 */
class PricingEngine {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    public function calculateCost(string $dimensionId, float $quantity): float {
        $dimension = $this->getDimension($dimensionId);

        if (!$dimension) {
            return 0;
        }

        // Check for custom pricing
        $customPricing = $this->getCustomPricing($dimensionId);
        if ($customPricing) {
            return $this->applyCustomPricing($customPricing, $quantity);
        }

        // Apply tiered pricing
        if ($dimension['pricing_model'] === 'tiered') {
            return $this->calculateTieredCost($dimensionId, $quantity);
        }

        // Fixed pricing
        return $dimension['base_price'] * $quantity;
    }

    private function calculateTieredCost(string $dimensionId, float $quantity): float {
        $tiers = $this->db->select(
            "SELECT * FROM mod_pricing_tiers
             WHERE dimension_id = ?
             ORDER BY min_quantity ASC",
            [$dimensionId]
        );

        $totalCost = 0;
        $remainingQuantity = $quantity;

        foreach ($tiers as $tier) {
            if ($remainingQuantity <= 0) break;

            $tierQuantity = min($remainingQuantity, $tier->max_quantity - $tier->min_quantity + 1);
            $totalCost += $tierQuantity * $tier->price_per_unit;
            $remainingQuantity -= $tierQuantity;
        }

        return $totalCost;
    }

    private function applyCustomPricing(array $customPricing, float $quantity): float {
        $baseCost = 0;

        if ($customPricing['price_per_unit']) {
            $baseCost = $customPricing['price_per_unit'] * $quantity;
        } else {
            $dimension = $this->getDimension($customPricing['dimension_id']);
            $baseCost = $dimension['base_price'] * $quantity;
        }

        $discount = $baseCost * ($customPricing['discount_percent'] / 100);
        return $baseCost - $discount;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_billing_dimensions` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(255) NOT NULL,
  `type` ENUM('compute', 'storage', 'network', 'api') NOT NULL,
  `unit` VARCHAR(50) NOT NULL,
  `pricing_model` ENUM('fixed', 'tiered', 'usage') DEFAULT 'fixed',
  `base_price` DECIMAL(10,6) DEFAULT 0,
  `currency` VARCHAR(3) DEFAULT 'USD',
  `description` TEXT,
  `active` TINYINT(1) DEFAULT 1,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_pricing_tiers` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `dimension_id` VARCHAR(50) NOT NULL,
  `min_quantity` INT NOT NULL,
  `max_quantity` INT NOT NULL,
  `price_per_unit` DECIMAL(10,6) NOT NULL,
  `created_at` DATETIME NOT NULL,
  FOREIGN KEY (`dimension_id`) REFERENCES `mod_billing_dimensions`(`id`)
);

CREATE TABLE `mod_billing_usage` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `dimension_id` VARCHAR(50) NOT NULL,
  `quantity` DECIMAL(15,6) NOT NULL,
  `unit` VARCHAR(50) NOT NULL,
  `period_start` DATE NOT NULL,
  `period_end` DATE NOT NULL,
  `recorded_at` DATETIME NOT NULL,
  INDEX `idx_service_period` (`service_id`, `period_start`, `period_end`)
);

CREATE TABLE `mod_cost_allocations` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `project_id` VARCHAR(50),
  `dimension_id` VARCHAR(50) NOT NULL,
  `percentage` DECIMAL(5,2) DEFAULT 100,
  `tags` TEXT,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_custom_pricing` (
  `service_id` INT NOT NULL,
  `dimension_id` VARCHAR(50) NOT NULL,
  `price_per_unit` DECIMAL(10,6),
  `discount_percent` DECIMAL(5,2) DEFAULT 0,
  `minimum_units` INT DEFAULT 0,
  `maximum_units` INT,
  `effective_from` DATE NOT NULL,
  `effective_to` DATE,
  PRIMARY KEY (`service_id`, `dimension_id`)
);
```

## Default Billing Dimensions

| Dimension | Type | Unit | Pricing Model | Base Price |
|-----------|------|------|---------------|------------|
| CPU Hours | Compute | hours | Tiered | $0.02/hour |
| Memory GB | Compute | GB-hours | Tiered | $0.01/GB |
| Storage GB | Storage | GB-month | Fixed | $0.10/GB |
| Outbound Bandwidth | Network | GB | Tiered | $0.12/GB |
| API Requests | API | per 1000 | Tiered | $0.50/1000 |
| IOPS | Storage | IOPS | Fixed | $0.05/IOPS |
| Snapshot GB | Storage | GB-month | Fixed | $0.05/GB |

## Best Practices

1. **Granular Tracking**: Track usage at component level
2. **Transparent Pricing**: Show breakdown on invoices
3. **Accurate Allocation**: Use tags for cost center assignment
4. **Forecast Integration**: Help customers predict costs
5. **Credit Handling**: Support credits and adjustments

## Related Skills

- whmcs-cost-tracking
- whmcs-resource-quotas
- whmcs-invoice-generation
- whmcs-overage-billing
---
name: whmcs-reserved-instances
description: Reserved capacity for WHMCS cloud services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Reserved Instances Skill

## Overview
This skill provides patterns and implementations for managing reserved instances in WHMCS, including reservation lifecycle, capacity planning, pricing, and renewal management.

## Implementation Patterns

### Reserved Instance Manager
```php
<?php
/**
 * WHMCS Reserved Instances
 * Manages reserved capacity for cloud services
 */

namespace WHMCS\Module\Server\Reserved;

class ReservedInstanceManager {
    private $db;
    private $capacityPlanner;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->capacityPlanner = new CapacityPlanner();
    }

    /**
     * Create reservation
     */
    public function createReservation(array $params): array {
        $reservationId = 'res_' . bin2hex(random_bytes(12));

        $reservation = [
            'id' => $reservationId,
            'service_id' => $params['service_id'] ?? null,
            'customer_id' => $params['customer_id'],
            'instance_type' => $params['instance_type'],
            'quantity' => $params['quantity'] ?? 1,
            'region' => $params['region'] ?? 'default',
            'term' => $params['term'] ?? 1, // years
            'payment_option' => $params['payment_option'] ?? 'all_upfront', // all_upfront, partial_upfront, no_upfront
            'fixed_price' => $params['fixed_price'] ?? 0,
            'hourly_backup_rate' => $params['hourly_backup_rate'] ?? 0,
            'status' => 'pending',
            'reserved_at' => date('Y-m-d H:i:s'),
            'effective_from' => $params['effective_from'] ?? date('Y-m-d'),
            'expires_at' => date('Y-m-d', strtotime("+{$params['term']} years")),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_reserved_instances', $reservation);

        // Calculate and record upfront cost
        if ($params['payment_option'] !== 'no_upfront') {
            $upfrontCost = $this->calculateUpfrontCost($reservation);
            $this->recordPayment($reservationId, $upfrontCost, 'upfront');
        }

        return [
            'success' => true,
            'reservation_id' => $reservationId,
            'expires_at' => $reservation['expires_at']
        ];
    }

    /**
     * List reservations
     */
    public function listReservations(array $filters = []): array {
        $query = "SELECT r.*, c.companyname as customer_name
                  FROM mod_reserved_instances r
                  LEFT JOIN tblclients c ON r.customer_id = c.id
                  WHERE 1=1";

        $bindings = [];

        if (!empty($filters['customer_id'])) {
            $query .= " AND r.customer_id = ?";
            $bindings[] = $filters['customer_id'];
        }

        if (!empty($filters['status'])) {
            $query .= " AND r.status = ?";
            $bindings[] = $filters['status'];
        }

        if (!empty($filters['expiring_soon'])) {
            $query .= " AND r.expires_at <= DATE_ADD(NOW(), INTERVAL 30 DAY)";
        }

        $query .= " ORDER BY r.created_at DESC";

        $reservations = $this->db->select($query, $bindings);

        return array_map(function($res) {
            return [
                'id' => $res->id,
                'customer_name' => $res->customer_name,
                'instance_type' => $res->instance_type,
                'quantity' => $res->quantity,
                'term' => $res->term,
                'status' => $res->status,
                'effective_from' => $res->effective_from,
                'expires_at' => $res->expires_at,
                'savings_percent' => $this->calculateSavingsPercent($res)
            ];
        }, $reservations);
    }

    /**
     * Get reservation utilization
     */
    public function getUtilization(string $reservationId): array {
        $reservation = $this->getReservation($reservationId);

        if (!$reservation) {
            throw new \Exception("Reservation not found: {$reservationId}");
        }

        // Get instances using this reservation
        $instances = $this->db->select(
            "SELECT * FROM mod_service_instances
             WHERE reservation_id = ?",
            [$reservationId]
        );

        $totalReserved = $reservation['quantity'];
        $inUse = count($instances);
        $available = $totalReserved - $inUse;

        return [
            'reservation_id' => $reservationId,
            'instance_type' => $reservation['instance_type'],
            'total_reserved' => $totalReserved,
            'in_use' => $inUse,
            'available' => $available,
            'utilization_percent' => round(($inUse / $totalReserved) * 100, 2),
            'status' => $reservation['status']
        ];
    }

    /**
     * Modify reservation
     */
    public function modifyReservation(string $reservationId, array $changes): array {
        $reservation = $this->getReservation($reservationId);

        if (!$reservation) {
            throw new \Exception("Reservation not found: {$reservationId}");
        }

        $updateData = [];

        if (isset($changes['quantity'])) {
            $updateData['quantity'] = $changes['quantity'];
        }

        if (isset($changes['instance_type'])) {
            $updateData['instance_type'] = $changes['instance_type'];
        }

        $this->db->update('mod_reserved_instances', $updateData, ['id' => $reservationId]);

        return [
            'success' => true,
            'reservation_id' => $reservationId,
            'changes' => array_keys($updateData)
        ];
    }

    /**
     * Exchange reservation
     */
    public function exchangeReservation(string $reservationId, string $newInstanceType): array {
        $reservation = $this->getReservation($reservationId);

        if (!$reservation) {
            throw new \Exception("Reservation not found: {$reservationId}");
        }

        // Validate exchange eligibility
        if (!$this->isExchangeEligible($reservation)) {
            throw new \Exception("Reservation not eligible for exchange");
        }

        // Calculate price difference
        $priceDifference = $this->calculateExchangePriceDiff(
            $reservation['instance_type'],
            $newInstanceType,
            $reservation['term']
        );

        // Update reservation
        $this->db->update('mod_reserved_instances', [
            'instance_type' => $newInstanceType,
            'exchanged_at' => date('Y-m-d H:i:s')
        ], ['id' => $reservationId]);

        // Record price difference billing
        if ($priceDifference != 0) {
            $this->recordExchangeBilling($reservationId, $priceDifference);
        }

        return [
            'success' => true,
            'reservation_id' => $reservationId,
            'old_type' => $reservation['instance_type'],
            'new_type' => $newInstanceType,
            'price_difference' => $priceDifference
        ];
    }

    /**
     * Renewal reminder and processing
     */
    public function processRenewals(): array {
        $expiringSoon = $this->db->select(
            "SELECT * FROM mod_reserved_instances
             WHERE status = 'active'
             AND expires_at BETWEEN NOW() AND DATE_ADD(NOW(), INTERVAL 30 DAY)"
        );

        $processed = [];

        foreach ($expiringSoon as $reservation) {
            // Check if auto-renew is enabled
            if ($reservation->auto_renew) {
                $this->renewReservation($reservation->id);
                $processed[] = ['id' => $reservation->id, 'action' => 'renewed'];
            } else {
                // Send reminder
                $this->sendRenewalReminder($reservation);
                $processed[] = ['id' => $reservation->id, 'action' => 'reminder_sent'];
            }
        }

        return [
            'processed_count' => count($processed),
            'details' => $processed
        ];
    }

    /**
     * Get savings report
     */
    public function getSavingsReport(string $reservationId): array {
        $reservation = $this->getReservation($reservationId);

        if (!$reservation) {
            throw new \Exception("Reservation not found: {$reservationId}");
        }

        // Calculate on-demand equivalent cost
        $onDemandRate = $this->getOnDemandRate($reservation['instance_type'], $reservation['region']);
        $reservedRate = $this->getReservedRate($reservation);

        $hoursPerYear = 8760;
        $totalHours = $hoursPerYear * $reservation['term'];

        $onDemandCost = $onDemandRate * $totalHours * $reservation['quantity'];
        $reservedCost = $reservedRate * $totalHours * $reservation['quantity'] + $this->getUpfrontCost($reservation);

        $savings = $onDemandCost - $reservedCost;
        $savingsPercent = ($savings / $onDemandCost) * 100;

        return [
            'reservation_id' => $reservationId,
            'on_demand_cost' => round($onDemandCost, 2),
            'reserved_cost' => round($reservedCost, 2),
            'total_savings' => round($savings, 2),
            'savings_percent' => round($savingsPercent, 2),
            'term_years' => $reservation['term']
        ];
    }

    // Private helper methods

    private function calculateUpfrontCost(array $reservation): float {
        $onDemandRate = $this->getOnDemandRate($reservation['instance_type'], $reservation['region']);
        $hoursPerYear = 8760;

        $totalOnDemand = $onDemandRate * $hoursPerYear * $reservation['term'] * $reservation['quantity'];
        $discountRate = $this->getReservedDiscountRate($reservation['payment_option']);
        $upfrontCost = $totalOnDemand * (1 - $discountRate) * 0.3;

        return $upfrontCost;
    }

    private function calculateSavingsPercent($reservation): float {
        $onDemandRate = $this->getOnDemandRate($reservation->instance_type, $reservation->region);
        $reservedRate = $this->getReservedRate($reservation);

        if ($onDemandRate == 0) return 0;

        return round((($onDemandRate - $reservedRate) / $onDemandRate) * 100, 2);
    }

    private function getOnDemandRate(string $instanceType, string $region): float {
        $rate = $this->db->select(
            "SELECT hourly_rate FROM mod_instance_types WHERE type = ? AND region = ?",
            [$instanceType, $region]
        );

        return $rate[0]->hourly_rate ?? 0.05;
    }

    private function getReservedRate($reservation): float {
        $onDemandRate = $this->getOnDemandRate($reservation->instance_type, $reservation->region);
        $discountRate = $this->getReservedDiscountRate($reservation->payment_option);

        return $onDemandRate * (1 - $discountRate);
    }

    private function getReservedDiscountRate(string $paymentOption): float {
        return match($paymentOption) {
            'all_upfront' => 0.40,
            'partial_upfront' => 0.30,
            'no_upfront' => 0.20,
            default => 0.30
        };
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_reserved_instances` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT,
  `customer_id` INT NOT NULL,
  `instance_type` VARCHAR(100) NOT NULL,
  `quantity` INT DEFAULT 1,
  `region` VARCHAR(50) DEFAULT 'default',
  `term` INT DEFAULT 1,
  `payment_option` ENUM('all_upfront', 'partial_upfront', 'no_upfront') DEFAULT 'all_upfront',
  `fixed_price` DECIMAL(10,2),
  `hourly_backup_rate` DECIMAL(10,6),
  `status` ENUM('pending', 'active', 'expired', 'cancelled') DEFAULT 'pending',
  `auto_renew` TINYINT(1) DEFAULT 0,
  `reserved_at` DATETIME,
  `effective_from` DATE NOT NULL,
  `expires_at` DATE NOT NULL,
  `cancelled_at` DATETIME,
  `exchanged_at` DATETIME,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_customer_id` (`customer_id`),
  INDEX `idx_status` (`status`),
  INDEX `idx_expires_at` (`expires_at`)
);

CREATE TABLE `mod_instance_types` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `type` VARCHAR(100) NOT NULL,
  `region` VARCHAR(50) DEFAULT 'default',
  `vcpu` INT,
  `memory_gb` INT,
  `hourly_rate` DECIMAL(10,6) NOT NULL,
  `reserved_hourly_rate` DECIMAL(10,6),
  `description` TEXT
);

CREATE TABLE `mod_reserved_payments` (
  `id` VARCHAR(50) PRIMARY KEY,
  `reservation_id` VARCHAR(50) NOT NULL,
  `type` ENUM('upfront', 'hourly', 'exchange') NOT NULL,
  `amount` DECIMAL(10,2) NOT NULL,
  `status` ENUM('pending', 'completed', 'refunded') DEFAULT 'pending',
  `created_at` DATETIME NOT NULL,
  FOREIGN KEY (`reservation_id`) REFERENCES `mod_reserved_instances`(`id`)
);
```

## Instance Type Pricing Matrix

| Instance Type | vCPU | Memory (GB) | On-Demand ($/hr) | 1-Year All Upfront | 3-Year All Upfront |
|---------------|------|-------------|------------------|-------------------|-------------------|
| small | 1 | 2 | $0.02 | $0.012 | $0.008 |
| medium | 2 | 4 | $0.04 | $0.024 | $0.016 |
| large | 4 | 8 | $0.08 | $0.048 | $0.032 |
| xlarge | 8 | 16 | $0.16 | $0.096 | $0.064 |

## Best Practices

1. **Capacity Planning**: Analyze usage patterns before reserving
2. **Term Selection**: Match term to expected usage duration
3. **Payment Options**: Balance upfront payment vs cash flow
4. **Monitoring Utilization**: Track reservation usage
5. **Renewal Management**: Process renewals before expiration

## Related Skills

- whmcs-spot-instances
- whmcs-billing-dimensions
- whmcs-resource-quotas
- whmcs-cost-tracking
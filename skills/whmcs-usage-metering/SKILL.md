# WHMCS Usage Metering Skill

## Purpose
Provides patterns for implementing usage metering in WHMCS SaaS applications, measuring resource consumption, tracking metered usage, and calculating consumption-based charges.

## Implementation Patterns

### Usage Meter
```php
<?php
class UsageMeter {
    private $db;
    
    public function recordMeteredUsage($clientId, $meterType, $quantity, $metadata = []) {
        $meter = [
            'client_id' => $clientId,
            'meter_type' => $meterType,
            'quantity' => $quantity,
            'unit_price' => $this->getUnitPrice($meterType),
            'metadata' => json_encode($metadata),
            'recorded_at' => date('Y-m-d H:i:s')
        ];
        
        $this->db->insert('mod_metered_usage', $meter);
        
        // Update running totals
        $this->updateRunningTotals($clientId, $meterType, $quantity);
    }
    
    private function getUnitPrice($meterType) {
        $prices = [
            'api_calls' => 0.001,
            'storage_gb' => 0.10,
            'bandwidth_gb' => 0.05,
            'compute_hours' => 0.15
        ];
        
        return $prices[$meterType] ?? 0;
    }
    
    private function updateRunningTotals($clientId, $meterType) {
        $monthStart = date('Y-m-01');
        
        $existing = $this->db->select(
            "SELECT * FROM mod_metered_totals 
             WHERE client_id = ? AND meter_type = ? AND period_start = ?",
            [$clientId, $meterType, $monthStart]
        );
        
        if ($existing) {
            $this->db->where('id', $existing['id'])->update('mod_metered_totals', [
                'total_quantity' => $existing['total_quantity'] + $quantity,
                'total_cost' => ($existing['total_quantity'] + $quantity) * $this->getUnitPrice($meterType)
            ]);
        } else {
            $this->db->insert('mod_metered_totals', [
                'client_id' => $clientId,
                'meter_type' => $meterType,
                'period_start' => $monthStart,
                'total_quantity' => $quantity,
                'total_cost' => $quantity * $this->getUnitPrice($meterType)
            ]);
        }
    }
    
    public function getUsageReport($clientId, $period = 'monthly') {
        return $this->db->select(
            "SELECT meter_type, SUM(quantity) as total_quantity, SUM(quantity * unit_price) as total_cost
             FROM mod_metered_usage
             WHERE client_id = ? AND recorded_at >= DATE_SUB(NOW(), INTERVAL ?)
             GROUP BY meter_type",
            [$clientId, $period]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_metered_usage (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    meter_type VARCHAR(50),
    quantity DECIMAL(15,4),
    unit_price DECIMAL(10,4),
    metadata JSON,
    recorded_at DATETIME
);

CREATE TABLE mod_metered_totals (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    meter_type VARCHAR(50),
    period_start DATE,
    total_quantity DECIMAL(15,4),
    total_cost DECIMAL(10,2)
);
```

## Usage Examples
```php
$meter = new UsageMeter();
$meter->recordMeteredUsage($clientId, 'api_calls', 1000, ['endpoint' => '/users']);
$report = $meter->getUsageReport($clientId);
```

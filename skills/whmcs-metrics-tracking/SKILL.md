# WHMCS Metrics Tracking Skill

## Purpose
Provides patterns for implementing metrics tracking in WHMCS SaaS environments, collecting data, tracking KPIs, and generating analytics reports.

## Implementation Patterns

### Metrics Tracker
```php
<?php
class MetricsTracker {
    private $db;
    
    public function track($metricType, $value, $context = []) {
        $metric = [
            'metric_type' => $metricType,
            'value' => $value,
            'context' => json_encode($context),
            'recorded_at' => date('Y-m-d H:i:s')
        ];
        
        $this->db->insert('mod_metrics', $metric);
        
        // Update aggregated metrics
        $this->updateAggregations($metricType, $value);
    }
    
    private function updateAggregations($metricType, $value) {
        $today = date('Y-m-d');
        
        $existing = $this->db->select(
            "SELECT * FROM mod_metrics_daily WHERE metric_type = ? AND date = ?",
            [$metricType, $today]
        );
        
        if ($existing) {
            $this->db->where('metric_type', $metricType)->where('date', $today)->update('mod_metrics_daily', [
                'total_value' => $existing['total_value'] + $value,
                'count' => $existing['count'] + 1,
                'min_value' => min($existing['min_value'], $value),
                'max_value' => max($existing['max_value'], $value)
            ]);
        } else {
            $this->db->insert('mod_metrics_daily', [
                'metric_type' => $metricType,
                'date' => $today,
                'total_value' => $value,
                'count' => 1,
                'min_value' => $value,
                'max_value' => $value
            ]);
        }
    }
    
    public function getMetric($metricType, $period = '7d') {
        $startDate = $this->getPeriodStart($period);
        
        return $this->db->select(
            "SELECT DATE(recorded_at) as date, AVG(value) as avg_value, 
                    SUM(value) as total_value, COUNT(*) as count
             FROM mod_metrics
             WHERE metric_type = ? AND recorded_at >= ?
             GROUP BY DATE(recorded_at)
             ORDER BY date DESC",
            [$metricType, $startDate]
        );
    }
    
    public function getDashboardMetrics() {
        return [
            'daily_active_users' => $this->getMetric('active_users', '1d'),
            'monthly_revenue' => $this->getMetric('revenue', '30d'),
            'support_tickets' => $this->getMetric('tickets_created', '7d'),
            'conversion_rate' => $this->getMetric('conversion', '30d')
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_metrics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    metric_type VARCHAR(50),
    value DECIMAL(15,4),
    context JSON,
    recorded_at DATETIME,
    INDEX idx_type_date (metric_type, recorded_at)
);

CREATE TABLE mod_metrics_daily (
    id INT AUTO_INCREMENT PRIMARY KEY,
    metric_type VARCHAR(50),
    date DATE,
    total_value DECIMAL(15,4),
    count INT,
    min_value DECIMAL(15,4),
    max_value DECIMAL(15,4),
    UNIQUE KEY idx_type_date (metric_type, date)
);
```

## Usage Examples
```php
$tracker = new MetricsTracker();
$tracker->track('api_calls', $count, ['client_id' => $clientId, 'endpoint' => '/api/data']);

$dashboard = $tracker->getDashboardMetrics();
```

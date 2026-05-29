# WHMCS Cohort Analysis Skill

## Purpose
Provides patterns for implementing cohort analysis in WHMCS, segmenting customers by acquisition date, tracking retention over time, and analyzing customer behavior patterns.

## Implementation Patterns

### Cohort Analyzer
```php
<?php
class CohortAnalyzer {
    private $db;
    
    public function analyzeRetention($cohortPeriod = 'monthly') {
        $clients = $this->db->select(
            "SELECT id, DATE_FORMAT(regdate, '%Y-%m') as cohort, regdate
             FROM tblclients WHERE status = 'Active'"
        );
        
        // Group by cohort
        $cohorts = [];
        foreach ($clients as $client) {
            $cohortKey = $client['cohort'];
            if (!isset($cohorts[$cohortKey])) {
                $cohorts[$cohortKey] = ['clients' => [], 'start_date' => $client['regdate']];
            }
            $cohorts[$cohortKey]['clients'][] = $client['id'];
        }
        
        // Calculate retention for each cohort
        $retentionData = [];
        foreach ($cohorts as $cohortKey => $cohort) {
            $retentionData[$cohortKey] = [
                'cohort' => $cohortKey,
                'size' => count($cohort['clients']),
                'retention_by_month' => $this->calculateRetentionCurve($cohort['clients'])
            ];
        }
        
        return $retentionData;
    }
    
    private function calculateRetentionCurve($clientIds) {
        $retention = [];
        $totalClients = count($clientIds);
        
        for ($month = 0; $month <= 12; $month++) {
            $activeCount = 0;
            
            foreach ($clientIds as $clientId) {
                if ($this->isClientActive($clientId, $month)) {
                    $activeCount++;
                }
            }
            
            $retention[$month] = [
                'active' => $activeCount,
                'retained' => $totalClients > 0 ? ($activeCount / $totalClients) * 100 : 0
            ];
        }
        
        return $retention;
    }
    
    private function isClientActive($clientId, $monthsAgo) {
        $checkDate = date('Y-m-d', strtotime("-{$monthsAgo} months"));
        
        $services = $this->db->select(
            "SELECT COUNT(*) as count FROM tblhosting
             WHERE userid = ? AND domainstatus = 'Active'
             AND created_at <= ?",
            [$clientId, $checkDate]
        );
        
        return $services['count'] > 0;
    }
    
    public function getRevenueByCohort($cohort) {
        $clients = $this->db->select(
            "SELECT id FROM tblclients WHERE DATE_FORMAT(regdate, '%Y-%m') = ?",
            [$cohort]
        );
        
        $clientIds = array_column($clients, 'id');
        
        if (empty($clientIds)) return 0;
        
        $revenue = $this->db->select(
            "SELECT SUM(monthly_amount) as total FROM tblhosting
             WHERE userid IN (" . implode(',', $clientIds) . ") AND domainstatus = 'Active'"
        );
        
        return $revenue['total'] ?? 0;
    }
    
    public function getCohortLifetimeValue($cohort) {
        $clients = $this->db->select(
            "SELECT id FROM tblclients WHERE DATE_FORMAT(regdate, '%Y-%m') = ?",
            [$cohort]
        );
        
        $totalLTV = 0;
        foreach ($clients as $client) {
            $ltv = $this->db->select(
                "SELECT SUM(amount) as total FROM tblaccounts WHERE userid = ?",
                [$client['id']]
            )['total'] ?? 0;
            $totalLTV += $ltv;
        }
        
        return [
            'cohort' => $cohort,
            'total_ltv' => $totalLTV,
            'avg_ltv' => count($clients) > 0 ? $totalLTV / count($clients) : 0,
            'customer_count' => count($clients)
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_cohort_snapshots (
    id INT AUTO_INCREMENT PRIMARY KEY,
    cohort VARCHAR(7),
    period INT,
    active_count INT,
    revenue DECIMAL(10,2),
    recorded_at DATETIME
);
```

## Usage Examples
```php
$analyzer = new CohortAnalyzer();
$retention = $analyzer->analyzeRetention();

foreach ($retention as $cohort => $data) {
    echo "{$cohort}: {$data['size']} customers, ";
    echo "Month 1 retention: " . round($data['retention_by_month'][1]['retained'], 1) . "%\n";
}
```

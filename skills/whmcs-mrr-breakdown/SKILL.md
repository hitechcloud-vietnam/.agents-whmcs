# WHMCS MRR Breakdown Skill

## Purpose
Provides patterns for implementing MRR (Monthly Recurring Revenue) breakdown analysis in WHMCS, segmenting revenue, tracking changes, and analyzing revenue composition.

## Implementation Patterns

### MRR Breakdown Analyzer
```php
<?php
class MRRBreakdown {
    private $db;
    
    public function getMRRBreakdown($date = null) {
        $date = $date ?? date('Y-m-01');
        
        return [
            'total_mrr' => $this->calculateTotalMRR(),
            'by_plan' => $this->getMRRByPlan(),
            'by_client_group' => $this->getMRRByClientGroup(),
            'new_mrr' => $this->getNewMRR($date),
            'churned_mrr' => $this->getChurnedMRR($date),
            'expansion_mrr' => $this->getExpansionMRR($date),
            'contraction_mrr' => $this->getContractionMRR($date),
            'net_new_mrr' => $this->calculateNetNewMRR($date)
        ];
    }
    
    private function calculateTotalMRR() {
        $result = $this->db->select(
            "SELECT SUM(monthly_amount) as total FROM tblhosting
             WHERE domainstatus = 'Active' AND billingcycle = 'Monthly'"
        );
        
        // Add pro-rated quarterly/annual
        $otherBilling = $this->db->select(
            "SELECT SUM(monthly_amount) as total FROM tblhosting
             WHERE domainstatus = 'Active' AND billingcycle IN ('Quarterly', 'Annually')"
        );
        
        return ($result['total'] ?? 0) + (($otherBilling['total'] ?? 0) / 3);
    }
    
    private function getMRRByPlan() {
        return $this->db->select(
            "SELECT p.name, SUM(h.monthly_amount) as mrr, COUNT(*) as subscribers
             FROM tblhosting h
             JOIN tblproducts p ON p.id = h.packageid
             WHERE h.domainstatus = 'Active'
             GROUP BY p.name
             ORDER BY mrr DESC"
        );
    }
    
    private function getMRRByClientGroup() {
        return $this->db->select(
            "SELECT cg.groupname, SUM(h.monthly_amount) as mrr, COUNT(DISTINCT h.userid) as clients
             FROM tblhosting h
             JOIN tblclients c ON c.id = h.userid
             LEFT JOIN tblclientgroups cg ON cg.id = c.groupid
             WHERE h.domainstatus = 'Active'
             GROUP BY cg.groupname
             ORDER BY mrr DESC"
        );
    }
    
    private function getNewMRR($date) {
        $newServices = $this->db->select(
            "SELECT SUM(monthly_amount) as total FROM tblhosting
             WHERE domainstatus = 'Active' AND created_at >= ?",
            [$date]
        );
        
        return $newServices['total'] ?? 0;
    }
    
    private function getChurnedMRR($date) {
        $churnedServices = $this->db->select(
            "SELECT SUM(monthly_amount) as total FROM tblhosting
             WHERE domainstatus = 'Terminated' AND termiinated_at >= ?",
            [$date]
        );
        
        return $churnedServices['total'] ?? 0;
    }
    
    private function getExpansionMRR($date) {
        // Compare current vs previous month
        return $this->db->select(
            "SELECT SUM(new_amount - old_amount) as total
             FROM (
                 SELECT h.monthly_amount as new_amount,
                        (SELECT monthly_amount FROM mod_hosting_history 
                         WHERE hosting_id = h.id AND recorded_at < ?) as old_amount
                 FROM tblhosting h
                 WHERE h.domainstatus = 'Active' AND h.updated_at >= ?
             ) sub
             WHERE new_amount > old_amount",
            [$date, $date]
        )['total'] ?? 0;
    }
    
    private function calculateNetNewMRR($date) {
        $breakdown = $this->getMRRBreakdown($date);
        
        return $breakdown['new_mrr'] - $breakdown['churned_mrr'] + 
               $breakdown['expansion_mrr'] - $breakdown['contraction_mrr'];
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_mrr_snapshots (
    id INT AUTO_INCREMENT PRIMARY KEY,
    snapshot_date DATE,
    total_mrr DECIMAL(10,2),
    new_mrr DECIMAL(10,2),
    churned_mrr DECIMAL(10,2),
    expansion_mrr DECIMAL(10,2),
    contraction_mrr DECIMAL(10,2),
    recorded_at DATETIME
);
```

## Usage Examples
```php
$analyzer = new MRRBreakdown();
$breakdown = $analyzer->getMRRBreakdown();

echo "Total MRR: $" . number_format($breakdown['total_mrr'], 2);
echo "Net New MRR: $" . number_format($breakdown['net_new_mrr'], 2);
```

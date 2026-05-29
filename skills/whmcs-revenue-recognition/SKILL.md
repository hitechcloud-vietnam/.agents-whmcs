# WHMCS Revenue Recognition Skill

## Purpose
Provides patterns for implementing revenue recognition in WHMCS, tracking deferred revenue, managing subscription recognition schedules, and calculating recognized revenue.

## Implementation Patterns

### Revenue Recognition
```php
<?php
class RevenueRecognition {
    private $db;
    
    public function recognizeRevenue($invoiceId, $amount, $method = 'straight_line') {
        $invoice = $this->getInvoice($invoiceId);
        
        switch ($method) {
            case 'immediate':
                $this->recordImmediateRecognition($invoiceId, $amount);
                break;
            case 'straight_line':
                $this->createStraightLineSchedule($invoiceId, $amount, $invoice['subscription_months']);
                break;
            case 'percentage_complete':
                $this->calculatePercentageComplete($invoiceId, $amount);
                break;
        }
    }
    
    private function createStraightLineSchedule($invoiceId, $amount, $months) {
        $monthlyAmount = $amount / $months;
        $startDate = date('Y-m-01');
        
        for ($i = 0; $i < $months; $i++) {
            $recognitionDate = date('Y-m-01', strtotime("+{$i} months", strtotime($startDate)));
            
            $this->db->insert('mod_revenue_recognition_schedules', [
                'invoice_id' => $invoiceId,
                'amount' => $monthlyAmount,
                'recognition_date' => $recognitionDate,
                'status' => 'pending'
            ]);
        }
    }
    
    public function processMonthlyRecognition() {
        $today = date('Y-m-01');
        
        $schedules = $this->db->select(
            "SELECT * FROM mod_revenue_recognition_schedules
             WHERE recognition_date <= ? AND status = 'pending'",
            [$today]
        );
        
        foreach ($schedules as $schedule) {
            $this->db->where('id', $schedule['id'])->update('mod_revenue_recognition_schedules', [
                'status' => 'recognized',
                'recognized_at' => date('Y-m-d H:i:s')
            ]);
            
            $this->db->insert('mod_recognized_revenue', [
                'invoice_id' => $schedule['invoice_id'],
                'amount' => $schedule['amount'],
                'recognized_at' => date('Y-m-d H:i:s')
            ]);
        }
        
        return count($schedules);
    }
    
    public function getDeferredRevenue($clientId = null) {
        $query = "SELECT SUM(amount) as total FROM mod_revenue_recognition_schedules WHERE status = 'pending'";
        $params = [];
        
        if ($clientId) {
            $query .= " AND invoice_id IN (SELECT id FROM tblinvoices WHERE userid = ?)";
            $params[] = $clientId;
        }
        
        $result = $this->db->select($query, $params);
        return $result['total'] ?? 0;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_revenue_recognition_schedules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    invoice_id INT,
    amount DECIMAL(10,2),
    recognition_date DATE,
    status ENUM('pending', 'recognized') DEFAULT 'pending',
    recognized_at DATETIME
);

CREATE TABLE mod_recognized_revenue (
    id INT AUTO_INCREMENT PRIMARY KEY,
    invoice_id INT,
    amount DECIMAL(10,2),
    recognized_at DATETIME
);
```

## Usage Examples
```php
$recognition = new RevenueRecognition();
$recognition->recognizeRevenue($invoiceId, 1200, 'straight_line');
$deferred = $recognition->getDeferredRevenue($clientId);
```

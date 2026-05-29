# WHMCS Alert System Skill

## Purpose
Provides patterns for implementing alert management systems in WHMCS, routing notifications, managing escalation, and tracking alert resolution.

## Implementation Patterns

### Alert Manager
```php
<?php
class AlertManager {
    private $db;
    
    public function createAlert($data) {
        $alert = [
            'type' => $data['type'],
            'severity' => $data['severity'] ?? 'medium',
            'title' => $data['title'],
            'message' => $data['message'],
            'entity_type' => $data['entity_type'] ?? null,
            'entity_id' => $data['entity_id'] ?? null,
            'status' => 'pending',
            'assigned_to' => $data['assigned_to'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        $alertId = $this->db->insert('mod_alerts', $alert);
        
        $this->routeAlert($alertId);
        
        return $alertId;
    }
    
    private function routeAlert($alertId) {
        $alert = $this->getAlert($alertId);
        
        $channels = $this->getNotificationChannels($alert['severity']);
        
        foreach ($channels as $channel) {
            $this->sendNotification($alert, $channel);
        }
    }
    
    private function getNotificationChannels($severity) {
        $channels = ['database'];
        
        if ($severity === 'high' || $severity === 'critical') {
            $channels[] = 'email';
            $channels[] = 'sms';
        }
        
        return $channels;
    }
    
    public function acknowledgeAlert($alertId, $userId) {
        $this->db->where('id', $alertId)->update('mod_alerts', [
            'status' => 'acknowledged',
            'acknowledged_by' => $userId,
            'acknowledged_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function resolveAlert($alertId, $userId, $resolution = null) {
        $this->db->where('id', $alertId)->update('mod_alerts', [
            'status' => 'resolved',
            'resolved_by' => $userId,
            'resolved_at' => date('Y-m-d H:i:s'),
            'resolution' => $resolution
        ]);
    }
    
    public function getActiveAlerts() {
        return $this->db->select(
            "SELECT * FROM mod_alerts WHERE status IN ('pending', 'acknowledged') ORDER BY severity DESC, created_at ASC"
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_alerts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    type VARCHAR(50),
    severity ENUM('low', 'medium', 'high', 'critical'),
    title VARCHAR(255),
    message TEXT,
    entity_type VARCHAR(50),
    entity_id INT,
    status ENUM('pending', 'acknowledged', 'resolved') DEFAULT 'pending',
    assigned_to INT,
    acknowledged_by INT,
    acknowledged_at DATETIME,
    resolved_by INT,
    resolved_at DATETIME,
    resolution TEXT,
    created_at DATETIME
);
```

## Usage Examples
```php
$alerts = new AlertManager();
$alerts->createAlert([
    'type' => 'payment_failed',
    'severity' => 'high',
    'title' => 'Payment failed',
    'message' => 'Multiple failed payments detected'
]);
```

# WHMCS Escalation Process Skill

## Purpose
Provides patterns for implementing escalation processes in WHMCS, managing ticket escalation, service level escalation, and priority escalation workflows.

## Implementation Patterns

### Escalation Manager
```php
<?php
class EscalationManager {
    private $db;
    
    public function escalate($entityType, $entityId, $reason, $targetLevel = null) {
        $entity = $this->getEntity($entityType, $entityId);
        
        $escalation = [
            'entity_type' => $entityType,
            'entity_id' => $entityId,
            'reason' => $reason,
            'from_level' => $entity['priority'] ?? 'medium',
            'to_level' => $targetLevel ?? $this->getNextLevel($entity['priority'] ?? 'medium'),
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        $escalationId = $this->db->insert('mod_escalations', $escalation);
        
        // Execute escalation actions
        $this->executeEscalation($escalationId);
        
        return $escalationId;
    }
    
    private function getNextLevel($currentLevel) {
        $levels = ['low' => 'medium', 'medium' => 'high', 'high' => 'urgent', 'urgent' => 'critical'];
        return $levels[$currentLevel] ?? 'critical';
    }
    
    private function executeEscalation($escalationId) {
        $escalation = $this->getEscalation($escalationId);
        
        // Update entity priority
        $this->updateEntityPriority($escalation['entity_type'], $escalation['entity_id'], $escalation['to_level']);
        
        // Assign to senior support
        $assignee = $this->getEscalationAssignee($escalation['to_level']);
        $this->assignToEscalation($escalationId, $assignee);
        
        // Send notifications
        $this->sendEscalationNotification($escalation);
        
        // Log escalation
        $this->logEscalation($escalation);
        
        $this->db->where('id', $escalationId)->update('mod_escalations', [
            'status' => 'completed',
            'completed_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    private function getEscalationAssignee($level) {
        $assignments = [
            'high' => $this->getSeniorSupport(),
            'urgent' => $this->getTeamLead(),
            'critical' => $this->getManager()
        ];
        
        return $assignments[$level] ?? $this->getSeniorSupport();
    }
    
    public function checkAutoEscalation() {
        $items = $this->db->select(
            "SELECT * FROM mod_escalation_triggers 
             WHERE last_checked < DATE_SUB(NOW(), INTERVAL 15 MINUTE)"
        );
        
        foreach ($items as $trigger) {
            $this->processEscalationTrigger($trigger);
        }
    }
    
    private function processEscalationTrigger($trigger) {
        $entity = $this->db->select(
            "SELECT * FROM {$trigger['entity_table']} WHERE id = ?",
            [$trigger['entity_id']]
        );
        
        $age = (time() - strtotime($entity['created_at'])) / 60;
        
        if ($age > $trigger['threshold_minutes']) {
            $this->escalate(
                $trigger['entity_type'],
                $trigger['entity_id'],
                "Auto-escalation: {$trigger['reason']}"
            );
        }
        
        $this->db->where('id', $trigger['id'])->update('mod_escalation_triggers', [
            'last_checked' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_escalations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    entity_type VARCHAR(50),
    entity_id INT,
    reason TEXT,
    from_level VARCHAR(50),
    to_level VARCHAR(50),
    assigned_to INT,
    status ENUM('pending', 'completed', 'cancelled') DEFAULT 'pending',
    created_at DATETIME,
    completed_at DATETIME
);

CREATE TABLE mod_escalation_triggers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    entity_type VARCHAR(50),
    entity_table VARCHAR(100),
    entity_id INT,
    reason VARCHAR(255),
    threshold_minutes INT,
    last_checked DATETIME
);
```

## Usage Examples
```php
$escalation = new EscalationManager();
$escalation->escalate('ticket', $ticketId, 'SLA breach imminent', 'high');
```

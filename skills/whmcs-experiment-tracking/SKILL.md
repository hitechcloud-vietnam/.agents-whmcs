# WHMCS Experiment Tracking Skill

## Purpose
Provides patterns for implementing experiment tracking in WHMCS, monitoring experiment health, tracking participant progress, and analyzing experiment outcomes.

## Implementation Patterns

### Experiment Tracker
```php
<?php
class ExperimentTracker {
    private $db;
    
    public function trackParticipant($experimentId, $participantId, $event, $data = []) {
        $this->db->insert('mod_experiment_events', [
            'experiment_id' => $experimentId,
            'participant_id' => $participantId,
            'event' => $event,
            'data' => json_encode($data),
            'occurred_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function enrollParticipant($experimentId, $participantId, $variant) {
        $this->db->insert('mod_experiment_participants', [
            'experiment_id' => $experimentId,
            'participant_id' => $participantId,
            'variant' => $variant,
            'enrolled_at' => date('Y-m-d H:i:s'),
            'status' => 'active'
        ]);
    }
    
    public function getParticipantProgress($experimentId, $participantId) {
        $events = $this->db->select(
            "SELECT event, data, occurred_at FROM mod_experiment_events
             WHERE experiment_id = ? AND participant_id = ?
             ORDER BY occurred_at ASC",
            [$experimentId, $participantId]
        );
        
        return [
            'enrolled_at' => $events[0]['occurred_at'] ?? null,
            'event_count' => count($events),
            'events' => $events
        ];
    }
    
    public function getExperimentHealth($experimentId) {
        return $this->db->select(
            "SELECT 
                COUNT(*) as total_participants,
                COUNT(CASE WHEN status = 'active' THEN 1 END) as active,
                COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed,
                COUNT(CASE WHEN status = 'dropped' THEN 1 END) as dropped,
                AVG(TIMESTAMPDIFF(HOUR, enrolled_at, completed_at)) as avg_completion_hours
             FROM mod_experiment_participants
             WHERE experiment_id = ?",
            [$experimentId]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_experiment_participants (
    id INT AUTO_INCREMENT PRIMARY KEY,
    experiment_id INT,
    participant_id INT,
    variant VARCHAR(50),
    status ENUM('active', 'completed', 'dropped'),
    enrolled_at DATETIME,
    completed_at DATETIME
);

CREATE TABLE mod_experiment_events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    experiment_id INT,
    participant_id INT,
    event VARCHAR(100),
    data JSON,
    occurred_at DATETIME
);
```

## Usage Examples
```php
$tracker = new ExperimentTracker();
$tracker->enrollParticipant($experimentId, $clientId, 'variant_a');
$tracker->trackParticipant($experimentId, $clientId, 'feature_used', ['feature' => 'dashboard']);
```

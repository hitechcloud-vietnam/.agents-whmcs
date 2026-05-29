# WHMCS Case Tracking Skill

## Purpose
Provides patterns for implementing case management and tracking in WHMCS, handling customer issues, tracking resolutions, and managing case workflows.

## Implementation Patterns

### Case Manager
```php
<?php
class CaseManager {
    private $db;
    
    public function createCase($data) {
        $case = [
            'case_number' => $this->generateCaseNumber(),
            'client_id' => $data['client_id'],
            'type' => $data['type'],
            'priority' => $data['priority'] ?? 'medium',
            'title' => $data['title'],
            'description' => $data['description'],
            'status' => 'open',
            'assigned_to' => $data['assigned_to'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return $this->db->insert('mod_cases', $case);
    }
    
    private function generateCaseNumber() {
        return 'CASE-' . date('Ymd') . '-' . str_pad(mt_rand(1, 9999), 4, '0', STR_PAD_LEFT);
    }
    
    public function addNote($caseId, $note, $userId) {
        $this->db->insert('mod_case_notes', [
            'case_id' => $caseId,
            'note' => $note,
            'created_by' => $userId,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function updateStatus($caseId, $status) {
        $this->db->where('id', $caseId)->update('mod_cases', [
            'status' => $status,
            'updated_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function assignCase($caseId, $userId) {
        $this->db->where('id', $caseId)->update('mod_cases', [
            'assigned_to' => $userId,
            'updated_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function getCase($caseId) {
        return $this->db->select(
            "SELECT c.*, u.firstname, u.lastname 
             FROM mod_cases c
             LEFT JOIN tblusers u ON u.id = c.assigned_to
             WHERE c.id = ?",
            [$caseId]
        );
    }
    
    public function getClientCases($clientId) {
        return $this->db->select(
            "SELECT * FROM mod_cases WHERE client_id = ? ORDER BY created_at DESC",
            [$clientId]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_cases (
    id INT AUTO_INCREMENT PRIMARY KEY,
    case_number VARCHAR(50) UNIQUE,
    client_id INT NOT NULL,
    type VARCHAR(50),
    priority ENUM('low', 'medium', 'high', 'urgent'),
    title VARCHAR(255),
    description TEXT,
    status ENUM('open', 'in_progress', 'resolved', 'closed') DEFAULT 'open',
    assigned_to INT,
    created_at DATETIME,
    updated_at DATETIME,
    resolved_at DATETIME
);

CREATE TABLE mod_case_notes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    case_id INT NOT NULL,
    note TEXT,
    created_by INT,
    created_at DATETIME
);
```

## Usage Examples
```php
$caseManager = new CaseManager();
$caseId = $caseManager->createCase([
    'client_id' => $clientId,
    'type' => 'billing',
    'title' => 'Invoice dispute'
]);

$caseManager->addNote($caseId, 'Investigating the issue', $adminId);
```

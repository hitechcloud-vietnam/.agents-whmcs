# WHMCS Approval Workflow Skill

## Purpose
Provides patterns for implementing approval workflows in WHMCS, managing approval chains, handling multi-level approvals, and tracking approval status.

## Implementation Patterns

### Approval Workflow
```php
<?php
class ApprovalWorkflow {
    private $db;
    
    public function createApproval($data) {
        $approval = [
            'reference_type' => $data['reference_type'],
            'reference_id' => $data['reference_id'],
            'amount' => $data['amount'] ?? null,
            'requested_by' => $data['requested_by'],
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        $approvalId = $this->db->insert('mod_approvals', $approval);
        
        // Create approval chain
        $this->createApprovalChain($approvalId, $data['steps']);
        
        // Notify first approver
        $this->notifyNextApprover($approvalId);
        
        return $approvalId;
    }
    
    private function createApprovalChain($approvalId, $steps) {
        foreach ($steps as $index => $step) {
            $this->db->insert('mod_approval_steps', [
                'approval_id' => $approvalId,
                'step_order' => $index + 1,
                'approver_type' => $step['type'],
                'approver_id' => $step['id'],
                'threshold' => $step['threshold'] ?? null,
                'status' => 'pending'
            ]);
        }
    }
    
    public function approve($stepId, $approverId, $comment = null) {
        $step = $this->db->select("SELECT * FROM mod_approval_steps WHERE id = ?", [$stepId]);
        
        $this->db->where('id', $stepId)->update('mod_approval_steps', [
            'status' => 'approved',
            'approved_by' => $approverId,
            'approved_at' => date('Y-m-d H:i:s'),
            'comment' => $comment
        ]);
        
        // Check if more approvals needed
        $approval = $this->db->select("SELECT * FROM mod_approvals WHERE id = ?", [$step['approval_id']]);
        
        if ($this->isApprovalComplete($approval['id'])) {
            $this->completeApproval($approval['id']);
        } else {
            $this->notifyNextApprover($approval['id']);
        }
    }
    
    public function reject($stepId, $approverId, $reason) {
        $step = $this->db->select("SELECT * FROM mod_approval_steps WHERE id = ?", [$stepId]);
        
        $this->db->where('id', $stepId)->update('mod_approval_steps', [
            'status' => 'rejected',
            'approved_by' => $approverId,
            'approved_at' => date('Y-m-d H:i:s'),
            'comment' => $reason
        ]);
        
        // Reject entire approval
        $this->db->where('id', $step['approval_id'])->update('mod_approvals', [
            'status' => 'rejected',
            'rejection_reason' => $reason,
            'rejected_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    private function isApprovalComplete($approvalId) {
        $pending = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_approval_steps 
             WHERE approval_id = ? AND status = 'pending'",
            [$approvalId]
        );
        return $pending['count'] == 0;
    }
    
    private function completeApproval($approvalId) {
        $this->db->where('id', $approvalId)->update('mod_approvals', [
            'status' => 'approved',
            'completed_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    private function notifyNextApprover($approvalId) {
        $nextStep = $this->db->select(
            "SELECT * FROM mod_approval_steps WHERE approval_id = ? AND status = 'pending' ORDER BY step_order ASC LIMIT 1",
            [$approvalId]
        );
        
        if ($nextStep) {
            // Send notification
            $this->sendApprovalNotification($nextStep);
        }
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_approvals (
    id INT AUTO_INCREMENT PRIMARY KEY,
    reference_type VARCHAR(50),
    reference_id INT,
    amount DECIMAL(15,2),
    requested_by INT,
    status ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    created_at DATETIME,
    completed_at DATETIME,
    rejected_at DATETIME,
    rejection_reason TEXT
);

CREATE TABLE mod_approval_steps (
    id INT AUTO_INCREMENT PRIMARY KEY,
    approval_id INT NOT NULL,
    step_order INT,
    approver_type VARCHAR(50),
    approver_id INT,
    threshold DECIMAL(15,2),
    status ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    approved_by INT,
    approved_at DATETIME,
    comment TEXT
);
```

## Usage Examples
```php
$workflow = new ApprovalWorkflow();

$approvalId = $workflow->createApproval([
    'reference_type' => 'order',
    'reference_id' => $orderId,
    'amount' => 5000,
    'requested_by' => $userId,
    'steps' => [
        ['type' => 'role', 'id' => $managerRoleId, 'threshold' => 1000],
        ['type' => 'role', 'id' => $directorRoleId, 'threshold' => 5000]
    ]
]);
```

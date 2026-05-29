# WHMCS Document Workflow Skill

## Purpose
Provides patterns for implementing document workflows in WHMCS, automating document processing, managing approval chains, tracking document status, and integrating with storage systems.

## Implementation Patterns

### Document Workflow Engine
```php
<?php
// includes/DocumentWorkflow.class.php

class DocumentWorkflowEngine {
    private $db;
    private $workflows = [];
    
    public function __construct() {
        $this->db = console::db();
        $this->loadWorkflowDefinitions();
    }
    
    private function loadWorkflowDefinitions() {
        $this->workflows = $this->db->select(
            "SELECT * FROM mod_document_workflows WHERE is_active = 1"
        );
    }
    
    // Initialize a document workflow
    public function startWorkflow($workflowCode, $documentId, $context = []) {
        $workflow = $this->getWorkflow($workflowCode);
        
        if (!$workflow) {
            throw new Exception("Workflow not found: {$workflowCode}");
        }
        
        $instance = [
            'workflow_id' => $workflow['id'],
            'workflow_code' => $workflowCode,
            'document_id' => $documentId,
            'current_step' => 1,
            'status' => 'active',
            'context' => json_encode($context),
            'started_by' => $context['user_id'] ?? null,
            'started_at' => date('Y-m-d H:i:s'),
            'completed_at' => null
        ];
        
        $instanceId = $this->db->insert('mod_document_workflow_instances', $instance);
        
        // Initialize first step
        $this->executeStep($instanceId, 1);
        
        return $instanceId;
    }
    
    private function executeStep($instanceId, $stepNumber) {
        $instance = $this->getInstance($instanceId);
        $steps = json_decode($instance['workflow_id']['steps'], true);
        
        $step = $steps[$stepNumber - 1] ?? null;
        
        if (!$step) {
            $this->completeWorkflow($instanceId);
            return;
        }
        
        $stepInstance = [
            'instance_id' => $instanceId,
            'step_number' => $stepNumber,
            'step_name' => $step['name'],
            'step_type' => $step['type'],
            'assigned_to' => $this->resolveAssignee($step['assignee'], $instance),
            'status' => 'pending',
            'deadline' => $this->calculateDeadline($step),
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        $stepId = $this->db->insert('mod_document_workflow_steps', $stepInstance);
        
        // Execute step action
        $this->executeStepAction($stepId, $step, $instance);
    }
    
    private function executeStepAction($stepId, $step, $instance) {
        switch ($step['type']) {
            case 'approval':
                $this->sendApprovalRequest($stepId, $step, $instance);
                break;
                
            case 'review':
                $this->sendReviewRequest($stepId, $step, $instance);
                break;
                
            case 'signature':
                $this->sendSignatureRequest($stepId, $step, $instance);
                break;
                
            case 'notification':
                $this->sendNotification($stepId, $step, $instance);
                $this->completeStep($stepId);
                $this->moveToNextStep($instance['id']);
                break;
                
            case 'transformation':
                $this->executeTransformation($stepId, $step, $instance);
                break;
                
            case 'condition':
                $this->evaluateCondition($stepId, $step, $instance);
                break;
                
            case 'integration':
                $this->callIntegration($stepId, $step, $instance);
                break;
                
            default:
                $this->completeStep($stepId);
                $this->moveToNextStep($instance['id']);
        }
    }
    
    // Process approval action
    public function processApproval($stepId, $decision, $approverId, $comment = null) {
        $step = $this->getStep($stepId);
        $instance = $this->getInstance($step['instance_id']);
        
        $approval = [
            'step_id' => $stepId,
            'approver_id' => $approverId,
            'decision' => $decision, // 'approved', 'rejected', 'rejected_with_amendments'
            'comment' => $comment,
            'decided_at' => date('Y-m-d H:i:s')
        ];
        
        $this->db->insert('mod_document_step_approvals', $approval);
        
        // Update step status
        $this->db->where('id', $stepId)
            ->update('mod_document_workflow_steps', [
                'status' => $decision === 'approved' ? 'approved' : 'rejected',
                'decided_at' => date('Y-m-d H:i:s')
            ]);
        
        // Log the decision
        $this->logStepDecision($step, $approval);
        
        // Handle decision
        if ($decision === 'rejected') {
            $this->handleRejection($instance, $step, $comment);
        } else {
            $this->moveToNextStep($instance['id']);
        }
        
        // Send notifications
        $this->notifyStepOutcome($step, $decision, $instance);
        
        return true;
    }
    
    private function handleRejection($instance, $step, $comment) {
        $workflow = $this->getWorkflow($instance['workflow_code']);
        
        // Update instance
        $this->db->where('id', $instance['id'])
            ->update('mod_document_workflow_instances', [
                'status' => 'rejected',
                'rejection_reason' => $comment,
                'rejected_at' => date('Y-m-d H:i:s')
            ]);
        
        // Notify document owner
        $this->sendRejectionNotification($instance, $step, $comment);
        
        // Execute rejection handler if defined
        if (isset($workflow['rejection_handler'])) {
            $this->executeHandler($workflow['rejection_handler'], $instance);
        }
    }
    
    private function moveToNextStep($instanceId) {
        $instance = $this->getInstance($instanceId);
        $nextStep = $instance['current_step'] + 1;
        
        // Check if workflow is complete
        $steps = json_decode($instance['workflow_id']['steps'], true);
        if (!isset($steps[$nextStep - 1])) {
            $this->completeWorkflow($instanceId);
            return;
        }
        
        // Update instance
        $this->db->where('id', $instanceId)
            ->update('mod_document_workflow_instances', [
                'current_step' => $nextStep
            ]);
        
        // Execute next step
        $this->executeStep($instanceId, $nextStep);
    }
    
    private function completeWorkflow($instanceId) {
        $instance = $this->getInstance($instanceId);
        
        $this->db->where('id', $instanceId)
            ->update('mod_document_workflow_instances', [
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s')
            ]);
        
        // Execute completion handler
        $workflow = $this->getWorkflow($instance['workflow_code']);
        if (isset($workflow['completion_handler'])) {
            $this->executeHandler($workflow['completion_handler'], $instance);
        }
        
        // Update document status
        $this->updateDocumentStatus($instance['document_id'], 'completed');
        
        // Send completion notification
        $this->sendCompletionNotification($instance);
    }
    
    private function resolveAssignee($assignee, $instance) {
        $context = json_decode($instance['context'], true);
        
        switch ($assignee['type']) {
            case 'user':
                return $assignee['id'];
                
            case 'role':
                return $this->getUserByRole($assignee['role'], $context);
                
            case 'department':
                return $this->getUserByDepartment($assignee['department'], $context);
                
            case 'document_owner':
                return $context['owner_id'] ?? null;
                
            case 'previous_approver':
                return $this->getPreviousApprover($instance['id']);
                
            case 'dynamic':
                return $this->resolveDynamicAssignee($assignee['expression'], $context);
                
            default:
                return null;
        }
    }
    
    private function calculateDeadline($step) {
        $hours = $step['deadline_hours'] ?? 48;
        return date('Y-m-d H:i:s', time() + ($hours * 3600));
    }
}
```

### Workflow Definitions
```php
class WorkflowDefinitions {
    // Invoice approval workflow
    public static function invoiceApprovalWorkflow($amountThreshold = 1000) {
        return [
            'code' => 'invoice_approval',
            'name' => 'Invoice Approval Workflow',
            'description' => 'Multi-level approval for large invoices',
            'steps' => [
                [
                    'name' => 'Initial Review',
                    'type' => 'review',
                    'assignee' => ['type' => 'role', 'role' => 'billing_manager'],
                    'deadline_hours' => 24
                ],
                [
                    'name' => 'Finance Approval',
                    'type' => 'approval',
                    'condition' => "amount > {$amountThreshold}",
                    'assignee' => ['type' => 'role', 'role' => 'finance_director'],
                    'deadline_hours' => 48
                ],
                [
                    'name' => 'CEO Approval',
                    'type' => 'approval',
                    'condition' => "amount > 10000",
                    'assignee' => ['type' => 'role', 'role' => 'ceo'],
                    'deadline_hours' => 72
                ],
                [
                    'name' => 'Notification',
                    'type' => 'notification',
                    'template' => 'invoice_approved',
                    'recipients' => ['owner', 'accounting']
                ]
            ],
            'rejection_handler' => 'notify_rejection',
            'completion_handler' => 'finalize_invoice'
        ];
    }
    
    // Document signature workflow
    public static function signatureWorkflow() {
        return [
            'code' => 'document_signature',
            'name' => 'Document Signature Workflow',
            'description' => 'Automated document signing process',
            'steps' => [
                [
                    'name' => 'Prepare Document',
                    'type' => 'transformation',
                    'action' => 'prepare_for_signing',
                    'deadline_hours' => 4
                ],
                [
                    'name' => 'Internal Review',
                    'type' => 'review',
                    'assignee' => ['type' => 'role', 'role' => 'legal'],
                    'deadline_hours' => 48
                ],
                [
                    'name' => 'First Signature',
                    'type' => 'signature',
                    'signer' => ['type' => 'document_owner'],
                    'signature_method' => 'digital',
                    'deadline_hours' => 72
                ],
                [
                    'name' => 'Counter Signature',
                    'type' => 'signature',
                    'signer' => ['type' => 'role', 'role' => 'contract_manager'],
                    'signature_method' => 'digital',
                    'deadline_hours' => 72
                ],
                [
                    'name' => 'Finalize',
                    'type' => 'integration',
                    'action' => 'store_signed_document',
                    'deadline_hours' => 24
                ]
            ]
        ];
    }
    
    // Client onboarding workflow
    public static function clientOnboardingWorkflow() {
        return [
            'code' => 'client_onboarding',
            'name' => 'Client Onboarding Workflow',
            'description' => 'Automated client onboarding process',
            'steps' => [
                [
                    'name' => 'Identity Verification',
                    'type' => 'approval',
                    'assignee' => ['type' => 'role', 'role' => 'compliance'],
                    'deadline_hours' => 24,
                    'action' => 'check_kyc'
                ],
                [
                    'name' => 'Credit Check',
                    'type' => 'approval',
                    'assignee' => ['type' => 'role', 'role' => 'credit_manager'],
                    'deadline_hours' => 48,
                    'action' => 'run_credit_check'
                ],
                [
                    'name' => 'Account Setup',
                    'type' => 'transformation',
                    'action' => 'setup_client_account',
                    'deadline_hours' => 24
                ],
                [
                    'name' => 'Welcome Package',
                    'type' => 'notification',
                    'template' => 'welcome_email',
                    'recipients' => ['client']
                ],
                [
                    'name' => 'Contract Signing',
                    'type' => 'signature',
                    'signer' => ['type' => 'document_owner'],
                    'deadline_hours' => 168 // 1 week
                ],
                [
                    'name' => 'Activate Services',
                    'type' => 'integration',
                    'action' => 'activate_services',
                    'deadline_hours' => 24
                ]
            ]
        ];
    }
}
```

### Document Version Management
```php
class DocumentVersionManager {
    public function createVersion($documentId, $content, $changeNote = null) {
        $version = [
            'document_id' => $documentId,
            'version_number' => $this->getNextVersionNumber($documentId),
            'content' => $content,
            'change_note' => $changeNote,
            'created_by' => $this->getCurrentUserId(),
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return $this->db->insert('mod_document_versions', $version);
    }
    
    public function getVersion($documentId, $versionNumber) {
        return $this->db->select(
            "SELECT * FROM mod_document_versions
             WHERE document_id = ? AND version_number = ?",
            [$documentId, $versionNumber]
        );
    }
    
    public function compareVersions($documentId, $version1, $version2) {
        $v1 = $this->getVersion($documentId, $version1);
        $v2 = $this->getVersion($documentId, $version2);
        
        return [
            'version1' => $version1,
            'version2' => $version2,
            'changes' => $this->computeDiff($v1['content'], $v2['content']),
            'size_diff' => strlen($v2['content']) - strlen($v1['content'])
        ];
    }
    
    public function restoreVersion($documentId, $versionNumber) {
        $version = $this->getVersion($documentId, $versionNumber);
        
        // Create new version from the restoration
        $this->createVersion(
            $documentId,
            $version['content'],
            "Restored from version {$versionNumber}"
        );
        
        // Update current document
        $this->updateDocument($documentId, $version['content']);
        
        return true;
    }
}
```

### Workflow Reporting
```php
class WorkflowReporting {
    public function getWorkflowMetrics($period = '30d') {
        return [
            'overview' => $this->getOverviewMetrics($period),
            'by_workflow' => $this->getMetricsByWorkflow($period),
            'by_step' => $this->getMetricsByStep($period),
            'bottlenecks' => $this->identifyBottlenecks($period),
            'sla_compliance' => $this->getSLACompliance($period)
        ];
    }
    
    private function getOverviewMetrics($period) {
        return $this->db->select(
            "SELECT 
                COUNT(*) as total_instances,
                COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed,
                COUNT(CASE WHEN status = 'active' THEN 1 END) as active,
                COUNT(CASE WHEN status = 'rejected' THEN 1 END) as rejected,
                AVG(TIMESTAMPDIFF(HOUR, started_at, completed_at)) as avg_completion_hours,
                MAX(TIMESTAMPDIFF(HOUR, started_at, completed_at)) as max_completion_hours
             FROM mod_document_workflow_instances
             WHERE started_at > DATE_SUB(NOW(), INTERVAL ?)",
            [$period]
        );
    }
    
    private function identifyBottlenecks($period) {
        return $this->db->select(
            "SELECT 
                step_name,
                COUNT(*) as instances,
                AVG(TIMESTAMPDIFF(HOUR, created_at, decided_at)) as avg_hours,
                COUNT(CASE WHEN TIMESTAMPDIFF(HOUR, created_at, decided_at) > deadline_hours 
                     AND status = 'pending' THEN 1 END) as overdue_count
             FROM mod_document_workflow_steps s
             JOIN mod_document_workflow_instances i ON i.id = s.instance_id
             WHERE s.created_at > DATE_SUB(NOW(), INTERVAL ?)
             GROUP BY step_name
             HAVING avg_hours > 24 OR overdue_count > 0
             ORDER BY avg_hours DESC",
            [$period]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_document_workflows (
    id INT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    steps JSON NOT NULL,
    rejection_handler VARCHAR(255),
    completion_handler VARCHAR(255),
    variables JSON,
    is_active TINYINT(1) DEFAULT 1,
    created_at DATETIME,
    updated_at DATETIME
);

CREATE TABLE mod_document_workflow_instances (
    id INT AUTO_INCREMENT PRIMARY KEY,
    workflow_id INT NOT NULL,
    workflow_code VARCHAR(100) NOT NULL,
    document_id INT NOT NULL,
    document_type VARCHAR(50),
    current_step INT DEFAULT 1,
    status ENUM('active', 'completed', 'rejected', 'cancelled', 'on_hold') DEFAULT 'active',
    context JSON,
    started_by INT,
    started_at DATETIME,
    completed_at DATETIME,
    rejection_reason TEXT,
    INDEX idx_status (status),
    INDEX idx_document (document_id)
);

CREATE TABLE mod_document_workflow_steps (
    id INT AUTO_INCREMENT PRIMARY KEY,
    instance_id INT NOT NULL,
    step_number INT NOT NULL,
    step_name VARCHAR(255),
    step_type VARCHAR(50),
    assigned_to INT,
    status ENUM('pending', 'in_progress', 'approved', 'rejected', 'skipped', 'overdue') DEFAULT 'pending',
    deadline DATETIME,
    started_at DATETIME,
    decided_at DATETIME,
    created_at DATETIME,
    INDEX idx_instance (instance_id),
    INDEX idx_assignee (assigned_to),
    INDEX idx_status (status)
);

CREATE TABLE mod_document_step_approvals (
    id INT AUTO_INCREMENT PRIMARY KEY,
    step_id INT NOT NULL,
    approver_id INT NOT NULL,
    decision ENUM('approved', 'rejected', 'rejected_with_amendments') NOT NULL,
    comment TEXT,
    decided_at DATETIME,
    INDEX idx_step (step_id)
);

CREATE TABLE mod_document_versions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    document_id INT NOT NULL,
    version_number INT NOT NULL,
    content LONGTEXT,
    change_note VARCHAR(255),
    created_by INT,
    created_at DATETIME,
    UNIQUE KEY idx_doc_version (document_id, version_number),
    INDEX idx_document (document_id)
);

CREATE TABLE mod_document_actions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    instance_id INT NOT NULL,
    step_id INT,
    action_type VARCHAR(50),
    action_data JSON,
    executed_by INT,
    executed_at DATETIME,
    result VARCHAR(50),
    error_message TEXT
);
```

## Usage Examples

### Start an Invoice Approval Workflow
```php
$engine = new DocumentWorkflowEngine();
$instanceId = $engine->startWorkflow('invoice_approval', $invoiceId, [
    'user_id' => $currentUserId,
    'owner_id' => $invoiceOwnerId,
    'amount' => $invoiceTotal,
    'client_id' => $clientId
]);
```

### Process Approval
```php
$engine = new DocumentWorkflowEngine();
$engine->processApproval($stepId, 'approved', $approverId, 'Looks good');

// Workflow automatically moves to next step
```

### Check Workflow Status
```php
$workflow = new DocumentWorkflowEngine();
$status = $workflow->getWorkflowStatus($instanceId);

echo "Status: {$status['status']}\n";
echo "Current Step: {$status['current_step_name']}\n";
echo "Assigned to: {$status['assigned_to_name']}\n";
```

### Get Workflow Metrics
```php
$reporting = new WorkflowReporting();
$metrics = $reporting->getWorkflowMetrics('30d');

echo "Completed: {$metrics['overview']['completed']}\n";
echo "Avg Completion Time: {$metrics['overview']['avg_completion_hours']} hours\n";
```

## Integration Points

- WHMCS modules for triggering workflows
- Document storage systems (S3, local, etc.)
- E-signature providers
- Email system for notifications
- Admin dashboard for workflow management
- Webhook integrations for external systems
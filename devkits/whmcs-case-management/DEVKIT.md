# WHMCS Case Management DevKit

## Overview

Case management system for WHMCS enabling tracking and resolution of customer issues, tasks, and workflows with escalation paths.

## Features

- Case creation
- Workflow automation
- Task management
- Escalation paths
- SLA tracking
- Team assignment
- Priority management
- Case analytics

## Module Files

```php
<?php
/**
 * WHMCS Case Management Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/CaseManager.php';

function whmcs_case_management_activate() {
    $manager = new CaseManager();
    return $manager->activate();
}

function whmcs_case_create($data) {
    $manager = new CaseManager();
    return $manager->createCase($data);
}

function whmcs_case_get($caseId) {
    $manager = new CaseManager();
    return $manager->getCase($caseId);
}
```

### lib/CaseManager.php

```php
<?php
namespace WHMCS\Module\CaseManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class CaseManager {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Case Management module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_cases` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `case_number` VARCHAR(50) NOT NULL,
                `case_type` VARCHAR(100) NOT NULL,
                `subject` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `service_id` INT UNSIGNED NULL,
                `priority` ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'medium',
                `status` ENUM('open', 'in_progress', 'pending', 'resolved', 'closed') NOT NULL DEFAULT 'open',
                `assigned_to` INT UNSIGNED NULL,
                `sla_due_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_case_tasks` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `case_id` INT UNSIGNED NOT NULL,
                `task_name` VARCHAR(255) NOT NULL,
                `assigned_to` INT UNSIGNED NULL,
                `due_at` DATETIME NULL,
                `status` ENUM('pending', 'in_progress', 'completed') NOT NULL DEFAULT 'pending',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createCase($data) {
        $caseId = Capsule::table('mod_cases')->insertGetId([
            'case_number' => 'CASE-' . date('Ymd') . '-' . str_pad(Capsule::table('mod_cases')->count() + 1, 4, '0', STR_PAD_LEFT),
            'case_type' => $data['type'],
            'subject' => $data['subject'],
            'description' => $data['description'] ?? null,
            'user_id' => $data['user_id'],
            'priority' => $data['priority'] ?? 'medium',
            'sla_due_at' => $this->calculateSlaDue($data['priority'] ?? 'medium'),
        ]);
        
        return ['success' => true, 'case_id' => $caseId];
    }
    
    protected function calculateSlaDue($priority) {
        $hours = [
            'critical' => 4,
            'high' => 24,
            'medium' => 72,
            'low' => 168,
        ];
        
        return Carbon::now()->addHours($hours[$priority] ?? 72)->toDateTimeString();
    }
    
    public function getCase($caseId) {
        $case = Capsule::table('mod_cases')->where('id', $caseId)->first();
        
        if (!$case) {
            return null;
        }
        
        $tasks = Capsule::table('mod_case_tasks')
            ->where('case_id', $caseId)
            ->get();
        
        return [
            'case' => $case,
            'tasks' => $tasks,
        ];
    }
    
    public function addTask($caseId, $taskData) {
        $id = Capsule::table('mod_case_tasks')->insertGetId([
            'case_id' => $caseId,
            'task_name' => $taskData['name'],
            'assigned_to' => $taskData['assigned_to'] ?? null,
            'due_at' => $taskData['due_at'] ?? null,
        ]);
        
        return ['success' => true, 'task_id' => $id];
    }
}
```

## API Endpoints

```
POST /api/v1/cases                       - Create case
GET  /api/v1/cases/{id}                - Get case
POST /api/v1/cases/{id}/tasks          - Add task
GET  /api/v1/cases                      - List cases
PUT  /api/v1/cases/{id}                - Update case
```

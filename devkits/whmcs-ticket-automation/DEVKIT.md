# WHMCS Ticket Automation Module

```php
<?php
/**
 * WHMCS Ticket Automation Module
 * 
 * Provides automated ticket handling with routing rules,
 * auto-responses, escalation, and SLA management.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function ticketautomation_MetaData() {
    return array('DisplayName' => 'Ticket Automation', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function ticketautomation_ConfigArray() {
    return array('FriendlyName' => array('Type' => 'System', 'Value' => 'Ticket Automation'),
        'EnableRouting' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable ticket routing'),
        'EnableAutoReply' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable auto-replies'),
        'EnableEscalation' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable escalation'),
        'EnableSLA' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable SLA tracking'));
}

function ticketautomation_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_ticketautomation_rules', "
            CREATE TABLE `mod_ticketautomation_rules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `rule_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `rule_type` ENUM('routing', 'auto_reply', 'escalation', 'sla') NOT NULL,
                `conditions` JSON NOT NULL,
                `actions` JSON NOT NULL,
                `priority` INT DEFAULT 0,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_ticketautomation_escalations', "
            CREATE TABLE `mod_ticketautomation_escalations` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `ticket_id` INT NOT NULL,
                `rule_id` INT NULL,
                `level` INT DEFAULT 1,
                `escalated_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `acknowledged` TINYINT(1) DEFAULT 0,
                INDEX `idx_ticket_id` (`ticket_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_ticketautomation_sla', "
            CREATE TABLE `mod_ticketautomation_sla` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `ticket_id` INT NOT NULL,
                `response_due` DATETIME NOT NULL,
                `resolution_due` DATETIME NOT NULL,
                `response_met` DATETIME NULL,
                `resolution_met` DATETIME NULL,
                `status` ENUM('pending', 'met', 'breached', 'cancelled') DEFAULT 'pending',
                INDEX `idx_ticket_id` (`ticket_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Ticket Automation module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function ticketautomation_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function ticketautomation_CreateRule($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $key = 'rule-' . substr(md5(uniqid()), 0, 10);
        Capsule::table('mod_ticketautomation_rules')->insert(array('rule_key' => $key, 'name' => $data['name'], 'rule_type' => $data['rule_type'], 'conditions' => json_encode($data['conditions']), 'actions' => json_encode($data['actions']), 'priority' => $data['priority'] ?? 0));
        return array('success' => true, 'rule_key' => $key);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function ticketautomation_ProcessTicket($ticketId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $ticket = Capsule::table('tbltickets')->where('id', $ticketId)->first();
    if (!$ticket) return array('success' => false, 'error' => 'Ticket not found');
    $rules = Capsule::table('mod_ticketautomation_rules')->where('is_active', 1)->orderBy('priority', 'desc')->get();
    $applied = 0;
    foreach ($rules as $rule) {
        $conditions = json_decode($rule->conditions, true);
        if (ticketautomation_CheckConditions($ticket, $conditions)) {
            $actions = json_decode($rule->actions, true);
            ticketautomation_ExecuteActions($ticketId, $ticket, $actions, $rule);
            $applied++;
        }
    }
    return array('success' => true, 'rules_applied' => $applied);
}

function ticketautomation_CheckConditions($ticket, $conditions) {
    foreach ($conditions as $condition) {
        $field = $condition['field'];
        $operator = $condition['operator'] ?? 'equals';
        $value = $condition['value'];
        $ticketValue = $ticket->$field ?? '';
        switch ($operator) {
            case 'equals': if ($ticketValue != $value) return false; break;
            case 'not_equals': if ($ticketValue == $value) return false; break;
            case 'contains': if (strpos($ticketValue, $value) === false) return false; break;
            case 'starts_with': if (strpos($ticketValue, $value) !== 0) return false; break;
            case 'in': if (!in_array($ticketValue, (array)$value)) return false; break;
        }
    }
    return true;
}

function ticketautomation_ExecuteActions($ticketId, $ticket, $actions, $rule) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    foreach ($actions as $action) {
        switch ($action['type']) {
            case 'set_department':
                Capsule::table('tbltickets')->where('id', $ticketId)->update(array('did' => $action['department_id']));
                break;
            case 'set_priority':
                Capsule::table('tbltickets')->where('id', $ticketId)->update(array('urgency' => $action['priority']));
                break;
            case 'set_status':
                Capsule::table('tbltickets')->where('id', $ticketId)->update(array('status' => $action['status']));
                break;
            case 'assign_to':
                Capsule::table('tbltickets')->where('id', $ticketId)->update(array('adminid' => $action['admin_id']));
                break;
            case 'add_tag':
                Capsule::table('tbltickets')->where('id', $ticketId)->update(array('flags' => ($ticket->flags ?? '') . ',' . $action['tag']));
                break;
            case 'send_auto_reply':
                if ($ticket->status == 'Open') {
                    sendEmail($ticket->email, $action['subject'], $action['message']);
                }
                break;
            case 'escalate':
                Capsule::table('mod_ticketautomation_escalations')->insert(array('ticket_id' => $ticketId, 'rule_id' => $rule->id, 'level' => $action['level'] ?? 1));
                break;
            case 'set_sla':
                $responseHours = $action['response_hours'] ?? 4;
                $resolutionHours = $action['resolution_hours'] ?? 24;
                Capsule::table('mod_ticketautomation_sla')->insert(array('ticket_id' => $ticketId, 'response_due' => date('Y-m-d H:i:s', time() + $responseHours * 3600), 'resolution_due' => date('Y-m-d H:i:s', time() + $resolutionHours * 3600)));
                break;
        }
    }
}

function ticketautomation_CreateSLAPlan($ticketId, $responseHours, $resolutionHours) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_ticketautomation_sla')->insert(array('ticket_id' => $ticketId, 'response_due' => date('Y-m-d H:i:s', time() + $responseHours * 3600), 'resolution_due' => date('Y-m-d H:i:s', time() + $resolutionHours * 3600)));
}

function ticketautomation_GetSLAStatus($ticketId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_ticketautomation_sla')->where('ticket_id', $ticketId)->first();
}

function ticketautomation_UpdateSLAStatus($ticketId, $type) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $field = $type == 'response' ? 'response_met' : 'resolution_met';
    Capsule::table('mod_ticketautomation_sla')->where('ticket_id', $ticketId)->update(array($field => date('Y-m-d H:i:s'), 'status' => 'met'));
}

function ticketautomation_GetBreachedSLAs() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $now = date('Y-m-d H:i:s');
    return Capsule::table('mod_ticketautomation_sla')->where('status', 'pending')->where(function($q) use ($now) { $q->where('response_due', '<', $now)->orWhere('resolution_due', '<', $now); })->get();
}

function ticketautomation_GetEscalations($ticketId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_ticketautomation_escalations')->where('ticket_id', $ticketId)->orderBy('level', 'desc')->get();
}

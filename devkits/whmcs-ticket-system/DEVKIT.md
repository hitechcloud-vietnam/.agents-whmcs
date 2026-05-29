# WHMCS Ticket System - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ticket_system_config() { return ['name' => 'Ticket System', 'description' => 'Enhanced ticket system', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_assign' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-assign tickets']
]];}

function whmcs_ticket_system_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_ticket_custom_fields` (`id` INT(11) NOT NULL AUTO_INCREMENT, `field_name` VARCHAR(255) NOT NULL, `field_type` VARCHAR(50) DEFAULT 'text', `is_required` TINYINT(1) DEFAULT 0, `show_in_client` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_ticket_system_deactivate() { return ['status' => 'success']; }

function whmcs_ticket_system_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Ticket System</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as open FROM tbltickets WHERE status='Open'"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Open Tickets: ' . $stats['open'] . '</p></div></div>';
    
    $recent = select_query('tbltickets', '*', '', 'id', 'DESC', '20');
    echo '<table class="datatable"><thead><tr><th>ID</th><th>Subject</th><th>Status</th><th>Priority</th></tr></thead><tbody>';
    while ($t = mysql_fetch_array($recent)) { echo '<tr><td>#' . $t['id'] . '</td><td>' . substr($t['title'], 0, 40) . '</td><td>' . $t['status'] . '</td><td>' . $t['urgency'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

add_hook('TicketOpen', 1, function($vars) {
    $dept = mysql_fetch_array(select_query('tblticketdepartments', '*', ['id' => $vars['deptid']]));
    return ['department' => $dept, 'auto_assign' => true];
});
```
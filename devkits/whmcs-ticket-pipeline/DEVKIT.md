# WHMCS Ticket Pipeline - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ticket_pipeline_config() { return ['name' => 'Ticket Pipeline', 'description' => 'Visual ticket pipeline', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_ticket_pipeline_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_ticket_stages` (`id` INT(11) NOT NULL AUTO_INCREMENT, `stage_name` VARCHAR(255) NOT NULL, `stage_order` INT(11) DEFAULT 0, `color` VARCHAR(7) DEFAULT '#007bff', `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_ticket_pipeline` (`id` INT(11) NOT NULL AUTO_INCREMENT, `ticket_id` INT(11) NOT NULL, `stage_id` INT(11) NOT NULL, `moved_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    $stages = [['New', '#28a745'], ['In Progress', '#007bff'], ['Waiting', '#ffc107'], ['Resolved', '#6c757d']];
    foreach ($stages as $i => $s) { insert_query('mod_ticket_stages', ['stage_name' => $s[0], 'stage_order' => $i, 'color' => $s[1]]); }
    return ['status' => 'success'];
}

function whmcs_ticket_pipeline_deactivate() { return ['status' => 'success']; }

function whmcs_ticket_pipeline_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Ticket Pipeline</h2>';
    $stages = select_query('mod_ticket_stages', '*', ['is_active' => 1], 'stage_order');
    echo '<div style="display:flex;gap:20px;">';
    while ($s = mysql_fetch_array($stages)) {
        $count = mysql_num_rows(select_query('mod_ticket_pipeline', 'id', ['stage_id' => $s['id']]));
        echo '<div class="panel panel-default" style="flex:1;">
              <div class="panel-heading" style="background:' . $s['color'] . ';color:white;">' . $s['stage_name'] . ' (' . $count . ')</div>
              <div class="panel-body">';
        $tickets = select_query('mod_ticket_pipeline p', 't.id, t.title', ['p.stage_id' => $s['id']], '', '', '', 't.id', 'INNER JOIN tbltickets t ON p.ticket_id = t.id');
        while ($t = mysql_fetch_array($tickets)) { echo '<div style="padding:5px;border-bottom:1px solid #eee;">#' . $t['id'] . ' - ' . substr($t['title'], 0, 20) . '</div>'; }
        echo '</div></div>';
    }
    echo '</div></div>';
}
```
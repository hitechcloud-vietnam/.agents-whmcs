# WHMCS Ticket Reports - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ticket_reports_config() { return ['name' => 'Ticket Reports', 'description' => 'Ticket analytics and reports', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_ticket_reports_activate() { return ['status' => 'success']; }
function whmcs_ticket_reports_deactivate() { return ['status' => 'success']; }

function whmcs_ticket_reports_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Ticket Reports</h2>';
    
    $total = mysql_fetch_array(full_query("SELECT COUNT(*) as count FROM tbltickets"));
    $open = mysql_fetch_array(full_query("SELECT COUNT(*) as count FROM tbltickets WHERE status='Open'"));
    $avgResponse = mysql_fetch_array(full_query("SELECT AVG(TIMESTAMPDIFF(HOUR, date, NOW())) as avg FROM tbltickets"));
    
    echo '<div class="row">
          <div class="col-md-4"><div class="panel panel-success"><div class="panel-heading">Total Tickets</div><div class="panel-body"><h2>' . $total['count'] . '</h2></div></div></div>
          <div class="col-md-4"><div class="panel panel-warning"><div class="panel-heading">Open Tickets</div><div class="panel-body"><h2>' . $open['count'] . '</h2></div></div></div>
          <div class="col-md-4"><div class="panel panel-info"><div class="panel-heading">Avg Response (hrs)</div><div class="panel-body"><h2>' . round($avgResponse['avg'] ?: 0, 1) . '</h2></div></div></div>
          </div>';
    
    $topDepts = full_query("SELECT d.name, COUNT(*) as count FROM tbltickets t INNER JOIN tblticketdepartments d ON t.did = d.id GROUP BY d.id ORDER BY count DESC LIMIT 10");
    echo '<div class="panel panel-default" style="margin-top:20px;"><div class="panel-heading">Top Departments</div><div class="panel-body">
          <table class="datatable"><thead><tr><th>Department</th><th>Tickets</th></tr></thead><tbody>';
    while ($d = mysql_fetch_array($topDepts)) { echo '<tr><td>' . $d['name'] . '</td><td>' . $d['count'] . '</td></tr>'; }
    echo '</tbody></table></div></div>';
}
```
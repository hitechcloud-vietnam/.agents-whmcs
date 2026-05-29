# WHMCS Ticket Feedback - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ticket_feedback_config() { return ['name' => 'Ticket Feedback', 'description' => 'Post-ticket feedback', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_ticket_feedback_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_ticket_feedback` (`id` INT(11) NOT NULL AUTO_INCREMENT, `ticket_id` INT(11) NOT NULL, `rating` TINYINT(1) NOT NULL, `comment` TEXT, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_ticket_feedback_deactivate() { return ['status' => 'success']; }

function whmcs_ticket_feedback_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Ticket Feedback</h2>';
    $avg = mysql_fetch_array(full_query("SELECT AVG(rating) as avg FROM mod_ticket_feedback"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Average Rating: ' . round($avg['avg'] ?: 0, 1) . '/5</p></div></div>';
    
    $feedback = select_query('mod_ticket_feedback', '*', '', 'id', 'DESC', '50');
    echo '<table class="datatable"><thead><tr><th>Ticket</th><th>Rating</th><th>Comment</th><th>Date</th></tr></thead><tbody>';
    while ($f = mysql_fetch_array($feedback)) { echo '<tr><td>#' . $f['ticket_id'] . '</td><td>' . str_repeat('★', $f['rating']) . '</td><td>' . substr($f['comment'], 0, 50) . '</td><td>' . $f['created_at'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}

add_hook('TicketClose', 1, function($vars) { return ['feedback_request' => true]; });
```
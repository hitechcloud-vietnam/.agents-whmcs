# WHMCS Ticket Split - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ticket_split_config() { return ['name' => 'Ticket Split', 'description' => 'Split tickets', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_ticket_split_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_ticket_splits` (`id` INT(11) NOT NULL AUTO_INCREMENT, `original_id` INT(11) NOT NULL, `new_ids` TEXT, `split_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_ticket_split_deactivate() { return ['status' => 'success']; }

function whmcs_ticket_split_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Ticket Split</h2>';
    if ($_POST['split_ticket']) {
        $originalId = (int)$_POST['original_id'];
        $titles = json_decode($_POST['titles'], true);
        $newIds = [];
        foreach ($titles as $title) {
            $result = localAPI('CreateTicket', ['subject' => $title, 'message' => 'Split from ticket #' . $originalId]);
            if ($result['ticketid']) $newIds[] = $result['ticketid'];
        }
        insert_query('mod_ticket_splits', ['original_id' => $originalId, 'new_ids' => implode(',', $newIds)]);
        echo '<div class="alert alert-success">Ticket split into: ' . implode(', ', $newIds) . '</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Split Ticket</div><div class="panel-body">
          <div class="form-group"><label>Original Ticket ID</label><input type="number" name="original_id" class="form-control" required /></div>
          <div class="form-group"><label>New Ticket Subjects (JSON)</label><textarea name="titles" class="form-control" rows="4" placeholder=\'["Subject 1", "Subject 2"]\'></textarea></div>
          <button type="submit" name="split_ticket" class="btn btn-primary">Split</button></div></form>';
}
```
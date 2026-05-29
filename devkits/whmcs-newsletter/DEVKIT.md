# WHMCS Newsletter - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_newsletter_config() { return ['name' => 'Newsletter', 'description' => 'Newsletter management', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_newsletter_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_newsletter_subscribers` (`id` INT(11) NOT NULL AUTO_INCREMENT, `email` VARCHAR(255) NOT NULL UNIQUE, `name` VARCHAR(255), `is_active` TINYINT(1) DEFAULT 1, `subscribed_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_newsletter_templates` (`id` INT(11) NOT NULL AUTO_INCREMENT, `template_name` VARCHAR(255) NOT NULL, `subject` VARCHAR(255), `content` TEXT, `is_default` TINYINT(1) DEFAULT 0, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_newsletter_campaigns` (`id` INT(11) NOT NULL AUTO_INCREMENT, `template_id` INT(11), `subject` VARCHAR(255), `scheduled_at` DATETIME DEFAULT NULL, `sent_at` DATETIME DEFAULT NULL, `recipients` INT(11) DEFAULT 0, `status` ENUM('draft','scheduled','sent') DEFAULT 'draft', PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_newsletter_deactivate() { return ['status' => 'success']; }

function whmcs_newsletter_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Newsletter</h2>';
    $stats = mysql_fetch_array(full_query("SELECT COUNT(*) as subscribers FROM mod_newsletter_subscribers WHERE is_active=1"));
    echo '<div class="panel panel-info"><div class="panel-body"><p>Active Subscribers: ' . $stats['subscribers'] . '</p></div></div>';
    
    if ($_POST['create_campaign']) {
        insert_query('mod_newsletter_campaigns', ['template_id' => $_POST['template_id'], 'subject' => $_POST['subject'], 'scheduled_at' => $_POST['scheduled_at'] ?: null, 'status' => 'scheduled']);
        echo '<div class="alert alert-success">Campaign created!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create Campaign</div><div class="panel-body">
          <div class="form-group"><label>Template</label><select name="template_id" class="form-control">';
    $templates = select_query('mod_newsletter_templates', '*', '', 'template_name');
    while ($t = mysql_fetch_array($templates)) { echo '<option value="' . $t['id'] . '">' . $t['template_name'] . '</option>'; }
    echo '</select></div>
          <div class="form-group"><label>Subject</label><input type="text" name="subject" class="form-control" required /></div>
          <div class="form-group"><label>Schedule (optional)</label><input type="datetime-local" name="scheduled_at" class="form-control" /></div>
          <button type="submit" name="create_campaign" class="btn btn-primary">Create Campaign</button></div></form>';
    
    $campaigns = select_query('mod_newsletter_campaigns', '*', '', 'id', 'DESC', '20');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Subject</th><th>Scheduled</th><th>Sent</th><th>Recipients</th><th>Status</th></tr></thead><tbody>';
    while ($c = mysql_fetch_array($campaigns)) { echo '<tr><td>' . $c['subject'] . '</td><td>' . ($c['scheduled_at'] ?: '-') . '</td><td>' . ($c['sent_at'] ?: '-') . '</td><td>' . $c['recipients'] . '</td><td>' . ucfirst($c['status']) . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```
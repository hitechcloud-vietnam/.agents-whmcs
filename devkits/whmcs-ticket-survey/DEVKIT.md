# WHMCS Ticket Survey - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_ticket_survey_config() { return ['name' => 'Ticket Survey', 'description' => 'Post-ticket surveys', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_ticket_survey_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_survey_questions` (`id` INT(11) NOT NULL AUTO_INCREMENT, `question` VARCHAR(500) NOT NULL, `question_type` VARCHAR(50) DEFAULT 'rating', `options` TEXT, `is_active` TINYINT(1) DEFAULT 1, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_survey_responses` (`id` INT(11) NOT NULL AUTO_INCREMENT, `ticket_id` INT(11) NOT NULL, `question_id` INT(11) NOT NULL, `answer` TEXT, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_ticket_survey_deactivate() { return ['status' => 'success']; }

function whmcs_ticket_survey_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Ticket Survey</h2>';
    if ($_POST['add_question']) {
        insert_query('mod_survey_questions', ['question' => $_POST['question'], 'question_type' => $_POST['type'], 'options' => json_encode($_POST['options'] ?? [])]);
        echo '<div class="alert alert-success">Question added!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Add Survey Question</div><div class="panel-body">
          <div class="form-group"><label>Question</label><input type="text" name="question" class="form-control" required /></div>
          <div class="form-group"><label>Type</label><select name="type" class="form-control"><option value="rating">Rating</option><option value="text">Text</option><option value="multiple">Multiple Choice</option></select></div>
          <button type="submit" name="add_question" class="btn btn-primary">Add</button></div></form>';
    
    $questions = select_query('mod_survey_questions', '*', '', 'id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Question</th><th>Type</th><th>Active</th></tr></thead><tbody>';
    while ($q = mysql_fetch_array($questions)) { echo '<tr><td>' . $q['question'] . '</td><td>' . $q['question_type'] . '</td><td>' . ($q['is_active'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```
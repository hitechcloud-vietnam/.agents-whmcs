# WHMCS Product Questions - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_product_questions_config() {
    return [
        'name' => 'Product Questions',
        'description' => 'Add custom questions to products with conditional logic',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'allow_uploads' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Allow file uploads'],
            'require_answers' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Require answers']
        ]
    ];
}

function whmcs_product_questions_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_product_questions` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `question_text` VARCHAR(500) NOT NULL,
        `question_type` ENUM('text','textarea','select','checkbox','radio','file','date') DEFAULT 'text',
        `options` TEXT,
        `is_required` TINYINT(1) DEFAULT 0,
        `sort_order` INT(11) DEFAULT 0,
        `depends_on` INT(11) DEFAULT NULL,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_question_responses` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `question_id` INT(11) NOT NULL,
        `order_id` INT(11) NOT NULL,
        `response_value` TEXT,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `order_id` (`order_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_product_questions_deactivate() { return ['status' => 'success']; }

function whmcs_product_questions_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Product Questions</h2>';
    
    if ($_POST['add_question']) {
        insert_query('mod_product_questions', [
            'product_id' => $_POST['product_id'],
            'question_text' => $_POST['question_text'],
            'question_type' => $_POST['question_type'],
            'options' => json_encode($_POST['options'] ?? []),
            'is_required' => $_POST['is_required'] ?? 0,
            'sort_order' => $_POST['sort_order'] ?? 0
        ]);
        echo '<div class="alert alert-success">Question added!</div>';
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Add Question</div>
          <div class="panel-body">
          <div class="row">
          <div class="col-md-6"><div class="form-group"><label>Product</label>
          <select name="product_id" class="form-control">';
    $products = select_query("tbld products", "id, name", "", "name");
    while ($p = mysql_fetch_array($products)) {
        echo '<option value="' . $p['id'] . '">' . $p['name'] . '</option>';
    }
    echo '</select></div></div>
          <div class="col-md-6"><div class="form-group"><label>Question Type</label>
          <select name="question_type" class="form-control">
          <option value="text">Text</option>
          <option value="textarea">Textarea</option>
          <option value="select">Dropdown</option>
          <option value="radio">Radio</option>
          <option value="checkbox">Checkbox</option>
          <option value="file">File Upload</option>
          <option value="date">Date</option>
          </select></div></div>
          </div>
          <div class="form-group"><label>Question Text</label>
          <input type="text" name="question_text" class="form-control" required /></div>
          <div class="form-group">
          <label><input type="checkbox" name="is_required" value="1" /> Required</label>
          </div>
          <button type="submit" name="add_question" class="btn btn-primary">Add Question</button>
          </div></form>';
    
    $questions = select_query("mod_product_questions", "*", "", "sort_order");
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>Product</th><th>Question</th><th>Type</th><th>Required</th></tr></thead>
          <tbody>';
    while ($q = mysql_fetch_array($questions)) {
        $p = mysql_fetch_array(select_query("tbld products", "name", ["id" => $q['product_id']]));
        echo '<tr><td>' . ($p['name'] ?? 'N/A') . '</td>
              <td>' . $q['question_text'] . '</td>
              <td>' . $q['question_type'] . '</td>
              <td>' . ($q['is_required'] ? 'Yes' : 'No') . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

add_hook('OrderFormProductDisplay', 1, function($vars) {
    $questions = select_query("mod_product_questions", "*", ["product_id" => $vars['pid'], "is_active" => 1], "sort_order");
    $items = [];
    while ($q = mysql_fetch_array($questions)) {
        $items[] = $q;
    }
    return ['questions' => $items];
});
```
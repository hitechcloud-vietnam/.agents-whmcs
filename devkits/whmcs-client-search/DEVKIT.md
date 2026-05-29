# WHMCS Client Search - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_search_config() {
    return [
        'name' => 'Client Search',
        'description' => 'Advanced client search with filters and saved searches',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'enable_saved' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Enable saved searches'],
            'track_analytics' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Track search analytics']
        ]
    ];
}

function whmcs_client_search_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_saved_searches` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `staff_id` INT(11) DEFAULT NULL,
        `search_name` VARCHAR(255) NOT NULL,
        `search_query` TEXT NOT NULL,
        `filters` TEXT,
        `is_shared` TINYINT(1) DEFAULT 0,
        `use_count` INT(11) DEFAULT 0,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_search_analytics` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `staff_id` INT(11) DEFAULT NULL,
        `search_term` VARCHAR(255) NOT NULL,
        `results_count` INT(11) DEFAULT 0,
        `searched_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_client_search_deactivate() { return ['status' => 'success']; }

function whmcs_client_search_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Search</h2>';
    
    // Save search
    if ($_POST['save_search']) {
        insert_query('mod_saved_searches', [
            'staff_id' => $_SESSION['adminid'],
            'search_name' => $_POST['search_name'],
            'search_query' => $_POST['search_query'],
            'filters' => json_encode($_POST['filters'] ?? [])
        ]);
        echo '<div class="alert alert-success">Search saved!</div>';
    }
    
    // Run search
    $results = [];
    if ($_POST['search_term']) {
        $term = $_POST['search_term'];
        $results = select_query('tblclients', '*', 
            "firstname LIKE '%" . db_escape($term) . "%' OR lastname LIKE '%" . db_escape($term) . "%' OR email LIKE '%" . db_escape($term) . "%' OR companyname LIKE '%" . db_escape($term) . "%'",
            'id', 'DESC', '100');
        
        // Track search
        insert_query('mod_search_analytics', [
            'staff_id' => $_SESSION['adminid'],
            'search_term' => $term,
            'results_count' => mysql_num_rows($results)
        ]);
    }
    
    echo '<form method="post" class="panel panel-default">
          <div class="panel-heading">Search Clients</div>
          <div class="panel-body">
          <div class="input-group">
          <input type="text" name="search_term" class="form-control" placeholder="Search by name, email, or company" />
          <span class="input-group-btn">
          <button type="submit" class="btn btn-primary">Search</button>
          </span>
          </div>
          </div></form>';
    
    if (!empty($results)) {
        echo '<table class="datatable" style="margin-top:20px;">
              <thead><tr><th>Name</th><th>Email</th><th>Company</th><th>Status</th><th>Created</th></tr></thead>
              <tbody>';
        while ($c = mysql_fetch_array($results)) {
            echo '<tr><td>' . $c['firstname'] . ' ' . $c['lastname'] . '</td>
                  <td>' . $c['email'] . '</td>
                  <td>' . $c['companyname'] . '</td>
                  <td>' . $c['status'] . '</td>
                  <td>' . $c['datecreated'] . '</td></tr>';
        }
        echo '</tbody></table>';
        
        echo '<form method="post" style="margin-top:10px;">
              <input type="hidden" name="search_query" value="' . htmlspecialchars($_POST['search_term']) . '" />
              <input type="text" name="search_name" placeholder="Save this search as..." class="form-control" style="display:inline;width:300px;" />
              <button type="submit" name="save_search" class="btn btn-default">Save</button>
              </form>';
    }
    
    // Saved searches
    $saved = select_query('mod_saved_searches', '*', ['staff_id' => $_SESSION['adminid']], 'use_count', 'DESC', '10');
    echo '<div class="panel panel-default" style="margin-top:20px;">
          <div class="panel-heading">Saved Searches</div>
          <div class="panel-body">';
    while ($s = mysql_fetch_array($saved)) {
        echo '<div class="saved-search-item">
              <a href="?search=' . urlencode($s['search_query']) . '">' . $s['search_name'] . '</a>
              <span class="badge">' . $s['use_count'] . ' uses</span>
              </div>';
    }
    echo '</div></div>';
}

function advancedSearch($params) {
    $where = [];
    if (!empty($params['name'])) { $where[] = "(firstname LIKE '%" . db_escape($params['name']) . "%' OR lastname LIKE '%" . db_escape($params['name']) . "%')"; }
    if (!empty($params['email'])) { $where[] = "email LIKE '%" . db_escape($params['email']) . "%'"; }
    if (!empty($params['company'])) { $where[] = "companyname LIKE '%" . db_escape($params['company']) . "%'"; }
    if (!empty($params['status'])) { $where[] = "status = '" . db_escape($params['status']) . "'"; }
    
    $sql = implode(' AND ', $where);
    return select_query('tblclients', '*', $sql ?: '1=1', 'id', 'DESC', '100');
}
```
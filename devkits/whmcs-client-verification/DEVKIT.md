# WHMCS Client Verification - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_verification_config() { return ['name' => 'Client Verification', 'description' => 'Identity verification and KYC', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_client_verification_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_client_verifications` (`id` INT(11) NOT NULL AUTO_INCREMENT, `user_id` INT(11) NOT NULL, `doc_type` VARCHAR(50) NOT NULL, `doc_file` VARCHAR(255), `status` ENUM('pending','approved','rejected') DEFAULT 'pending', `reviewed_by` INT(11) DEFAULT NULL, `reviewed_at` DATETIME DEFAULT NULL, `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`), KEY `user_id` (`user_id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_client_verification_deactivate() { return ['status' => 'success']; }

function whmcs_client_verification_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Verification</h2>';
    if ($_POST['approve'] || $_POST['reject']) {
        $status = $_POST['approve'] ? 'approved' : 'rejected';
        update_query('mod_client_verifications', ['status' => $status, 'reviewed_by' => $_SESSION['adminid'], 'reviewed_at' => date('Y-m-d H:i:s')], ['id' => $_POST['vid']]);
        echo '<div class="alert alert-success">Verification ' . $status . '!</div>';
    }
    $pending = select_query('mod_client_verifications', '*', ['status' => 'pending'], 'created_at', 'DESC');
    echo '<table class="datatable"><thead><tr><th>Client</th><th>Doc Type</th><th>File</th><th>Date</th><th>Actions</th></tr></thead><tbody>';
    while ($v = mysql_fetch_array($pending)) {
        $c = mysql_fetch_array(select_query('tblclients', 'firstname, lastname', ['id' => $v['user_id']]));
        echo '<tr><td>' . $c['firstname'] . ' ' . $c['lastname'] . '</td><td>' . $v['doc_type'] . '</td><td><a href="../' . $v['doc_file'] . '" target="_blank">View</a></td><td>' . $v['created_at'] . '</td>
              <td><form method="post" style="display:inline;"><input type="hidden" name="vid" value="' . $v['id'] . '" /><button name="approve" class="btn btn-xs btn-success">Approve</button> <button name="reject" class="btn btn-xs btn-danger">Reject</button></form></td></tr>';
    }
    echo '</tbody></table></div>';
}
```
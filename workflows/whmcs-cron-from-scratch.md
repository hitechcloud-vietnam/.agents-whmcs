# WHMCS Cron Module From Scratch Workflow

## Description
Create custom cron job modules for WHMCS automated tasks.

## Prerequisites
- WHMCS 7.0+
- PHP 8.1+
- Understanding of scheduled tasks

## Steps

### Step 1: Create Cron Module Directory
```bash
mkdir -p /var/www/whmcs/modules/crons/clicodes_automation
```

### Step 2: Create Cron Module
```php
<?php
/**
 * WHMCS Cron Module - CLICodes Automation
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define cron module info
 */
function clicodes_automation_cron()
{
    return [
        'name' => 'CLICodes Automation',
        'description' => 'Automated tasks for business operations',
        'frequency' => cronInterval(15), // Run every 15 minutes
        ' Runs on CLI only' => true,
        'logging' => true,
    ];
}

/**
 * Execute cron tasks
 */
function clicodes_automation_cron_exec()
{
    $results = [];
    
    // Task 1: Cleanup expired sessions
    $results['sessions'] = clicodes_cleanup_sessions();
    
    // Task 2: Send payment reminders
    $results['reminders'] = clicodes_send_reminders();
    
    // Task 3: Process pending upgrades
    $results['upgrades'] = clicodes_process_upgrades();
    
    // Task 4: Generate usage reports
    $results['reports'] = clicodes_generate_reports();
    
    // Task 5: Check service health
    $results['health'] = clicodes_check_health();
    
    return $results;
}

/**
 * Cleanup expired sessions
 */
function clicodes_cleanup_sessions()
{
    $deleted = Capsule::table('mod_active_sessions')
        ->where('last_activity', '<', date('Y-m-d H:i:s', strtotime('-30 minutes')))
        ->delete();
    
    logActivity("Cleaned up $deleted expired sessions");
    
    return ['deleted' => $deleted];
}

/**
 * Send payment reminders
 */
function clicodes_send_reminders()
{
    // Get invoices due in 3 days
    $invoices = Capsule::table('tblinvoices')
        ->where('status', 'Unpaid')
        ->where('duedate', date('Y-m-d', strtotime('+3 days')))
        ->get();
    
    $sent = 0;
    
    foreach ($invoices as $invoice) {
        // Check if reminder already sent
        $existing = Capsule::table('mod_reminder_log')
            ->where('invoice_id', $invoice->id)
            ->where('reminder_type', 'due_soon')
            ->first();
        
        if (!$existing) {
            // Send reminder email
            sendEmail('InvoiceReminder', $invoice->userid, [
                'invoice_id' => $invoice->id,
                'amount' => $invoice->total,
                'due_date' => $invoice->duedate,
            ]);
            
            // Log reminder
            Capsule::table('mod_reminder_log')->insert([
                'invoice_id' => $invoice->id,
                'reminder_type' => 'due_soon',
                'sent_at' => date('Y-m-d H:i:s'),
            ]);
            
            $sent++;
        }
    }
    
    return ['sent' => $sent];
}

/**
 * Process pending upgrades
 */
function clicodes_process_upgrades()
{
    $pending = Capsule::table('mod_upgrade_queue')
        ->where('status', 'pending')
        ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
        ->get();
    
    $processed = 0;
    
    foreach ($pending as $upgrade) {
        try {
            // Process the upgrade
            $result = clicodes_execute_upgrade($upgrade);
            
            if ($result) {
                Capsule::table('mod_upgrade_queue')
                    ->where('id', $upgrade->id)
                    ->update(['status' => 'completed', 'completed_at' => date('Y-m-d H:i:s')]);
                $processed++;
            }
        } catch (Exception $e) {
            Capsule::table('mod_upgrade_queue')
                ->where('id', $upgrade->id)
                ->update(['status' => 'failed', 'error' => $e->getMessage()]);
            
            logActivity("Upgrade failed: " . $e->getMessage());
        }
    }
    
    return ['processed' => $processed];
}

/**
 * Generate usage reports
 */
function clicodes_generate_reports()
{
    $today = date('Y-m-d');
    
    // Daily stats
    $stats = [
        'new_orders' => Capsule::table('tblorders')
            ->whereDate('date', $today)
            ->count(),
        'new_clients' => Capsule::table('tblclients')
            ->whereDate('created_at', $today)
            ->count(),
        'revenue' => Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereDate('date', $today)
            ->sum('total'),
    ];
    
    // Store in history
    Capsule::table('mod_daily_stats')->insert([
        'date' => $today,
        'data' => json_encode($stats),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Cleanup old records (keep 90 days)
    Capsule::table('mod_daily_stats')
        ->where('date', '<', date('Y-m-d', strtotime('-90 days')))
        ->delete();
    
    return $stats;
}

/**
 * Check service health
 */
function clicodes_check_health()
{
    $alerts = [];
    
    // Check disk space
    $diskUsage = disk_free_space('/') / disk_total_space('/') * 100;
    if ($diskUsage > 90) {
        $alerts[] = ['type' => 'disk', 'message' => "Disk usage at {$diskUsage}%"];
    }
    
    // Check database size
    $dbSize = Capsule::connection()->selectOne(
        "SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size 
         FROM information_schema.tables WHERE table_schema = DATABASE()"
    );
    if ($dbSize && $dbSize->size > 1000) { // > 1GB
        $alerts[] = ['type' => 'database', 'message' => "Database size: {$dbSize->size} MB"];
    }
    
    // Check pending cron jobs
    $lastCron = Capsule::table('tblactivitylog')
        ->where('description', 'LIKE', '%Cron Job%')
        ->orderBy('id', 'desc')
        ->first();
    
    if ($lastCron) {
        $hoursSince = (time() - strtotime($lastCron->date)) / 3600;
        if ($hoursSince > 1) {
            $alerts[] = ['type' => 'cron', 'message' => "Last cron: " . round($hoursSince, 1) . " hours ago"];
        }
    }
    
    // Log alerts
    foreach ($alerts as $alert) {
        logActivity("Health Alert [{$alert['type']}]: {$alert['message']}");
        
        // Send notification if critical
        if ($alert['type'] === 'disk' || $alert['type'] === 'cron') {
            sendAlertEmail($alert);
        }
    }
    
    return [
        'status' => empty($alerts) ? 'healthy' : 'warning',
        'alerts' => $alerts,
    ];
}

/**
 * Execute upgrade helper
 */
function clicodes_execute_upgrade($upgrade)
{
    // Implementation depends on upgrade type
    switch ($upgrade->type) {
        case 'package_change':
            return clicodes_change_package($upgrade->service_id, $upgrade->new_package_id);
        case 'config_change':
            return clicodes_update_config($upgrade->service_id, json_decode($upgrade->config_data, true));
        default:
            return false;
    }
}
```

### Step 3: Register Cron Task
```php
<?php
// Add to WHMCS crons configuration
// Or via admin interface: Configuration > System Settings > Automation Settings
```

### Step 4: Create Database Tables
```php
<?php
function clicodes_automation_activate()
{
    Capsule::schema()->create('mod_active_sessions', function($table) {
        $table->increments('id');
        $table->integer('user_id');
        $table->string('session_key', 64);
        $table->timestamp('last_activity');
    });
    
    Capsule::schema()->create('mod_reminder_log', function($table) {
        $table->increments('id');
        $table->integer('invoice_id');
        $table->string('reminder_type', 50);
        $table->timestamp('sent_at');
    });
    
    Capsule::schema()->create('mod_upgrade_queue', function($table) {
        $table->increments('id');
        $table->integer('service_id');
        $table->string('type', 50);
        $table->text('config_data')->nullable();
        $table->string('status', 20)->default('pending');
        $table->timestamp('scheduled_at');
        $table->timestamp('completed_at')->nullable();
        $table->text('error')->nullable();
    });
    
    Capsule::schema()->create('mod_daily_stats', function($table) {
        $table->increments('id');
        $table->date('date')->unique();
        $table->json('data');
        $table->timestamp('created_at');
    });
    
    return ['status' => 'success'];
}
```

### Step 5: Install and Configure
```bash
# Copy module
cp -r clicodes_automation /var/www/whmcs/modules/crons/

# Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/crons/clicodes_automation
chmod -R 755 /var/www/whmcs/modules/crons/clicodes_automation

# Configure cron in WHMCS admin
# Go to: Configuration > System Settings > Automation Settings
# Add custom cron task
```

## Cron Frequency Options
```php
// Available intervals
cronInterval(5);   // Every 5 minutes
cronInterval(15);  // Every 15 minutes
cronInterval(30);  // Every 30 minutes
cronInterval(60);  // Every hour
cronInterval(360); // Every 6 hours
cronInterval(1440); // Daily
```

## Tags
- cron
- automation
- scheduled-tasks
- development
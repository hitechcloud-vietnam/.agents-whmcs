# WHMCS Data Cleanup Workflow

## Purpose
Clean up and optimize data in WHMCS by removing stale records.

## Prerequisites
- WHMCS installation
- Admin access
- Backup completed

## Step-by-Step Process

### Step 1: Create Data Cleanup Handler

**Create hooks/data_cleanup.php:**
```php
<?php
/**
 * WHMCS Data Cleanup Handler
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataCleanup {
    
    private $deletedCounts = [];
    
    /**
     * Clean up stale data
     */
    public function cleanup($options = []) {
        $defaults = [
            'cleanup_inactive_clients' => true,
            'inactive_days' => 365,
            'cleanup_abandoned_carts' => true,
            'cart_days' => 30,
            'cleanup_expired_tokens' => true,
            'cleanup_spam_tickets' => true,
            'cleanup_temp_files' => true,
            'cleanup_activity_logs' => true,
            'log_days' => 90,
            'dry_run' => false
        ];
        $options = array_merge($defaults, $options);
        
        $results = [];
        
        if ($options['cleanup_inactive_clients']) {
            $results['inactive_clients'] = $this->cleanupInactiveClients(
                $options['inactive_days'],
                $options['dry_run']
            );
        }
        
        if ($options['cleanup_abandoned_carts']) {
            $results['abandoned_carts'] = $this->cleanupAbandonedCarts(
                $options['cart_days'],
                $options['dry_run']
            );
        }
        
        if ($options['cleanup_expired_tokens']) {
            $results['expired_tokens'] = $this->cleanupExpiredTokens($options['dry_run']);
        }
        
        if ($options['cleanup_spam_tickets']) {
            $results['spam_tickets'] = $this->cleanupSpamTickets($options['dry_run']);
        }
        
        if ($options['cleanup_temp_files']) {
            $results['temp_files'] = $this->cleanupTempFiles($options['dry_run']);
        }
        
        if ($options['cleanup_activity_logs']) {
            $results['activity_logs'] = $this->cleanupActivityLogs(
                $options['log_days'],
                $options['dry_run']
            );
        }
        
        return $results;
    }
    
    /**
     * Clean up inactive clients
     */
    private function cleanupInactiveClients($days, $dryRun) {
        $cutoffDate = Carbon::now()->subDays($days)->toDateString();
        
        $query = Capsule::table('tblclients')
            ->where('status', '!=', 'Active')
            ->where('datecreated', '<', $cutoffDate);
        
        // Only archive (don't delete)
        $clients = $query->get(['id', 'email']);
        $count = $clients->count();
        
        if (!$dryRun && $count > 0) {
            $query->update([
                'email' => Capsule::raw("CONCAT('archived_', id, '_', UNIX_TIMESTAMP(), '@cleanup.local')"),
                'status' => 'Closed',
                'notes' => Capsule::raw("CONCAT(COALESCE(notes, ''), ' [Archived by cleanup: " . Carbon::now()->toDateTimeString() . "]')")
            ]);
        }
        
        return [
            'count' => $count,
            'action' => $dryRun ? 'would_archive' : 'archived',
            'cutoff_date' => $cutoffDate
        ];
    }
    
    /**
     * Clean up abandoned carts
     */
    private function cleanupAbandonedCarts($days, $dryRun) {
        $cutoffDate = Carbon::now()->subDays($days)->toDateString();
        
        // Get cart items to delete
        $query = Capsule::table('tblcart')
            ->where('last_updated', '<', $cutoffDate);
        
        $carts = $query->get(['id']);
        $count = $carts->count();
        
        if (!$dryRun && $count > 0) {
            $cartIds = $carts->pluck('id')->toArray();
            
            // Delete cart items
            Capsule::table('tblcartitems')
                ->whereIn('cart_id', $cartIds)
                ->delete();
            
            // Delete carts
            $query->delete();
        }
        
        return [
            'count' => $count,
            'action' => $dryRun ? 'would_delete' : 'deleted'
        ];
    }
    
    /**
     * Clean up expired tokens
     */
    private function cleanupExpiredTokens($dryRun) {
        $now = Carbon::now()->toDateTimeString();
        
        $query = Capsule::table('tblapilog')
            ->where('expiry', '!=', '0000-00-00 00:00:00')
            ->where('expiry', '<', $now);
        
        $count = $query->count();
        
        if (!$dryRun && $count > 0) {
            $query->delete();
        }
        
        // Also clean up admin session tokens
        $sessionQuery = Capsule::table('tbladminlog');
        $sessionCount = $sessionQuery->count();
        
        return [
            'expired_tokens' => $count,
            'action' => $dryRun ? 'would_delete' : 'deleted'
        ];
    }
    
    /**
     * Clean up spam tickets
     */
    private function cleanupSpamTickets($dryRun) {
        // Find tickets marked as spam
        $query = Capsule::table('tbltickets')
            ->where('status', 'Spam');
        
        $count = $query->count();
        
        if (!$dryRun && $count > 0) {
            $ticketIds = $query->pluck('id')->toArray();
            
            // Delete ticket replies
            Capsule::table('tblticketreplies')
                ->whereIn('tid', $ticketIds)
                ->delete();
            
            // Delete ticket attachments
            Capsule::table('tblticketattachments')
                ->whereIn('tid', $ticketIds)
                ->delete();
            
            // Delete tickets
            $query->delete();
        }
        
        return [
            'count' => $count,
            'action' => $dryRun ? 'would_delete' : 'deleted'
        ];
    }
    
    /**
     * Clean up temporary files
     */
    private function cleanupTempFiles($dryRun) {
        $tempDirs = [
            ROOTDIR . '/temp/',
            ROOTDIR . '/downloads/temp/',
            ROOTDIR . '/attachments/temp/'
        ];
        
        $totalDeleted = 0;
        $totalSize = 0;
        
        foreach ($tempDirs as $dir) {
            if (!is_dir($dir)) continue;
            
            $files = glob($dir . '*');
            $cutoff = time() - (24 * 60 * 60); // 24 hours old
            
            foreach ($files as $file) {
                if (is_file($file) && filemtime($file) < $cutoff) {
                    $size = filesize($file);
                    
                    if (!$dryRun) {
                        unlink($file);
                    }
                    
                    $totalDeleted++;
                    $totalSize += $size;
                }
            }
        }
        
        return [
            'files_deleted' => $totalDeleted,
            'size_freed' => $totalSize,
            'action' => $dryRun ? 'would_delete' : 'deleted'
        ];
    }
    
    /**
     * Clean up activity logs
     */
    private function cleanupActivityLogs($days, $dryRun) {
        $cutoffDate = Carbon::now()->subDays($days)->toDateString();
        
        $query = Capsule::table('tblactivitylog')
            ->where('date', '<', $cutoffDate);
        
        $count = $query->count();
        
        if (!$dryRun && $count > 0) {
            $query->delete();
        }
        
        return [
            'count' => $count,
            'action' => $dryRun ? 'would_delete' : 'deleted',
            'cutoff_date' => $cutoffDate
        ];
    }
    
    /**
     * Optimize database tables
     */
    public function optimizeDatabase() {
        $tables = [
            'tblclients', 'tblhosting', 'tbldomains', 'tblinvoices',
            'tblorders', 'tbltickets', 'tblactivitylog'
        ];
        
        $results = [];
        
        foreach ($tables as $table) {
            try {
                Capsule::statement("OPTIMIZE TABLE `{$table}`");
                $results[$table] = ['status' => 'optimized'];
            } catch (Exception $e) {
                $results[$table] = ['status' => 'error', 'message' => $e->getMessage()];
            }
        }
        
        return $results;
    }
    
    /**
     * Get cleanup statistics
     */
    public function getStatistics() {
        return [
            'inactive_clients' => Capsule::table('tblclients')
                ->where('status', '!=', 'Active')
                ->count(),
            'abandoned_carts' => Capsule::table('tblcart')->count(),
            'spam_tickets' => Capsule::table('tbltickets')
                ->where('status', 'Spam')
                ->count(),
            'activity_logs' => Capsule::table('tblactivitylog')->count(),
            'database_size' => $this->getDatabaseSize()
        ];
    }
    
    /**
     * Get database size
     */
    private function getDatabaseSize() {
        try {
            $result = Capsule::select("
                SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) as size_mb
                FROM information_schema.tables
                WHERE table_schema = DATABASE()
            ");
            
            return $result[0]->size_mb ?? 0;
        } catch (Exception $e) {
            return 0;
        }
    }
}
```

### Step 2: Execute Cleanup

```php
<?php
/**
 * Execute data cleanup
 */
$cleanup = new DataCleanup();

// Dry run first
$results = $cleanup->cleanup([
    'dry_run' => true,
    'inactive_days' => 365,
    'cleanup_inactive_clients' => true,
    'cleanup_abandoned_carts' => true,
    'cleanup_activity_logs' => true,
    'log_days' => 90
]);

echo "Data Cleanup Report (Dry Run)\n";
echo "============================\n\n";

foreach ($results as $type => $result) {
    echo ucfirst(str_replace('_', ' ', $type)) . ":\n";
    echo "  Would " . $result['action'] . " " . $result['count'] . " records\n";
    echo "\n";
}

// Execute actual cleanup
if (isset($_GET['confirm']) && $_GET['confirm'] === 'yes') {
    $results = $cleanup->cleanup([
        'dry_run' => false,
        'inactive_days' => 365
    ]);
    
    echo "Cleanup completed.\n";
}
```

## Best Practices
- Always run dry run first
- Backup before cleanup
- Document all deletions
- Set appropriate retention periods
- Review before permanent deletes
- Schedule regular cleanups
- Monitor database size
- Test cleanup procedures
- Keep audit logs
- Verify data integrity

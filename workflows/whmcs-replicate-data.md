# WHMCS Data Replication Workflow

## Purpose
Set up real-time data replication between WHMCS systems.

## Prerequisites
- WHMCS installation
- Database replication access
- Admin access

## Step-by-Step Process

### Step 1: Create Data Replication Handler

**Create hooks/data_replication.php:**
```php
<?php
/**
 * WHMCS Data Replication System
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataReplication {
    
    private $sourceDb;
    private $targetDb;
    private $replicationConfig;
    
    public function __construct($config = []) {
        $this->replicationConfig = $config;
    }
    
    /**
     * Set up replication tables
     */
    public function setupReplicationTables() {
        $tables = [
            'replication_log' => function($table) {
                $table->increments('id');
                $table->string('operation', 20);
                $table->string('table_name', 50);
                $table->integer('record_id');
                $table->json('old_data')->nullable();
                $table->json('new_data')->nullable();
                $table->timestamp('replicated_at')->nullable();
                $table->string('status', 20)->default('pending');
                $table->timestamps();
            },
            'replication_status' => function($table) {
                $table->increments('id');
                $table->string('table_name', 50);
                $table->integer('last_replicated_id');
                $table->timestamp('last_replicated_at');
                $table->timestamps();
            }
        ];
        
        foreach ($tables as $tableName => $callback) {
            if (!Capsule::schema()->hasTable($tableName)) {
                Capsule::schema()->create($tableName, $callback);
            }
        }
    }
    
    /**
     * Log change for replication
     */
    public function logChange($operation, $table, $recordId, $oldData = null, $newData = null) {
        Capsule::table('replication_log')->insert([
            'operation' => $operation,
            'table_name' => $table,
            'record_id' => $recordId,
            'old_data' => $oldData ? json_encode($oldData) : null,
            'new_data' => $newData ? json_encode($newData) : null,
            'status' => 'pending'
        ]);
    }
    
    /**
     * Process pending replications
     */
    public function processReplications($limit = 100) {
        $pending = Capsule::table('replication_log')
            ->where('status', 'pending')
            ->orderBy('id')
            ->limit($limit)
            ->get();
        
        $processed = 0;
        
        foreach ($pending as $log) {
            try {
                $this->replicateChange($log);
                
                Capsule::table('replication_log')
                    ->where('id', $log->id)
                    ->update([
                        'replicated_at' => Carbon::now()->toDateTimeString(),
                        'status' => 'completed'
                    ]);
                
                $processed++;
                
            } catch (Exception $e) {
                Capsule::table('replication_log')
                    ->where('id', $log->id)
                    ->update([
                        'status' => 'failed',
                        'error' => $e->getMessage()
                    ]);
            }
        }
        
        return ['processed' => $processed];
    }
    
    /**
     * Replicate single change
     */
    private function replicateChange($log) {
        if (!$this->targetDb) {
            throw new Exception("Target database not configured");
        }
        
        $table = $log->table_name;
        $data = json_decode($log->new_data, true);
        
        switch ($log->operation) {
            case 'INSERT':
                $this->targetDb->table($table)->insert($data);
                break;
                
            case 'UPDATE':
                if (!empty($data['id'])) {
                    $id = $data['id'];
                    unset($data['id']);
                    $this->targetDb->table($table)->where('id', $id)->update($data);
                }
                break;
                
            case 'DELETE':
                $oldData = json_decode($log->old_data, true);
                if (!empty($oldData['id'])) {
                    $this->targetDb->table($table)->where('id', $oldData['id'])->delete();
                }
                break;
        }
    }
    
    /**
     * Full sync from source
     */
    public function fullSync($tables = []) {
        if (empty($tables)) {
            $tables = ['tblclients', 'tblhosting', 'tbldomains'];
        }
        
        $results = [];
        
        foreach ($tables as $table) {
            $sourceTable = str_replace('tbl', '', $table);
            
            try {
                $count = $this->syncTable($table);
                $results[$table] = ['synced' => $count, 'status' => 'success'];
            } catch (Exception $e) {
                $results[$table] = ['status' => 'error', 'message' => $e->getMessage()];
            }
        }
        
        return $results;
    }
    
    /**
     * Sync single table
     */
    private function syncTable($table) {
        $sourceData = Capsule::table($table)->get();
        $count = 0;
        
        foreach ($sourceData as $row) {
            $data = (array)$row;
            $id = $data['id'];
            unset($data['id']);
            
            $existing = $this->targetDb->table($table)->where('id', $id)->first();
            
            if ($existing) {
                $this->targetDb->table($table)->where('id', $id)->update($data);
            } else {
                $data['id'] = $id;
                $this->targetDb->table($table)->insert($data);
            }
            
            $count++;
        }
        
        return $count;
    }
    
    /**
     * Get replication status
     */
    public function getReplicationStatus() {
        return [
            'pending' => Capsule::table('replication_log')->where('status', 'pending')->count(),
            'completed' => Capsule::table('replication_log')->where('status', 'completed')->count(),
            'failed' => Capsule::table('replication_log')->where('status', 'failed')->count()
        ];
    }
}
```

### Step 2: Execute Replication

```php
<?php
$replication = new DataReplication();

// Setup replication tables
$replication->setupReplicationTables();

// Full sync
$results = $replication->fullSync(['tblclients', 'tblhosting']);
print_r($results);

// Process pending changes
$result = $replication->processReplications(100);
echo "Processed {$result['processed']} changes\n";

// Get status
$status = $replication->getReplicationStatus();
print_r($status);
```

## Best Practices
- Monitor replication lag
- Handle conflicts gracefully
- Test on staging
- Document configuration
- Regular health checks
- Handle failures
- Set up alerts
- Verify data consistency
- Test failover
- Document procedures

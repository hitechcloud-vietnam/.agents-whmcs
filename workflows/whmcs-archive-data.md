# WHMCS Data Archival Workflow

## Purpose
Archive old data from WHMCS for long-term storage while keeping active database lean.

## Prerequisites
- WHMCS installation
- Admin access
- Separate archive database (optional)

## Step-by-Step Process

### Step 1: Create Data Archival Handler

**Create hooks/data_archive.php:**
```php
<?php
/**
 * WHMCS Data Archival System
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataArchive {
    
    private $archiveDb;
    
    public function __construct($archiveConfig = []) {
        if (!empty($archiveConfig)) {
            $this->archiveDb = Capsule::connection('archive');
        }
    }
    
    /**
     * Archive old clients
     */
    public function archiveClients($beforeDate, $options = []) {
        $defaults = [
            'status' => ['Closed', 'Inactive'],
            'archive_services' => true,
            'archive_domains' => true,
            'archive_invoices' => true,
            'archive_tickets' => true,
            'delete_from_main' => false
        ];
        $options = array_merge($defaults, $options);
        
        $query = Capsule::table('tblclients')
            ->where('datecreated', '<', $beforeDate)
            ->whereIn('status', $options['status']);
        
        if (!empty($options['excluded_ids'])) {
            $query->whereNotIn('id', $options['excluded_ids']);
        }
        
        $clients = $query->get();
        $archived = 0;
        
        foreach ($clients as $client) {
            $clientData = (array)$client;
            
            // Archive related data
            if ($options['archive_services']) {
                $this->archiveClientServices($client->id);
            }
            
            if ($options['archive_domains']) {
                $this->archiveClientDomains($client->id);
            }
            
            if ($options['archive_invoices']) {
                $this->archiveClientInvoices($client->id);
            }
            
            if ($options['archive_tickets']) {
                $this->archiveClientTickets($client->id);
            }
            
            // Archive client
            $this->archiveRecord('clients', $clientData);
            
            // Delete from main if requested
            if ($options['delete_from_main']) {
                Capsule::table('tblclients')
                    ->where('id', $client->id)
                    ->delete();
            } else {
                // Mark as archived
                Capsule::table('tblclients')
                    ->where('id', $client->id)
                    ->update([
                        'notes' => Capsule::raw("CONCAT(COALESCE(notes, ''), ' [Archived: " . Carbon::now()->toDateTimeString() . "]')"),
                        'email' => 'archived_' . $client->id . '@whmcs.local'
                    ]);
            }
            
            $archived++;
        }
        
        return [
            'archived' => $archived,
            'before_date' => $beforeDate
        ];
    }
    
    /**
     * Archive client services
     */
    private function archiveClientServices($clientId) {
        $services = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->get();
        
        foreach ($services as $service) {
            $this->archiveRecord('services', (array)$service);
            
            if ($this->archiveDb) {
                $this->archiveDb->table('services')->insert((array)$service);
            }
        }
    }
    
    /**
     * Archive client domains
     */
    private function archiveClientDomains($clientId) {
        $domains = Capsule::table('tbldomains')
            ->where('userid', $clientId)
            ->get();
        
        foreach ($domains as $domain) {
            $this->archiveRecord('domains', (array)$domain);
        }
    }
    
    /**
     * Archive client invoices
     */
    private function archiveClientInvoices($clientId) {
        $invoices = Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->get();
        
        foreach ($invoices as $invoice) {
            $this->archiveRecord('invoices', (array)$invoice);
            
            // Archive invoice items
            $items = Capsule::table('tblinvoiceitems')
                ->where('invoiceid', $invoice->id)
                ->get();
            
            foreach ($items as $item) {
                $this->archiveRecord('invoice_items', (array)$item);
            }
        }
    }
    
    /**
     * Archive client tickets
     */
    private function archiveClientTickets($clientId) {
        $tickets = Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->get();
        
        foreach ($tickets as $ticket) {
            $this->archiveRecord('tickets', (array)$ticket);
            
            // Archive ticket replies
            $replies = Capsule::table('tblticketreplies')
                ->where('tid', $ticket->id)
                ->get();
            
            foreach ($replies as $reply) {
                $this->archiveRecord('ticket_replies', (array)$reply);
            }
        }
    }
    
    /**
     * Archive single record
     */
    private function archiveRecord($type, $data) {
        // Create archive table if not exists
        $this->ensureArchiveTable($type);
        
        $data['archived_at'] = Carbon::now()->toDateTimeString();
        $data['original_table'] = 'tbl' . rtrim($type, 's');
        
        // Insert into archive
        $tableName = 'archive_' . $type;
        
        if ($this->archiveDb) {
            $this->archiveDb->table($tableName)->insert($data);
        } else {
            Capsule::connection()->table($tableName)->insert($data);
        }
    }
    
    /**
     * Ensure archive table exists
     */
    private function ensureArchiveTable($type) {
        $tableName = 'archive_' . $type;
        
        // Create table based on original structure
        $schema = $this->getTableSchema('tbl' . rtrim($type, 's'));
        
        if ($schema && !$this->tableExists($tableName)) {
            Capsule::schema()->create($tableName, function($table) use ($schema) {
                foreach ($schema as $column) {
                    $table->{$column['type']}($column['name']);
                }
                $table->timestamps();
            });
        }
    }
    
    /**
     * Check if table exists
     */
    private function tableExists($tableName) {
        return Capsule::schema()->hasTable($tableName);
    }
    
    /**
     * Get table schema
     */
    private function getTableSchema($tableName) {
        // Simplified - in production would query information_schema
        return null;
    }
    
    /**
     * Restore from archive
     */
    public function restoreFromArchive($clientId, $archiveTable = 'archive_clients') {
        $archivedClient = Capsule::table($archiveTable)
            ->where('id', $clientId)
            ->first();
        
        if (!$archivedClient) {
            throw new Exception("Archived client not found");
        }
        
        $data = (array)$archivedClient;
        unset($data['archived_at'], $data['original_table'], $data['created_at'], $data['updated_at']);
        
        // Restore client
        Capsule::table('tblclients')->insert($data);
        
        // Restore related data
        $this->restoreRelatedData($clientId);
        
        return [
            'success' => true,
            'client_id' => $clientId
        ];
    }
    
    /**
     * Restore related data
     */
    private function restoreRelatedData($clientId) {
        // Restore services
        $services = Capsule::table('archive_services')
            ->where('userid', $clientId)
            ->get();
        
        foreach ($services as $service) {
            $data = (array)$service;
            unset($data['archived_at'], $data['original_table']);
            Capsule::table('tblhosting')->insert($data);
        }
        
        // Similar for other related tables
    }
    
    /**
     * Get archive statistics
     */
    public function getArchiveStats() {
        $tables = ['clients', 'services', 'domains', 'invoices', 'tickets'];
        $stats = [];
        
        foreach ($tables as $table) {
            $archiveTable = 'archive_' . $table;
            
            if ($this->tableExists($archiveTable)) {
                $stats[$table] = [
                    'archived_count' => Capsule::table($archiveTable)->count(),
                    'total_size' => $this->getTableSize($archiveTable)
                ];
            }
        }
        
        return $stats;
    }
    
    /**
     * Get table size
     */
    private function getTableSize($tableName) {
        try {
            $result = Capsule::select("
                SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) as size_mb
                FROM information_schema.tables
                WHERE table_schema = DATABASE()
                AND table_name = ?
            ", [$tableName]);
            
            return $result[0]->size_mb ?? 0;
        } catch (Exception $e) {
            return 0;
        }
    }
}
```

### Step 2: Execute Archive

```php
<?php
/**
 * Execute data archival
 */
$archiver = new DataArchive();

$results = $archiver->archiveClients('2024-01-01', [
    'status' => ['Closed'],
    'delete_from_main' => false,
    'archive_services' => true,
    'archive_domains' => true,
    'archive_invoices' => true
]);

echo "Archival Complete\n";
echo "Archived: " . $results['archived'] . " clients\n";
```

## Best Practices
- Archive before deleting
- Maintain data relationships
- Document archive procedures
- Test restore process
- Set retention policies
- Compress archive data
- Secure archived data
- Monitor archive size
- Schedule regular archives
- Verify archive integrity

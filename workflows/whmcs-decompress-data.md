# WHMCS Data Decompression Workflow

## Purpose
Decompress compressed data in WHMCS.

## Prerequisites
- WHMCS installation
- Compression key available
- Admin access

## Step-by-Step Process

### Step 1: Create Decompression Handler

```php
<?php
/**
 * WHMCS Data Decompression System
 */

class DataDecompression {
    
    private $compression;
    
    public function __construct() {
        $this->compression = new DataCompression();
    }
    
    /**
     * Decompress client data
     */
    public function decompressClientNotes($clientId) {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        
        if (empty($client->compressed_notes)) {
            return $client->notes ?? '';
        }
        
        return $this->compression->decompress($client->compressed_notes);
    }
    
    /**
     * Decompress old logs
     */
    public function decompressOldLogs($limit = 100) {
        $logs = Capsule::table('tblactivitylog')
            ->where('compressed', 1)
            ->limit($limit)
            ->get();
        
        $decompressed = 0;
        
        foreach ($logs as $log) {
            $description = $this->compression->decompress($log->description);
            
            Capsule::table('tblactivitylog')
                ->where('id', $log->id)
                ->update([
                    'description' => $description,
                    'compressed' => 0
                ]);
            
            $decompressed++;
        }
        
        return ['decompressed' => $decompressed];
    }
}
```

### Step 2: Execute Decompression

```php
<?php
$decompression = new DataDecompression();

// Decompress client notes
$notes = $decompression->decompressClientNotes(123);

// Decompress old logs
$result = $decompression->decompressOldLogs(100);
```

## Best Practices
- Selective decompression
- Monitor performance
- Audit access
- Secure handling

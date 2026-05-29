# WHMCS Data Compression Workflow

## Purpose
Compress data and files in WHMCS for storage optimization.

## Prerequisites
- WHMCS installation
- Admin access

## Step-by-Step Process

### Step 1: Create Compression Handler

**Create hooks/data_compression.php:**
```php
<?php
/**
 * WHMCS Data Compression System
 */

class DataCompression {
    
    /**
     * Compress string data
     */
    public function compress($data, $level = 6) {
        if (is_array($data)) {
            $data = json_encode($data);
        }
        
        return base64_encode(gzcompress($data, $level));
    }
    
    /**
     * Decompress string data
     */
    public function decompress($compressedData) {
        $data = base64_decode($compressedData);
        $decompressed = @gzuncompress($data);
        
        if ($decompressed === false) {
            return $data; // Return as-is if not compressed
        }
        
        $json = json_decode($decompressed, true);
        return $json ?? $decompressed;
    }
    
    /**
     * Compress file
     */
    public function compressFile($source, $destination = null) {
        $destination = $destination ?? $source . '.gz';
        
        $fp = gzopen($destination, 'w9');
        gzwrite($fp, file_get_contents($source));
        gzclose($fp);
        
        return $destination;
    }
    
    /**
     * Decompress file
     */
    public function decompressFile($source, $destination = null) {
        $destination = $destination ?? str_replace('.gz', '', $source);
        
        $fp = gzopen($source, 'rb');
        $content = gzread($fp, filesize($source));
        gzclose($fp);
        
        file_put_contents($destination, $content);
        
        return $destination;
    }
    
    /**
     * Compress old activity logs
     */
    public function compressOldLogs($days = 90) {
        $cutoff = date('Y-m-d H:i:s', strtotime("-{$days} days"));
        
        $logs = Capsule::table('tblactivitylog')
            ->where('date', '<', $cutoff)
            ->where('compressed', 0)
            ->limit(1000)
            ->get();
        
        $compressed = 0;
        
        foreach ($logs as $log) {
            $compressedData = $this->compress($log->description);
            
            Capsule::table('tblactivitylog')
                ->where('id', $log->id)
                ->update([
                    'description' => $compressedData,
                    'compressed' => 1
                ]);
            
            $compressed++;
        }
        
        return ['compressed' => $compressed];
    }
}
```

### Step 2: Execute Compression

```php
<?php
$compression = new DataCompression();

// Compress data
$compressed = $compression->compress(['key' => 'value']);

// Decompress data
$data = $compression->decompress($compressed);

// Compress file
$compression->compressFile('/path/to/file.txt');

// Compress old logs
$result = $compression->compressOldLogs(90);
```

## Best Practices
- Test compression ratios
- Monitor performance impact
- Backup before compression
- Document compressed fields
- Regular maintenance

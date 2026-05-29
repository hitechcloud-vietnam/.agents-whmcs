# WHMCS SFTP Sync Configuration Workflow

## Purpose
Configure and manage SFTP-based data synchronization for WHMCS.

## Prerequisites
- WHMCS installation
- SFTP server access
- SSH key or password authentication

## Step-by-Step Process

### Step 1: Create SFTP Sync Handler

**Create hooks/sftp_sync.php:**
```php
<?php
/**
 * WHMCS SFTP Synchronization Handler
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class SFTPSync {
    
    private $config;
    private $connection;
    
    public function __construct($config = []) {
        $defaults = [
            'host' => '',
            'port' => 22,
            'username' => '',
            'password' => '', // or use private key
            'private_key' => '',
            'passphrase' => '',
            'remote_path' => '/whmcs_sync/',
            'local_path' => ROOTDIR . '/temp/sftp_sync/',
            'archive_path' => ROOTDIR . '/archive/sftp_sync/',
            'known_hosts' => ROOTDIR . '/config/known_hosts'
        ];
        $this->config = array_merge($defaults, $config);
        
        $this->ensureDirectories();
    }
    
    /**
     * Connect to SFTP server
     */
    public function connect() {
        if (!function_exists('ssh2_connect')) {
            throw new Exception("SSH2 extension not installed");
        }
        
        $this->connection = ssh2_connect(
            $this->config['host'],
            $this->config['port'],
            ['hostkey' => 'ssh-rsa']
        );
        
        if (!$this->connection) {
            throw new Exception("Cannot connect to SFTP server");
        }
        
        // Authenticate
        if (!empty($this->config['private_key'])) {
            $key = ssh2_auth_pubkey_file(
                $this->connection,
                $this->config['username'],
                $this->config['private_key'] . '.pub',
                $this->config['private_key'],
                $this->config['passphrase']
            );
        } else {
            $key = ssh2_auth_password(
                $this->connection,
                $this->config['username'],
                $this->config['password']
            );
        }
        
        if (!$key) {
            throw new Exception("SFTP authentication failed");
        }
        
        return true;
    }
    
    /**
     * Get SFTP handle
     */
    private function getSftp() {
        return ssh2_sftp($this->connection);
    }
    
    /**
     * List remote files
     */
    public function listRemoteFiles($path = null) {
        $path = $path ?? $this->config['remote_path'];
        $sftp = $this->getSftp();
        
        $files = [];
        $handle = opendir("ssh2.sftp://{$sftp}{$path}");
        
        if (!$handle) {
            return [];
        }
        
        while (($file = readdir($handle)) !== false) {
            if ($file !== '.' && $file !== '..') {
                $files[] = $file;
            }
        }
        
        closedir($handle);
        return $files;
    }
    
    /**
     * Download file from SFTP
     */
    public function downloadFile($remoteFile, $localFile) {
        $sftp = $this->getSftp();
        $remotePath = $this->config['remote_path'] . $remoteFile;
        
        $success = copy(
            "ssh2.sftp://{$sftp}{$remotePath}",
            $localFile
        );
        
        if (!$success) {
            throw new Exception("Failed to download: $remoteFile");
        }
        
        return true;
    }
    
    /**
     * Upload file to SFTP
     */
    public function uploadFile($localFile, $remoteFile = null) {
        $sftp = $this->getSftp();
        $remoteFile = $remoteFile ?? basename($localFile);
        $remotePath = $this->config['remote_path'] . $remoteFile;
        
        $success = copy(
            $localFile,
            "ssh2.sftp://{$sftp}{$remotePath}"
        );
        
        if (!$success) {
            throw new Exception("Failed to upload: $localFile");
        }
        
        return true;
    }
    
    /**
     * Delete remote file
     */
    public function deleteRemoteFile($remoteFile) {
        $sftp = $this->getSftp();
        $remotePath = $this->config['remote_path'] . $remoteFile;
        
        return unlink("ssh2.sftp://{$sftp}{$remotePath}");
    }
    
    /**
     * Ensure local directories exist
     */
    private function ensureDirectories() {
        foreach (['local_path', 'archive_path'] as $dir) {
            if (!is_dir($this->config[$dir])) {
                mkdir($this->config[$dir], 0755, true);
            }
        }
    }
    
    /**
     * Full bidirectional sync
     */
    public function fullSync() {
        $results = [
            'start_time' => Carbon::now()->toDateTimeString(),
            'downloads' => [],
            'uploads' => [],
            'errors' => []
        ];
        
        try {
            $this->connect();
            
            // Download pending files from remote
            $results['downloads'] = $this->processInbound();
            
            // Upload local changes
            $results['uploads'] = $this->processOutbound();
            
        } catch (Exception $e) {
            $results['errors'][] = $e->getMessage();
        } finally {
            $this->disconnect();
        }
        
        $results['end_time'] = Carbon::now()->toDateTimeString();
        
        return $results;
    }
    
    /**
     * Process inbound files (download and import)
     */
    public function processInbound() {
        $remoteFiles = $this->listRemoteFiles();
        $results = [];
        
        foreach ($remoteFiles as $file) {
            $ext = pathinfo($file, PATHINFO_EXTENSION);
            
            if (in_array($ext, ['csv', 'json', 'xml'])) {
                $localFile = $this->config['local_path'] . $file;
                
                try {
                    // Download file
                    $this->downloadFile($file, $localFile);
                    
                    // Process based on type
                    $importResult = $this->processInboundFile($localFile, $ext);
                    
                    // Archive remote file
                    $archiveFile = date('Y-m-d_His') . '_' . $file;
                    $this->renameRemoteFile($file, 'archive/' . $archiveFile);
                    
                    $results[] = [
                        'file' => $file,
                        'status' => 'success',
                        'imported' => $importResult['imported'] ?? 0
                    ];
                    
                } catch (Exception $e) {
                    $results[] = [
                        'file' => $file,
                        'status' => 'error',
                        'error' => $e->getMessage()
                    ];
                }
            }
        }
        
        return $results;
    }
    
    /**
     * Process inbound file based on type
     */
    private function processInboundFile($filePath, $type) {
        switch ($type) {
            case 'csv':
                return $this->processCSVImport($filePath);
            case 'json':
                return $this->processJSONImport($filePath);
            case 'xml':
                return $this->processXMLImport($filePath);
            default:
                return ['imported' => 0];
        }
    }
    
    /**
     * Process CSV import
     */
    private function processCSVImport($filePath) {
        $importer = new CSVImport();
        $csv = $importer->parseCSV($filePath);
        $validation = $importer->validateData($csv['data'], 'clients');
        
        if (!$validation['valid']) {
            throw new Exception(implode(', ', $validation['errors']));
        }
        
        return $importer->importClients($csv['data']);
    }
    
    /**
     * Process JSON import
     */
    private function processJSONImport($filePath) {
        // Implementation similar to CSV
    }
    
    /**
     * Process XML import
     */
    private function processXMLImport($filePath) {
        // Implementation similar to CSV
    }
    
    /**
     * Process outbound files (export and upload)
     */
    public function processOutbound() {
        $results = [];
        $exporter = new DataExport();
        
        // Export and upload clients
        try {
            $export = $exporter->exportClients([
                'format' => 'csv',
                'filename' => 'clients_export_' . date('Y-m-d_His')
            ]);
            
            $this->uploadFile($export['path']);
            unlink($export['path']);
            
            $results[] = [
                'type' => 'clients',
                'file' => basename($export['path']),
                'status' => 'success'
            ];
            
        } catch (Exception $e) {
            $results[] = [
                'type' => 'clients',
                'status' => 'error',
                'error' => $e->getMessage()
            ];
        }
        
        return $results;
    }
    
    /**
     * Rename remote file
     */
    private function renameRemoteFile($from, $to) {
        $sftp = $this->getSftp();
        $fromPath = $this->config['remote_path'] . $from;
        $toPath = $this->config['remote_path'] . $to;
        
        return ssh2_sftp_rename($sftp, $fromPath, $toPath);
    }
    
    /**
     * Disconnect from SFTP
     */
    public function disconnect() {
        $this->connection = null;
    }
}
```

### Step 2: Execute SFTP Sync

```php
<?php
/**
 * Execute SFTP sync
 */
$sftp = new SFTPSync([
    'host' => \WHMCS\Config\Setting::getValue('SftpHost'),
    'port' => \WHMCS\Config\Setting::getValue('SftpPort') ?: 22,
    'username' => \WHMCS\Config\Setting::getValue('SftpUsername'),
    'private_key' => ROOTDIR . '/config/ssh_keys/sftp_private',
    'passphrase' => \WHMCS\Config\Setting::getValue('SftpPassphrase'),
    'remote_path' => '/whmcs_sync/'
]);

$result = $sftp->fullSync();

echo "SFTP Sync Complete\n";
echo "Start: " . $result['start_time'] . "\n";
echo "End: " . $result['end_time'] . "\n";
echo "Downloads: " . count($result['downloads']) . "\n";
echo "Uploads: " . count($result['uploads']) . "\n";
```

### Step 3: SFTP Cron Job

```php
<?php
/**
 * SFTP Sync Cron Job
 */
add_hook('HourlyCronJob', 1, function($vars) {
    $sftp = new SFTPSync([
        'host' => \WHMCS\Config\Setting::getValue('SftpHost'),
        'username' => \WHMCS\Config\Setting::getValue('SftpUsername'),
        'private_key' => ROOTDIR . '/config/ssh_keys/sftp_private'
    ]);
    
    try {
        $results = $sftp->fullSync();
        logActivity('SFTP Sync: ' . json_encode($results));
        return $results;
    } catch (Exception $e) {
        logActivity('SFTP Sync Error: ' . $e->getMessage());
        return ['error' => $e->getMessage()];
    }
});
```

## Best Practices
- Use SSH key authentication
- Protect private keys with passphrases
- Use known_hosts for security
- Implement connection timeouts
- Handle partial transfers
- Log all sync operations
- Archive processed files
- Verify file integrity
- Test sync thoroughly
- Monitor disk space

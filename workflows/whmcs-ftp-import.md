# WHMCS FTP Import Workflow

## Purpose
Import data into WHMCS from FTP/SFTP server drop locations.

## Prerequisites
- WHMCS installation
- FTP/SFTP credentials
- Scheduled task access

## Step-by-Step Process

### Step 1: Create FTP Import Handler

**Create hooks/ftp_import.php:**
```php
<?php
/**
 * WHMCS FTP Import Handler
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class FTPImport {
    
    private $config;
    private $connection;
    
    public function __construct($config = []) {
        $this->config = array_merge([
            'host' => '',
            'port' => 21,
            'username' => '',
            'password' => '',
            'ssl' => false,
            'passive' => true,
            'remote_path' => '/imports/',
            'local_path' => ROOTDIR . '/temp/ftp_import/',
            'archive_path' => ROOTDIR . '/archive/ftp_import/',
            'delete_after_import' => false
        ], $config);
        
        $this->ensureDirectories();
    }
    
    /**
     * Connect to FTP server
     */
    public function connect() {
        if ($this->config['ssl']) {
            $this->connection = ftp_ssl_connect($this->config['host'], $this->config['port']);
        } else {
            $this->connection = ftp_connect($this->config['host'], $this->config['port']);
        }
        
        if (!$this->connection) {
            throw new Exception("Cannot connect to FTP server");
        }
        
        $login = ftp_login($this->connection, $this->config['username'], $this->config['password']);
        
        if (!$login) {
            ftp_close($this->connection);
            throw new Exception("Cannot login to FTP server");
        }
        
        if ($this->config['passive']) {
            ftp_pasv($this->connection, true);
        }
        
        return true;
    }
    
    /**
     * Disconnect from FTP server
     */
    public function disconnect() {
        if ($this->connection) {
            ftp_close($this->connection);
            $this->connection = null;
        }
    }
    
    /**
     * List files in remote directory
     */
    public function listFiles() {
        return ftp_nlist($this->connection, $this->config['remote_path']);
    }
    
    /**
     * Download file from FTP
     */
    public function downloadFile($remoteFile, $localFile) {
        $success = ftp_get($this->connection, $localFile, $remoteFile, FTP_BINARY);
        
        if (!$success) {
            throw new Exception("Failed to download file: $remoteFile");
        }
        
        return true;
    }
    
    /**
     * Process import directory
     */
    public function processImports() {
        $results = [
            'processed' => 0,
            'success' => 0,
            'errors' => []
        ];
        
        try {
            $this->connect();
            
            $files = $this->listFiles();
            
            foreach ($files as $remoteFile) {
                if (pathinfo($remoteFile, PATHINFO_EXTENSION) === 'csv') {
                    $localFile = $this->config['local_path'] . basename($remoteFile);
                    
                    try {
                        $this->downloadFile($remoteFile, $localFile);
                        $importResult = $this->processFile($localFile);
                        
                        if ($importResult['success']) {
                            $results['success']++;
                            
                            // Archive or delete file
                            if ($this->config['delete_after_import']) {
                                ftp_delete($this->connection, $remoteFile);
                            } else {
                                $archiveFile = $this->config['archive_path'] . basename($remoteFile);
                                ftp_rename($this->connection, $remoteFile, $archiveFile);
                            }
                        } else {
                            $results['errors'][] = basename($remoteFile) . ': ' . $importResult['error'];
                        }
                        
                        $results['processed']++;
                        
                    } catch (Exception $e) {
                        $results['errors'][] = basename($remoteFile) . ': ' . $e->getMessage();
                    }
                }
            }
            
        } finally {
            $this->disconnect();
        }
        
        return $results;
    }
    
    /**
     * Process downloaded file
     */
    private function processFile($filePath) {
        $importer = new CSVImport();
        
        try {
            $csv = $importer->parseCSV($filePath);
            $validation = $importer->validateData($csv['data'], 'clients');
            
            if (!$validation['valid']) {
                return [
                    'success' => false,
                    'error' => implode(', ', $validation['errors'])
                ];
            }
            
            $result = $importer->importClients($csv['data'], [
                'update_existing' => true
            ]);
            
            return [
                'success' => true,
                'imported' => $result['imported'],
                'updated' => $result['updated']
            ];
            
        } catch (Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage()
            ];
        }
    }
    
    /**
     * Ensure required directories exist
     */
    private function ensureDirectories() {
        if (!is_dir($this->config['local_path'])) {
            mkdir($this->config['local_path'], 0755, true);
        }
        
        if (!is_dir($this->config['archive_path'])) {
            mkdir($this->config['archive_path'], 0755, true);
        }
    }
    
    /**
     * Upload file to FTP
     */
    public function uploadFile($localFile, $remotePath) {
        $remoteFile = rtrim($this->config['remote_path'], '/') . '/' . basename($remotePath);
        
        $success = ftp_put($this->connection, $remoteFile, $localFile, FTP_BINARY);
        
        if (!$success) {
            throw new Exception("Failed to upload file: $localFile");
        }
        
        return true;
    }
}
```

### Step 2: Execute FTP Import via Cron

```php
<?php
/**
 * FTP Import Cron Job
 */
add_hook('DailyCronJob', 1, function($vars) {
    $ftp = new FTPImport([
        'host' => \WHMCS\Config\Setting::getValue('FtpImportHost'),
        'port' => \WHMCS\Config\Setting::getValue('FtpImportPort') ?: 21,
        'username' => \WHMCS\Config\Setting::getValue('FtpImportUsername'),
        'password' => \WHMCS\Config\Setting::getValue('FtpImportPassword'),
        'ssl' => (bool)\WHMCS\Config\Setting::getValue('FtpImportSSL'),
        'passive' => true,
        'remote_path' => '/imports/',
        'delete_after_import' => false
    ]);
    
    try {
        $results = $ftp->processImports();
        
        logActivity('FTP Import completed: ' . $results['success'] . ' files processed');
        
        return $results;
        
    } catch (Exception $e) {
        logActivity('FTP Import failed: ' . $e->getMessage());
        return ['error' => $e->getMessage()];
    }
});
```

### Step 3: FTP Export (Reverse)

```php
<?php
/**
 * FTP Export Handler
 */
class FTPExport {
    
    private $config;
    private $connection;
    
    public function __construct($config = []) {
        $this->config = array_merge([
            'host' => '',
            'port' => 21,
            'username' => '',
            'password' => '',
            'ssl' => false,
            'passive' => true,
            'remote_path' => '/exports/'
        ], $config);
    }
    
    /**
     * Connect to FTP
     */
    public function connect() {
        // Similar to FTPImport::connect()
    }
    
    /**
     * Export and upload data
     */
    public function exportAndUpload($entityType, $filters = []) {
        $exporter = new DataExport();
        
        // Export data
        switch ($entityType) {
            case 'clients':
                $result = $exporter->exportClients($filters);
                break;
            case 'services':
                $result = $exporter->exportServices($filters);
                break;
            case 'invoices':
                $result = $exporter->exportInvoices($filters);
                break;
            default:
                throw new Exception("Unknown entity type: $entityType");
        }
        
        // Upload to FTP
        $this->connect();
        $remotePath = $this->config['remote_path'] . basename($result['path']);
        $this->uploadFile($result['path'], $remotePath);
        $this->disconnect();
        
        return [
            'success' => true,
            'file' => basename($result['path']),
            'size' => $result['size'],
            'rows' => $result['rows'] ?? 0
        ];
    }
}
```

## Best Practices
- Use FTPS/SFTP for security
- Implement connection timeouts
- Archive processed files
- Validate files before import
- Log all FTP operations
- Handle connection failures
- Use passive mode for firewalls
- Set appropriate permissions
- Monitor FTP storage space
- Document FTP configuration

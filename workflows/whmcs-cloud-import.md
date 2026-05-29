# WHMCS Cloud Storage Import Workflow

## Purpose
Import data into WHMCS from cloud storage services (AWS S3, Google Cloud Storage, Azure Blob).

## Prerequisites
- WHMCS installation
- Cloud storage credentials
- PHP extensions (AWS SDK, etc.)

## Step-by-Step Process

### Step 1: Create Cloud Storage Handler

**Create hooks/cloud_import.php:**
```php
<?php
/**
 * WHMCS Cloud Storage Import Handler
 * Supports AWS S3, Google Cloud Storage, Azure Blob
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class CloudStorageImport {
    
    private $provider;
    private $config;
    private $localPath;
    
    public function __construct($config = []) {
        $this->provider = $config['provider'] ?? 's3';
        $this->config = $config;
        $this->localPath = ROOTDIR . '/temp/cloud_import/';
        
        if (!is_dir($this->localPath)) {
            mkdir($this->localPath, 0755, true);
        }
    }
    
    /**
     * Get S3 client
     */
    private function getS3Client() {
        if (!class_exists('Aws\S3\S3Client')) {
            throw new Exception("AWS SDK not installed");
        }
        
        return new \Aws\S3\S3Client([
            'version' => 'latest',
            'region' => $this->config['region'],
            'credentials' => [
                'key' => $this->config['access_key'],
                'secret' => $this->config['secret_key']
            ]
        ]);
    }
    
    /**
     * Get Google Cloud Storage client
     */
    private function getGCSClient() {
        if (!class_exists('Google\Cloud\Storage\StorageClient')) {
            throw new Exception("Google Cloud SDK not installed");
        }
        
        return new \Google\Cloud\Storage\StorageClient([
            'projectId' => $this->config['project_id'],
            'keyFile' => json_decode($this->config['key_file'], true)
        ]);
    }
    
    /**
     * List objects in bucket
     */
    public function listObjects($prefix = '') {
        switch ($this->provider) {
            case 's3':
                return $this->listS3Objects($prefix);
            case 'gcs':
                return $this->listGCSObjects($prefix);
            case 'azure':
                return $this->listAzureObjects($prefix);
            default:
                throw new Exception("Unknown provider: {$this->provider}");
        }
    }
    
    /**
     * List S3 objects
     */
    private function listS3Objects($prefix) {
        $client = $this->getS3Client();
        
        $result = $client->listObjects([
            'Bucket' => $this->config['bucket'],
            'Prefix' => $prefix
        ]);
        
        return array_map(function($obj) {
            return [
                'key' => $obj['Key'],
                'size' => $obj['Size'],
                'last_modified' => $obj['LastModified']
            ];
        }, $result['Contents'] ?? []);
    }
    
    /**
     * List GCS objects
     */
    private function listGCSObjects($prefix) {
        $client = $this->getGCSClient();
        $bucket = $client->bucket($this->config['bucket']);
        
        $objects = [];
        foreach ($bucket->objects(['prefix' => $prefix]) as $object) {
            $objects[] = [
                'key' => $object->name(),
                'size' => $object->info()['size'],
                'last_modified' => $object->info()['updated']
            ];
        }
        
        return $objects;
    }
    
    /**
     * Download object
     */
    public function downloadObject($key, $localFile = null) {
        $localFile = $localFile ?? $this->localPath . basename($key);
        
        switch ($this->provider) {
            case 's3':
                return $this->downloadFromS3($key, $localFile);
            case 'gcs':
                return $this->downloadFromGCS($key, $localFile);
            case 'azure':
                return $this->downloadFromAzure($key, $localFile);
            default:
                throw new Exception("Unknown provider");
        }
    }
    
    /**
     * Download from S3
     */
    private function downloadFromS3($key, $localFile) {
        $client = $this->getS3Client();
        
        $result = $client->getObject([
            'Bucket' => $this->config['bucket'],
            'Key' => $key,
            'SaveAs' => $localFile
        ]);
        
        return $localFile;
    }
    
    /**
     * Download from GCS
     */
    private function downloadFromGCS($key, $localFile) {
        $client = $this->getGCSClient();
        $bucket = $client->bucket($this->config['bucket']);
        $object = $bucket->object($key);
        $object->downloadToFile($localFile);
        
        return $localFile;
    }
    
    /**
     * Process cloud imports
     */
    public function processImports($prefix = 'imports/') {
        $results = [
            'processed' => 0,
            'success' => 0,
            'errors' => []
        ];
        
        $objects = $this->listObjects($prefix);
        
        foreach ($objects as $object) {
            $key = $object['key'];
            $ext = pathinfo($key, PATHINFO_EXTENSION);
            
            if (!in_array($ext, ['csv', 'json', 'xml'])) {
                continue;
            }
            
            try {
                // Download file
                $localFile = $this->downloadObject($key);
                
                // Process file
                $importResult = $this->processFile($localFile, $ext);
                
                // Archive or delete
                $this->archiveObject($key);
                
                $results['success']++;
                
            } catch (Exception $e) {
                $results['errors'][] = basename($key) . ': ' . $e->getMessage();
            }
            
            $results['processed']++;
        }
        
        return $results;
    }
    
    /**
     * Process downloaded file
     */
    private function processFile($filePath, $type) {
        switch ($type) {
            case 'csv':
                return $this->processCSV($filePath);
            case 'json':
                return $this->processJSON($filePath);
            case 'xml':
                return $this->processXML($filePath);
            default:
                return ['imported' => 0];
        }
    }
    
    /**
     * Process CSV file
     */
    private function processCSV($filePath) {
        $importer = new CSVImport();
        $csv = $importer->parseCSV($filePath);
        return $importer->importClients($csv['data']);
    }
    
    /**
     * Process JSON file
     */
    private function processJSON($filePath) {
        // JSON processing logic
    }
    
    /**
     * Process XML file
     */
    private function processXML($filePath) {
        // XML processing logic
    }
    
    /**
     * Archive processed object
     */
    private function archiveObject($key) {
        $archiveKey = 'archive/' . date('Y-m-d') . '/' . basename($key);
        
        switch ($this->provider) {
            case 's3':
                $this->copyS3Object($key, $archiveKey);
                $this->deleteS3Object($key);
                break;
            case 'gcs':
                $this->copyGCSObject($key, $archiveKey);
                $this->deleteGCSObject($key);
                break;
        }
    }
    
    /**
     * Copy S3 object
     */
    private function copyS3Object($from, $to) {
        $client = $this->getS3Client();
        $client->copyObject([
            'Bucket' => $this->config['bucket'],
            'CopySource' => $this->config['bucket'] . '/' . $from,
            'Key' => $to
        ]);
    }
    
    /**
     * Delete S3 object
     */
    private function deleteS3Object($key) {
        $client = $this->getS3Client();
        $client->deleteObject([
            'Bucket' => $this->config['bucket'],
            'Key' => $key
        ]);
    }
    
    /**
     * Upload to cloud storage
     */
    public function uploadFile($localFile, $remoteKey) {
        switch ($this->provider) {
            case 's3':
                return $this->uploadToS3($localFile, $remoteKey);
            case 'gcs':
                return $this->uploadToGCS($localFile, $remoteKey);
            case 'azure':
                return $this->uploadToAzure($localFile, $remoteKey);
        }
    }
    
    /**
     * Upload to S3
     */
    private function uploadToS3($localFile, $key) {
        $client = $this->getS3Client();
        
        $client->putObject([
            'Bucket' => $this->config['bucket'],
            'Key' => $key,
            'SourceFile' => $localFile,
            'ContentType' => $this->getContentType($key)
        ]);
        
        return true;
    }
    
    /**
     * Get content type
     */
    private function getContentType($key) {
        $ext = pathinfo($key, PATHINFO_EXTENSION);
        $types = [
            'csv' => 'text/csv',
            'json' => 'application/json',
            'xml' => 'application/xml'
        ];
        
        return $types[$ext] ?? 'application/octet-stream';
    }
}
```

### Step 2: Execute Cloud Import

```php
<?php
/**
 * Execute cloud storage import
 */

// AWS S3
$s3 = new CloudStorageImport([
    'provider' => 's3',
    'access_key' => 'your-access-key',
    'secret_key' => 'your-secret-key',
    'region' => 'us-east-1',
    'bucket' => 'your-bucket-name'
]);

$result = $s3->processImports('imports/');

echo "Cloud Import Complete\n";
echo "Processed: " . $result['processed'] . "\n";
echo "Success: " . $result['success'] . "\n";
```

## Best Practices
- Use IAM roles when possible
- Implement least privilege access
- Enable server-side encryption
- Use HTTPS for all operations
- Validate files before import
- Archive processed files
- Monitor storage costs
- Implement retry logic
- Test thoroughly
- Document bucket structure

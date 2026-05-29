# WHMCS Module Auto-Update Workflow

## Description
Implement automatic updates for WHMCS modules.

## Steps

### Step 1: Create Update Checker
```php
<?php
/**
 * Module Update Checker
 */

class CLICodesUpdateChecker
{
    private $currentVersion;
    private $updateServer = 'https://updates.example.com';
    private $moduleId;
    
    public function __construct($moduleId, $currentVersion)
    {
        $this->moduleId = $moduleId;
        $this->currentVersion = $currentVersion;
    }
    
    /**
     * Check for updates
     */
    public function check()
    {
        $response = $this->apiCall('check', [
            'module_id' => $this->moduleId,
            'current_version' => $this->currentVersion,
            'domain' => $_SERVER['HTTP_HOST'],
        ]);
        
        if (version_compare($response['version'], $this->currentVersion, '>')) {
            return [
                'update_available' => true,
                'new_version' => $response['version'],
                'changelog' => $response['changelog'],
                'download_url' => $response['download_url'],
            ];
        }
        
        return ['update_available' => false];
    }
    
    /**
     * Download and install update
     */
    public function update($downloadUrl)
    {
        // Download update
        $zipFile = $this->download($downloadUrl);
        
        if (!$zipFile) {
            return ['success' => false, 'error' => 'Download failed'];
        }
        
        // Extract and install
        $result = $this->install($zipFile);
        
        // Cleanup
        unlink($zipFile);
        
        return $result;
    }
    
    /**
     * Download update package
     */
    private function download($url)
    {
        $tempFile = sys_get_temp_dir() . '/update_' . time() . '.zip';
        
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_FOLLOWLOCATION => true,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 300,
            CURLOPT_FILE => fopen($tempFile, 'w'),
        ]);
        
        $result = curl_exec($ch);
        curl_close($ch);
        
        return $result ? $tempFile : false;
    }
    
    /**
     * Install update
     */
    private function install($zipFile)
    {
        $moduleDir = __DIR__;
        $tempDir = sys_get_temp_dir() . '/whmcs_update_' . time();
        
        // Extract
        $zip = new ZipArchive();
        if ($zip->open($zipFile) === true) {
            $zip->extractTo($tempDir);
            $zip->close();
        } else {
            return ['success' => false, 'error' => 'Extraction failed'];
        }
        
        // Backup current version
        $backupDir = $moduleDir . '_backup_' . date('Ymd');
        rename($moduleDir, $backupDir);
        
        // Copy new files
        $this->copyDirectory($tempDir, $moduleDir);
        
        // Cleanup
        $this->deleteDirectory($tempDir);
        
        // Update version reference
        file_put_contents(
            $moduleDir . '/VERSION',
            $this->newVersion
        );
        
        return [
            'success' => true,
            'backup_dir' => $backupDir,
        ];
    }
    
    private function apiCall($action, $data)
    {
        $ch = curl_init($this->updateServer . '/' . $action);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
}
```

### Step 2: Add Update Hook
```php
<?php
// In module output function

function clicodes_example_output($vars)
{
    $checker = new CLICodesUpdateChecker('clicodes_example', $vars['version']);
    $updateInfo = $checker->check();
    
    $smarty->assign('updateInfo', $updateInfo);
    
    // Show update notification if available
    if ($updateInfo['update_available']) {
        $smarty->assign('showUpdateNotice', true);
    }
}
```

## Update Server Requirements
- Version checking API
- Changelog endpoint
- Secure download URL generation
- License verification

## Tags
- update
- auto-update
- version-management
# WHMCS File Storage

## Overview

File storage management handles file uploads, attachments, and module assets.

## File Upload Handling

```php
<?php
function handleFileUpload(array $file, string $uploadDir): array
{
    // Validate file
    $errors = validateUploadedFile($file);
    
    if (!empty($errors)) {
        return ['success' => false, 'errors' => $errors];
    }
    
    // Generate unique filename
    $extension = pathinfo($file['name'], PATHINFO_EXTENSION);
    $filename = uniqid() . '_' . time() . '.' . $extension;
    
    // Create upload directory if not exists
    if (!is_dir($uploadDir)) {
        mkdir($uploadDir, 0755, true);
    }
    
    // Move file
    $destination = $uploadDir . '/' . $filename;
    
    if (move_uploaded_file($file['tmp_name'], $destination)) {
        return [
            'success' => true,
            'filename' => $filename,
            'path' => $destination,
            'size' => filesize($destination),
        ];
    }
    
    return ['success' => false, 'errors' => ['Failed to move uploaded file']];
}

function validateUploadedFile(array $file): array
{
    $errors = [];
    
    // Check for upload errors
    if ($file['error'] !== UPLOAD_ERR_OK) {
        $uploadErrors = [
            UPLOAD_ERR_INI_SIZE => 'File exceeds upload limit',
            UPLOAD_ERR_FORM_SIZE => 'File exceeds form limit',
            UPLOAD_ERR_PARTIAL => 'File only partially uploaded',
            UPLOAD_ERR_NO_FILE => 'No file uploaded',
            UPLOAD_ERR_NO_TMP_DIR => 'Missing temp folder',
            UPLOAD_ERR_CANT_WRITE => 'Failed to write file',
        ];
        
        $errors[] = $uploadErrors[$file['error']] ?? 'Unknown upload error';
        return $errors;
    }
    
    // Check file size
    $maxSize = 10 * 1024 * 1024; // 10MB
    if ($file['size'] > $maxSize) {
        $errors[] = 'File size exceeds maximum allowed';
    }
    
    // Check file type
    $allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'application/pdf'];
    $finfo = finfo_open(FILEINFO_MIME_TYPE);
    $mimeType = finfo_file($finfo, $file['tmp_name']);
    finfo_close($finfo);
    
    if (!in_array($mimeType, $allowedTypes)) {
        $errors[] = 'File type not allowed';
    }
    
    // Check extension
    $allowedExtensions = ['jpg', 'jpeg', 'png', 'gif', 'pdf'];
    $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
    
    if (!in_array($extension, $allowedExtensions)) {
        $errors[] = 'File extension not allowed';
    }
    
    return $errors;
}
```

## WHMCS Attachment Storage

```php
<?php
class AttachmentManager
{
    private string $storagePath;
    
    public function __construct(string $storagePath = null)
    {
        $this->storagePath = $storagePath ?? ROOTDIR . '/data/attachments';
    }
    
    public function saveTicketAttachment(int $ticketId, array $file): array
    {
        // Validate
        $errors = validateUploadedFile($file);
        if (!empty($errors)) {
            return ['success' => false, 'errors' => $errors];
        }
        
        // Create directory structure
        $dir = $this->storagePath . '/tickets/' . date('Y/m');
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }
        
        // Generate filename
        $extension = pathinfo($file['name'], PATHINFO_EXTENSION);
        $filename = $ticketId . '_' . time() . '_' . bin2hex(random_bytes(8)) . '.' . $extension;
        $path = $dir . '/' . $filename;
        
        // Move file
        if (!move_uploaded_file($file['tmp_name'], $path)) {
            return ['success' => false, 'errors' => ['Failed to save file']];
        }
        
        // Save to database
        $attachmentId = Capsule::table('tblticketattachments')->insertGetId([
            'ticketid' => $ticketId,
            'filename' => $file['name'],
            'filepath' => $path,
            'filetype' => $file['type'],
            'filesize' => $file['size'],
            'uploaded_at' => date('Y-m-d H:i:s'),
        ]);
        
        return [
            'success' => true,
            'attachment_id' => $attachmentId,
            'filename' => $filename,
            'original_name' => $file['name'],
        ];
    }
    
    public function deleteAttachment(int $attachmentId): bool
    {
        $attachment = Capsule::table('tblticketattachments')
            ->where('id', $attachmentId)
            ->first();
        
        if ($attachment && file_exists($attachment->filepath)) {
            unlink($attachment->filepath);
        }
        
        Capsule::table('tblticketattachments')
            ->where('id', $attachmentId)
            ->delete();
        
        return true;
    }
}
```

## Module Assets

```php
<?php
class AssetManager
{
    private string $modulePath;
    
    public function __construct(string $moduleName)
    {
        $this->modulePath = ROOTDIR . '/modules/' . $moduleName;
    }
    
    public function getCssUrl(string $file): string
    {
        return rtrim($this->modulePath, '/') . '/assets/css/' . $file;
    }
    
    public function getJsUrl(string $file): string
    {
        return rtrim($this->modulePath, '/') . '/assets/js/' . $file;
    }
    
    public function getImageUrl(string $file): string
    {
        return rtrim($this->modulePath, '/') . '/assets/images/' . $file;
    }
    
    public function enqueueStyles(string|array $styles): string
    {
        $baseUrl = rtrim(\WHMCS\Config\Setting::getValue('systemurl'), '/');
        $moduleDir = str_replace(ROOTDIR, '', $this->modulePath);
        
        $html = '';
        foreach ((array) $styles as $style) {
            $html .= '<link rel="stylesheet" href="' . $baseUrl . $moduleDir . '/assets/css/' . $style . '">' . "\n";
        }
        
        return $html;
    }
    
    public function enqueueScripts(string|array $scripts): string
    {
        $baseUrl = rtrim(\WHMCS\Config\Setting::getValue('systemurl'), '/');
        $moduleDir = str_replace(ROOTDIR, '', $this->modulePath);
        
        $html = '';
        foreach ((array) $scripts as $script) {
            $html .= '<script src="' . $baseUrl . $moduleDir . '/assets/js/' . $script . '"></script>' . "\n";
        }
        
        return $html;
    }
}

// Usage
$assets = new AssetManager('yourmodule');
echo $assets->enqueueStyles(['main.css', 'custom.css']);
echo $assets->enqueueScripts(['main.js', 'charts.js']);
```

## File Download Handler

```php
<?php
function serveDownload(string $filePath, string $downloadName = null): void
{
    if (!file_exists($filePath)) {
        http_response_code(404);
        exit('File not found');
    }
    
    // Validate path (prevent directory traversal)
    $realPath = realpath($filePath);
    $allowedPath = realpath(ROOTDIR . '/data');
    
    if (strpos($realPath, $allowedPath) !== 0) {
        http_response_code(403);
        exit('Access denied');
    }
    
    // Set headers
    $filename = $downloadName ?? basename($filePath);
    $filesize = filesize($filePath);
    $mimeType = mime_content_type($filePath);
    
    header('Content-Type: ' . $mimeType);
    header('Content-Length: ' . $filesize);
    header('Content-Disposition: attachment; filename="' . $filename . '"');
    header('Cache-Control: no-cache');
    
    // Serve file
    readfile($filePath);
    exit;
}
```

## Best Practices

1. **Validate all uploads** - Check type, size, and content
2. **Use secure paths** - Prevent path traversal
3. **Generate unique names** - Avoid filename conflicts
4. **Store outside webroot** - Keep uploads secure
5. **Clean up regularly** - Remove old files

## Related Documentation

- [WHMCS API File Uploads](/docs/whmcs-api-file-uploads.md)
- [WHMCS Module Security](/docs/whmcs-module-security.md)
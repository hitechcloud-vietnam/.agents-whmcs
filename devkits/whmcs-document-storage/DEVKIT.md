# WHMCS Document Storage DevKit

## Overview

A document storage and management system for WHMCS that enables secure file storage, document organization, version control, access control, search functionality, and integration with workflows.

## Features

- Secure document upload/download
- Folder organization
- Version control
- Access control
- Document search
- File type support
- Storage quotas
- Audit logging
- Template integration
- OCR support (basic)

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_document_folders` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `folder_name` VARCHAR(255) NOT NULL,
    `folder_path` VARCHAR(500) NOT NULL,
    `parent_id` INT UNSIGNED NULL,
    `owner_type` ENUM('user', 'admin', 'system') NOT NULL DEFAULT 'user',
    `owner_id` INT UNSIGNED NULL,
    `is_shared` TINYINT(1) NOT NULL DEFAULT 0,
    `permissions` JSON NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_folder_path` (`folder_path`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_documents` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `document_number` VARCHAR(50) NOT NULL,
    `document_name` VARCHAR(255) NOT NULL,
    `description` TEXT NULL,
    `folder_id` INT UNSIGNED NOT NULL,
    `user_id` INT UNSIGNED NULL,
    `service_id` INT UNSIGNED NULL,
    `document_type` VARCHAR(100) NULL,
    `file_path` VARCHAR(500) NOT NULL,
    `file_name` VARCHAR(255) NOT NULL,
    `file_size` BIGINT UNSIGNED NOT NULL DEFAULT 0,
    `mime_type` VARCHAR(100) NOT NULL,
    `encryption_key` VARCHAR(255) NULL,
    `current_version` INT UNSIGNED NOT NULL DEFAULT 1,
    `is_public` TINYINT(1) NOT NULL DEFAULT 0,
    `access_level` ENUM('private', 'shared', 'organization', 'public') NOT NULL DEFAULT 'private',
    `expires_at` DATETIME NULL,
    `tags` JSON NULL,
    `metadata` JSON NULL,
    `uploaded_by` INT UNSIGNED NOT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_document_number` (`document_number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_document_versions` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `document_id` INT UNSIGNED NOT NULL,
    `version_number` INT UNSIGNED NOT NULL,
    `file_path` VARCHAR(500) NOT NULL,
    `file_size` BIGINT UNSIGNED NOT NULL DEFAULT 0,
    `change_summary` TEXT NULL,
    `uploaded_by` INT UNSIGNED NOT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_document_version` (`document_id`, `version_number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_document_access` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `document_id` INT UNSIGNED NOT NULL,
    `user_id` INT UNSIGNED NULL,
    `admin_id` INT UNSIGNED NULL,
    `access_type` ENUM('read', 'write', 'delete', 'share') NOT NULL DEFAULT 'read',
    `granted_by` INT UNSIGNED NOT NULL,
    `expires_at` DATETIME NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    CONSTRAINT `fk_access_document` FOREIGN KEY (`document_id`) REFERENCES `mod_documents`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Class

```php
<?php
/**
 * WHMCS Document Storage Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/DocumentManager.php';
require_once __DIR__ . '/lib/DocumentStorage.php';
require_once __DIR__ . '/lib/DocumentAccess.php';

function whmcs_document_storage_activate() {
    $manager = new DocumentManager();
    return $manager->activate();
}

function whmcs_document_storage_deactivate() {
    return ['success' => true, 'msg' => 'Document Storage module deactivated'];
}

function whmcs_document_storage_config() {
    return [
        'max_file_size' => [
            'FriendlyName' => 'Max File Size (MB)',
            'Type' => 'text',
            'Default' => '50',
        ],
        'allowed_extensions' => [
            'FriendlyName' => 'Allowed Extensions',
            'Type' => 'text',
            'Default' => 'pdf,doc,docx,xls,xlsx,png,jpg,jpeg',
        ],
        'storage_path' => [
            'FriendlyName' => 'Storage Path',
            'Type' => 'text',
            'Size' => '50',
            'Default' => 'documents',
        ],
        'enable_encryption' => [
            'FriendlyName' => 'Enable File Encryption',
            'Type' => 'yesno',
        ],
    ];
}

function whmcs_document_storage_upload($userId, $folderId, $file) {
    $manager = new DocumentManager();
    return $manager->uploadDocument($userId, $folderId, $file);
}

function whmcs_document_storage_download($documentId, $userId) {
    $storage = new DocumentStorage();
    return $storage->downloadDocument($documentId, $userId);
}

function whmcs_document_storage_list($folderId = null, $userId = null) {
    $manager = new DocumentManager();
    return $manager->listDocuments($folderId, $userId);
}

function whmcs_document_storage_search($query, $filters = []) {
    $manager = new DocumentManager();
    return $manager->searchDocuments($query, $filters);
}
```

### lib/DocumentManager.php

```php
<?php
namespace WHMCS\Module\DocumentStorage;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class DocumentManager {
    
    protected $storageDir = 'documents';
    protected $allowedExtensions = ['pdf', 'doc', 'docx', 'xls', 'xlsx', 'png', 'jpg', 'jpeg', 'txt'];
    protected $maxFileSize = 52428800; // 50MB
    
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultFolders();
            $this->ensureStorageDirectory();
            return ['success' => true, 'msg' => 'Document Storage module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_document_folders` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `folder_name` VARCHAR(255) NOT NULL,
                `folder_path` VARCHAR(500) NOT NULL,
                `parent_id` INT UNSIGNED NULL,
                `owner_type` VARCHAR(20) NOT NULL DEFAULT 'user',
                `owner_id` INT UNSIGNED NULL,
                `is_shared` TINYINT(1) NOT NULL DEFAULT 0,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_folder_path` (`folder_path`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_documents` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `document_number` VARCHAR(50) NOT NULL UNIQUE,
                `document_name` VARCHAR(255) NOT NULL,
                `folder_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NULL,
                `document_type` VARCHAR(100) NULL,
                `file_path` VARCHAR(500) NOT NULL,
                `file_name` VARCHAR(255) NOT NULL,
                `file_size` BIGINT NOT NULL DEFAULT 0,
                `mime_type` VARCHAR(100) NOT NULL,
                `current_version` INT UNSIGNED NOT NULL DEFAULT 1,
                `is_public` TINYINT(1) NOT NULL DEFAULT 0,
                `uploaded_by` INT UNSIGNED NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_document_versions` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `document_id` INT UNSIGNED NOT NULL,
                `version_number` INT UNSIGNED NOT NULL,
                `file_path` VARCHAR(500) NOT NULL,
                `file_size` BIGINT NOT NULL DEFAULT 0,
                `change_summary` TEXT NULL,
                `uploaded_by` INT UNSIGNED NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_document_access` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `document_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NULL,
                `admin_id` INT UNSIGNED NULL,
                `access_type` VARCHAR(20) NOT NULL DEFAULT 'read',
                `granted_by` INT UNSIGNED NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function createDefaultFolders() {
        $folders = [
            ['name' => 'My Documents', 'path' => '/my-documents'],
            ['name' => 'Contracts', 'path' => '/contracts'],
            ['name' => 'Invoices', 'path' => '/invoices'],
            ['name' => 'Support', 'path' => '/support'],
        ];
        
        foreach ($folders as $folder) {
            if (!Capsule::table('mod_document_folders')->where('folder_path', $folder['path'])->exists()) {
                Capsule::table('mod_document_folders')->insert($folder);
            }
        }
    }
    
    protected function ensureStorageDirectory() {
        $path = WHMCS\Application::getInstance()->getRootDir() . '/' . $this->storageDir;
        if (!is_dir($path)) {
            mkdir($path, 0755, true);
        }
    }
    
    protected function generateDocumentNumber() {
        return 'DOC-' . date('Y') . '-' . strtoupper(substr(uniqid(), -8));
    }
    
    public function uploadDocument($userId, $folderId, $file) {
        $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
        
        if (!in_array($extension, $this->allowedExtensions)) {
            return ['success' => false, 'msg' => 'File type not allowed'];
        }
        
        if ($file['size'] > $this->maxFileSize) {
            return ['success' => false, 'msg' => 'File size exceeds maximum allowed'];
        }
        
        $documentNumber = $this->generateDocumentNumber();
        $storagePath = $this->storageDir . '/' . date('Y/m') . '/' . $documentNumber . '.' . $extension;
        
        $fullPath = WHMCS\Application::getInstance()->getRootDir() . '/' . $storagePath;
        $dir = dirname($fullPath);
        
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }
        
        move_uploaded_file($file['tmp_name'], $fullPath);
        
        $documentId = Capsule::table('mod_documents')->insertGetId([
            'document_number' => $documentNumber,
            'document_name' => $file['name'],
            'folder_id' => $folderId,
            'user_id' => $userId,
            'file_path' => $storagePath,
            'file_name' => $file['name'],
            'file_size' => $file['size'],
            'mime_type' => $file['type'],
            'uploaded_by' => $userId,
        ]);
        
        Capsule::table('mod_document_versions')->insert([
            'document_id' => $documentId,
            'version_number' => 1,
            'file_path' => $storagePath,
            'file_size' => $file['size'],
            'uploaded_by' => $userId,
        ]);
        
        return [
            'success' => true,
            'document_id' => $documentId,
            'document_number' => $documentNumber,
        ];
    }
    
    public function listDocuments($folderId = null, $userId = null) {
        $query = Capsule::table('mod_documents as d')
            ->join('mod_document_folders as f', 'd.folder_id', '=', 'f.id');
        
        if ($folderId) {
            $query->where('d.folder_id', $folderId);
        }
        if ($userId) {
            $query->where('d.user_id', $userId);
        }
        
        return $query->select('d.*', 'f.folder_name')
            ->orderBy('d.created_at', 'desc')
            ->get();
    }
    
    public function createFolder($name, $parentId = null, $userId = null) {
        $path = $parentId 
            ? Capsule::table('mod_document_folders')->where('id', $parentId)->value('folder_path')
            : '/';
        
        $fullPath = rtrim($path, '/') . '/' . preg_replace('/[^a-zA-Z0-9\-_]/', '-', $name);
        
        $folderId = Capsule::table('mod_document_folders')->insertGetId([
            'folder_name' => $name,
            'folder_path' => $fullPath,
            'parent_id' => $parentId,
            'owner_type' => 'user',
            'owner_id' => $userId,
        ]);
        
        return ['success' => true, 'folder_id' => $folderId];
    }
    
    public function searchDocuments($query, $filters = []) {
        $searchQuery = Capsule::table('mod_documents as d')
            ->join('mod_document_folders as f', 'd.folder_id', '=', 'f.id')
            ->where(function($q) use ($query) {
                $q->where('d.document_name', 'like', "%{$query}%")
                    ->orWhere('d.description', 'like', "%{$query}%")
                    ->orWhere('d.document_number', 'like', "%{$query}%");
            });
        
        if (!empty($filters['user_id'])) {
            $searchQuery->where('d.user_id', $filters['user_id']);
        }
        if (!empty($filters['document_type'])) {
            $searchQuery->where('d.document_type', $filters['document_type']);
        }
        if (!empty($filters['folder_id'])) {
            $searchQuery->where('d.folder_id', $filters['folder_id']);
        }
        
        return $searchQuery->select('d.*', 'f.folder_name')
            ->orderBy('d.created_at', 'desc')
            ->get();
    }
    
    public function deleteDocument($documentId, $userId) {
        $document = Capsule::table('mod_documents')->where('id', $documentId)->first();
        
        if (!$document) {
            return ['success' => false, 'msg' => 'Document not found'];
        }
        
        if ($document->uploaded_by != $userId) {
            $hasAccess = Capsule::table('mod_document_access')
                ->where('document_id', $documentId)
                ->where('user_id', $userId)
                ->where('access_type', 'delete')
                ->exists();
            
            if (!$hasAccess) {
                return ['success' => false, 'msg' => 'Access denied'];
            }
        }
        
        $fullPath = WHMCS\Application::getInstance()->getRootDir() . '/' . $document->file_path;
        if (file_exists($fullPath)) {
            unlink($fullPath);
        }
        
        Capsule::table('mod_documents')->where('id', $documentId)->delete();
        
        return ['success' => true];
    }
}
```

### lib/DocumentAccess.php

```php
<?php
namespace WHMCS\Module\DocumentStorage;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class DocumentAccess {
    
    public function grantAccess($documentId, $userId, $accessType, $grantedBy) {
        Capsule::table('mod_document_access')->insert([
            'document_id' => $documentId,
            'user_id' => $userId,
            'access_type' => $accessType,
            'granted_by' => $grantedBy,
        ]);
        
        return ['success' => true];
    }
    
    public function revokeAccess($accessId) {
        Capsule::table('mod_document_access')->where('id', $accessId)->delete();
        return ['success' => true];
    }
    
    public function checkAccess($documentId, $userId, $accessType) {
        $document = Capsule::table('mod_documents')->where('id', $documentId)->first();
        
        if ($document->uploaded_by == $userId) {
            return true;
        }
        
        if ($document->user_id == $userId) {
            return true;
        }
        
        if ($document->is_public && $accessType === 'read') {
            return true;
        }
        
        return Capsule::table('mod_document_access')
            ->where('document_id', $documentId)
            ->where('user_id', $userId)
            ->where(function($q) use ($accessType) {
                $q->where('access_type', $accessType)
                    ->orWhere('access_type', 'share');
            })
            ->where(function($q) {
                $q->whereNull('expires_at')
                    ->orWhere('expires_at', '>', Carbon::now());
            })
            ->exists();
    }
    
    public function shareDocument($documentId, $targetUserId, $accessType, $sharedBy) {
        return $this->grantAccess($documentId, $targetUserId, $accessType, $sharedBy);
    }
}
```

## API Endpoints

```
GET  /api/v1/documents                    - List documents
GET  /api/v1/documents/{id}              - Get document details
POST /api/v1/documents                    - Upload document
DELETE /api/v1/documents/{id}           - Delete document
GET  /api/v1/documents/{id}/download     - Download document
GET  /api/v1/documents/{id}/versions     - Get versions
POST /api/v1/documents/{id}/versions     - Upload new version
GET  /api/v1/folders                     - List folders
POST /api/v1/folders                    - Create folder
GET  /api/v1/documents/search            - Search documents
```

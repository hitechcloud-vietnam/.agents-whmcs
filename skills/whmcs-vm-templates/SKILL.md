---
name: whmcs-vm-templates
description: VM template management for WHMCS provisioning
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS VM Template Management Skill

## Overview
This skill provides patterns and implementations for managing VM templates in WHMCS provisioning modules, including template creation, versioning, storage, and deployment workflows.

## Implementation Patterns

### Template Manager Class
```php
<?php
/**
 * WHMCS VM Template Management
 * Handles VM template lifecycle for provisioning
 */

namespace WHMCS\Module\Server\TemplateManager;

class VMTemplateManager {
    private $db;
    private $storagePath;
    private $templateCache = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->storagePath = '/var/lib/whmcs/templates';
    }

    /**
     * Create a new VM template from existing instance
     */
    public function createTemplate(array $params): array {
        $this->validateTemplateParams($params);

        $templateId = $this->generateTemplateId();
        $templateName = $params['name'];
        $sourceVmId = $params['source_vm_id'];
        $hypervisor = $params['hypervisor'];

        // Create template metadata
        $metadata = [
            'id' => $templateId,
            'name' => $templateName,
            'version' => $params['version'] ?? '1.0.0',
            'os_family' => $params['os_family'],
            'os_version' => $params['os_version'],
            'description' => $params['description'] ?? '',
            'size_mb' => 0,
            'checksum' => '',
            'created_at' => date('Y-m-d H:i:s'),
            'created_by' => $params['admin_id'],
            'hypervisor_type' => $hypervisor
        ];

        // Export VM to template based on hypervisor
        $templatePath = $this->exportTemplate($hypervisor, $sourceVmId, $templateId);

        // Calculate size and checksum
        $metadata['size_mb'] = filesize($templatePath) / (1024 * 1024);
        $metadata['checksum'] = hash_file('sha256', $templatePath);

        // Store template metadata
        $this->saveTemplateMetadata($metadata);

        // Upload to template repository if configured
        if (!empty($params['repository_url'])) {
            $this->uploadToRepository($templatePath, $metadata);
        }

        return [
            'success' => true,
            'template_id' => $templateId,
            'path' => $templatePath,
            'size_mb' => round($metadata['size_mb'], 2)
        ];
    }

    /**
     * List available templates with filtering
     */
    public function listTemplates(array $filters = []): array {
        $query = "SELECT * FROM mod_vm_templates WHERE 1=1";

        $bindings = [];

        if (!empty($filters['os_family'])) {
            $query .= " AND os_family = ?";
            $bindings[] = $filters['os_family'];
        }

        if (!empty($filters['hypervisor'])) {
            $query .= " AND hypervisor_type = ?";
            $bindings[] = $filters['hypervisor'];
        }

        if (!empty($filters['search'])) {
            $query .= " AND (name LIKE ? OR description LIKE ?)";
            $bindings[] = '%' . $filters['search'] . '%';
            $bindings[] = '%' . $filters['search'] . '%';
        }

        $query .= " ORDER BY created_at DESC";

        $templates = $this->db->select($query, $bindings);

        return array_map(function($template) {
            return $this->formatTemplateResponse($template);
        }, $templates);
    }

    /**
     * Get template by ID with caching
     */
    public function getTemplate(string $templateId): ?array {
        if (isset($this->templateCache[$templateId])) {
            return $this->templateCache[$templateId];
        }

        $template = $this->db->select(
            "SELECT * FROM mod_vm_templates WHERE id = ?",
            [$templateId]
        );

        if (empty($template)) {
            return null;
        }

        $formatted = $this->formatTemplateResponse($template[0]);
        $this->templateCache[$templateId] = $formatted;

        return $formatted;
    }

    /**
     * Deploy VM from template
     */
    public function deployFromTemplate(array $params): array {
        $template = $this->getTemplate($params['template_id']);

        if (!$template) {
            throw new \Exception("Template not found: {$params['template_id']}");
        }

        $this->validateDeploymentParams($params);

        $vmId = $this->generateVmId();
        $targetHost = $this->selectTargetHost($params);

        // Clone template to target host
        $vmPath = $this->cloneTemplate(
            $template,
            $vmId,
            $targetHost,
            $params
        );

        // Customize VM (hostname, IP, etc.)
        $this->customizeVM($vmId, $params);

        // Start VM
        $this->startVM($targetHost, $vmId);

        return [
            'success' => true,
            'vm_id' => $vmId,
            'vm_path' => $vmPath,
            'target_host' => $targetHost,
            'template_used' => $template['name']
        ];
    }

    /**
     * Update template version
     */
    public function updateTemplate(string $templateId, array $updates): array {
        $template = $this->getTemplate($templateId);

        if (!$template) {
            throw new \Exception("Template not found: {$templateId}");
        }

        $allowedFields = ['name', 'version', 'description', 'is_active'];
        $updateData = [];

        foreach ($allowedFields as $field) {
            if (isset($updates[$field])) {
                $updateData[$field] = $updates[$field];
            }
        }

        $this->db->update(
            'mod_vm_templates',
            $updateData,
            ['id' => $templateId]
        );

        // Clear cache
        unset($this->templateCache[$templateId]);

        return [
            'success' => true,
            'template_id' => $templateId,
            'updated_fields' => array_keys($updateData)
        ];
    }

    /**
     * Delete template with cleanup
     */
    public function deleteTemplate(string $templateId): bool {
        $template = $this->getTemplate($templateId);

        if (!$template) {
            return false;
        }

        // Delete from storage
        if (file_exists($template['storage_path'])) {
            unlink($template['storage_path']);
        }

        // Delete from repository if linked
        if (!empty($template['repository_url'])) {
            $this->deleteFromRepository($template);
        }

        // Remove from database
        $this->db->delete('mod_vm_templates', ['id' => $templateId]);

        // Clear cache
        unset($this->templateCache[$templateId]);

        return true;
    }

    /**
     * Import template from URL or file
     */
    public function importTemplate(string $source, array $metadata): array {
        $templateId = $this->generateTemplateId();

        // Download/extract template
        $tempPath = $this->downloadTemplate($source);

        // Validate template format
        $this->validateTemplateFile($tempPath);

        // Move to storage
        $storagePath = $this->storagePath . '/' . $templateId . '.qcow2';
        rename($tempPath, $storagePath);

        // Create metadata
        $fullMetadata = array_merge($metadata, [
            'id' => $templateId,
            'size_mb' => filesize($storagePath) / (1024 * 1024),
            'checksum' => hash_file('sha256', $storagePath),
            'created_at' => date('Y-m-d H:i:s')
        ]);

        $this->saveTemplateMetadata($fullMetadata);

        return [
            'success' => true,
            'template_id' => $templateId,
            'storage_path' => $storagePath
        ];
    }

    /**
     * Sync templates with remote repository
     */
    public function syncFromRepository(string $repositoryUrl): array {
        $manifest = $this->fetchRepositoryManifest($repositoryUrl);

        $synced = [];
        $failed = [];

        foreach ($manifest['templates'] as $remoteTemplate) {
            try {
                // Check if template exists locally
                $local = $this->db->select(
                    "SELECT id FROM mod_vm_templates WHERE repository_id = ?",
                    [$remoteTemplate['id']]
                );

                if (empty($local)) {
                    // New template - import
                    $result = $this->importTemplate(
                        $repositoryUrl . '/' . $remoteTemplate['file'],
                        $remoteTemplate
                    );
                    $synced[] = $result['template_id'];
                } elseif ($local[0]->checksum !== $remoteTemplate['checksum']) {
                    // Updated - download new version
                    $this->updateTemplateFromRepository($local[0]->id, $remoteTemplate);
                    $synced[] = $local[0]->id;
                }
            } catch (\Exception $e) {
                $failed[] = [
                    'template_id' => $remoteTemplate['id'],
                    'error' => $e->getMessage()
                ];
            }
        }

        return [
            'synced' => count($synced),
            'failed' => count($failed),
            'details' => $synced + $failed
        ];
    }

    // Private helper methods

    private function exportTemplate(string $hypervisor, string $sourceVmId, string $templateId): string {
        switch ($hypervisor) {
            case 'proxmox':
                return $this->exportProxmoxTemplate($sourceVmId, $templateId);
            case 'vmware':
                return $this->exportVMwareTemplate($sourceVmId, $templateId);
            case 'libvirt':
                return $this->exportLibvirtTemplate($sourceVmId, $templateId);
            default:
                throw new \Exception("Unsupported hypervisor: {$hypervisor}");
        }
    }

    private function exportProxmoxTemplate(string $sourceVmId, string $templateId): string {
        $node = $this->getNodeForVM($sourceVmId);
        $storage = $this->getPrimaryStorage();

        $command = "qm template {$sourceVmId} {$storage}:{$templateId}";
        exec($command, $output, $return);

        if ($return !== 0) {
            throw new \Exception("Failed to export Proxmox template: " . implode("\n", $output));
        }

        return "{$storage}:template/{$templateId}";
    }

    private function exportVMwareTemplate(string $sourceVmId, string $templateId): string {
        // Use VMware CLI tools
        $command = "govc vm.clone -vm {$sourceVmId} -template -name template-{$templateId}";
        exec($command, $output, $return);

        return "/datastore/vm-templates/{$templateId}.vmdk";
    }

    private function exportLibvirtTemplate(string $sourceVmId, string $templateId): string {
        $sourcePath = $this->getVMDiskPath($sourceVmId);
        $templatePath = $this->storagePath . "/{$templateId}.qcow2";

        exec("qemu-img convert -O qcow2 {$sourcePath} {$templatePath}");

        return $templatePath;
    }

    private function cloneTemplate(array $template, string $vmId, string $targetHost, array $params): string {
        $sourcePath = $template['storage_path'];

        switch ($template['hypervisor_type']) {
            case 'proxmox':
                $cmd = "qm clone {$template['id']} {$vmId} --target {$targetHost}";
                break;
            case 'vmware':
                $cmd = "govc vm.clone -vm {$template['id']} -host {$targetHost} -name {$vmId}";
                break;
            default:
                $cmd = "cp {$sourcePath} /var/lib/kvm/images/{$vmId}.qcow2";
        }

        exec($cmd, $output, $return);

        return "/var/lib/kvm/images/{$vmId}.qcow2";
    }

    private function customizeVM(string $vmId, array $params): void {
        // Inject custom hostname
        if (!empty($params['hostname'])) {
            $this->setVMHostname($vmId, $params['hostname']);
        }

        // Configure network
        if (!empty($params['ip_address'])) {
            $this->configureVMNetwork($vmId, $params);
        }

        // Run customization scripts
        if (!empty($params['customization_script'])) {
            $this->runCustomizationScript($vmId, $params['customization_script']);
        }
    }

    private function generateTemplateId(): string {
        return 'tmpl_' . bin2hex(random_bytes(12));
    }

    private function generateVmId(): string {
        return 'vm_' . bin2hex(random_bytes(8));
    }

    private function formatTemplateResponse($template): array {
        return [
            'id' => $template->id,
            'name' => $template->name,
            'version' => $template->version,
            'os_family' => $template->os_family,
            'os_version' => $template->os_version,
            'description' => $template->description,
            'size_mb' => round($template->size_mb, 2),
            'checksum' => $template->checksum,
            'hypervisor_type' => $template->hypervisor_type,
            'storage_path' => $template->storage_path,
            'is_active' => (bool) $template->is_active,
            'created_at' => $template->created_at
        ];
    }

    private function validateTemplateParams(array $params): void {
        $required = ['name', 'source_vm_id', 'hypervisor', 'os_family', 'os_version'];
        foreach ($required as $field) {
            if (empty($params[$field])) {
                throw new \Exception("Missing required parameter: {$field}");
            }
        }
    }

    private function validateDeploymentParams(array $params): void {
        $required = ['template_id', 'hostname'];
        foreach ($required as $field) {
            if (empty($params[$field])) {
                throw new \Exception("Missing required parameter: {$field}");
            }
        }
    }
}
```

### Template Storage Management
```php
<?php
/**
 * Template storage handler
 */

class TemplateStorage {
    private $providers = [];
    private $currentProvider;

    public function __construct() {
        $this->providers = [
            'local' => new LocalStorageProvider(),
            's3' => new S3StorageProvider(),
            'nfs' => new NFSStorageProvider()
        ];
        $this->currentProvider = $this->providers['local'];
    }

    public function setProvider(string $type, array $config): void {
        if (!isset($this->providers[$type])) {
            throw new \Exception("Unknown storage provider: {$type}");
        }

        $this->providers[$type]->configure($config);
        $this->currentProvider = $this->providers[$type];
    }

    public function store(string $templateId, string $content): string {
        return $this->currentProvider->store($templateId, $content);
    }

    public function retrieve(string $templateId): string {
        return $this->currentProvider->retrieve($templateId);
    }

    public function delete(string $templateId): bool {
        return $this->currentProvider->delete($templateId);
    }

    public function getUrl(string $templateId): string {
        return $this->currentProvider->getUrl($templateId);
    }
}

class S3StorageProvider {
    private $config;

    public function configure(array $config): void {
        $this->config = $config;
    }

    public function store(string $templateId, string $content): string {
        $s3 = new \Aws\S3\S3Client([
            'region' => $this->config['region'],
            'version' => 'latest',
            'credentials' => [
                'key' => $this->config['access_key'],
                'secret' => $this->config['secret_key']
            ]
        ]);

        $result = $s3->putObject([
            'Bucket' => $this->config['bucket'],
            'Key' => "templates/{$templateId}",
            'Body' => $content
        ]);

        return $result['ObjectURL'];
    }

    public function retrieve(string $templateId): string {
        $s3 = new \Aws\S3\S3Client($this->getClientConfig());

        $result = $s3->getObject([
            'Bucket' => $this->config['bucket'],
            'Key' => "templates/{$templateId}"
        ]);

        return $result['Body']->getContents();
    }

    public function getUrl(string $templateId): string {
        return "https://{$this->config['bucket']}.s3.amazonaws.com/templates/{$templateId}";
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_vm_templates` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(255) NOT NULL,
  `version` VARCHAR(20) NOT NULL DEFAULT '1.0.0',
  `os_family` VARCHAR(50) NOT NULL,
  `os_version` VARCHAR(100) NOT NULL,
  `description` TEXT,
  `storage_path` VARCHAR(500) NOT NULL,
  `size_mb` BIGINT NOT NULL DEFAULT 0,
  `checksum` VARCHAR(64) NOT NULL,
  `hypervisor_type` VARCHAR(50) NOT NULL,
  `repository_url` VARCHAR(500),
  `repository_id` VARCHAR(100),
  `is_active` TINYINT(1) NOT NULL DEFAULT 1,
  `created_at` DATETIME NOT NULL,
  `updated_at` DATETIME NOT NULL,
  INDEX `idx_os_family` (`os_family`),
  INDEX `idx_hypervisor` (`hypervisor_type`)
);

CREATE TABLE `mod_template_versions` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `template_id` VARCHAR(50) NOT NULL,
  `version` VARCHAR(20) NOT NULL,
  `storage_path` VARCHAR(500) NOT NULL,
  `size_mb` BIGINT NOT NULL DEFAULT 0,
  `checksum` VARCHAR(64) NOT NULL,
  `created_at` DATETIME NOT NULL,
  FOREIGN KEY (`template_id`) REFERENCES `mod_vm_templates`(`id`)
);
```

## Best Practices

1. **Template Versioning**: Maintain multiple versions for rollback capability
2. **Checksum Verification**: Always verify template integrity after transfer
3. **Storage Optimization**: Use compression for large templates
4. **Template Pruning**: Remove old unused templates periodically
5. **Access Control**: Restrict template management to authorized admins

## Related Skills

- whmcs-cloud-init
- whmcs-snapshot-management
- whmcs-backup-scheduling
- whmcs-provisioning-master
---
name: whmcs-gpu-provisioning
description: GPU instance provisioning for WHMCS
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS GPU Provisioning Skill

## Overview
This skill provides patterns and implementations for provisioning GPU instances in WHMCS, including GPU type selection, driver management, licensing, and performance optimization.

## Implementation Patterns

### GPU Provisioning Manager
```php
<?php
/**
 * WHMCS GPU Instance Provisioning
 * Handles GPU-enabled VM provisioning
 */

namespace WHMCS\Module\Server\GPU;

class GPUProvisioningManager {
    private $db;
    private $driverManager;
    private $licenseManager;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->driverManager = new GPUDriverManager();
        $this->licenseManager = new GPULicenseManager();
    }

    /**
     * Provision GPU instance
     */
    public function provisionGPUInstance(array $params): array {
        $gpuId = 'gpu_' . bin2hex(random_bytes(12));

        $gpuInstance = [
            'id' => $gpuId,
            'service_id' => $params['service_id'],
            'gpu_type' => $params['gpu_type'],
            'gpu_count' => $params['gpu_count'] ?? 1,
            'vram_gb' => $params['vram_gb'],
            'driver_version' => $params['driver_version'] ?? 'latest',
            'cuda_version' => $params['cuda_version'] ?? '12.0',
            'provisioning_status' => 'preparing',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_gpu_instances', $gpuInstance);

        // Check GPU availability
        $availability = $this->checkGPUAvailability($params['gpu_type'], $params['gpu_count']);

        if (!$availability['available']) {
            throw new \Exception("GPU not available: {$params['gpu_type']}");
        }

        // Reserve GPUs
        $reservationId = $this->reserveGPUs($params['gpu_type'], $params['gpu_count']);

        // Create VM with GPU passthrough
        $vmId = $this->createGPUVM($gpuInstance, $params);

        // Install drivers
        $this->installGPUDrivers($vmId, $params);

        // Install CUDA/cuDNN
        $this->installCUDAPackages($vmId, $params);

        // Verify GPU access
        $verification = $this->verifyGPUAccess($vmId);

        $this->db->update('mod_gpu_instances', [
            'vm_id' => $vmId,
            'reservation_id' => $reservationId,
            'provisioning_status' => $verification['success'] ? 'ready' : 'failed',
            'ready_at' => $verification['success'] ? date('Y-m-d H:i:s') : null
        ], ['id' => $gpuId]);

        return [
            'success' => $verification['success'],
            'gpu_id' => $gpuId,
            'vm_id' => $vmId,
            'gpu_type' => $params['gpu_type'],
            'gpu_count' => $params['gpu_count'],
            'verification' => $verification
        ];
    }

    /**
     * Get available GPU types
     */
    public function getAvailableGPUTypes(): array {
        $gpuTypes = $this->db->select(
            "SELECT * FROM mod_gpu_types WHERE available = 1 ORDER BY price_per_hour ASC"
        );

        return array_map(function($type) {
            return [
                'id' => $type->id,
                'name' => $type->name,
                'vram_gb' => $type->vram_gb,
                'cores' => $type->cuda_cores,
                'memory_bandwidth' => $type->memory_bandwidth,
                'price_per_hour' => $type->price_per_hour,
                'available_count' => $type->available_count
            ];
        }, $gpuTypes);
    }

    /**
     * Configure GPU-optimized settings
     */
    public function configureGPUOptimizations(string $gpuId, array $config): array {
        $optimizations = [
            'vm_id' => $this->getGPUInstance($gpuId)['vm_id'],
            'memory_allocation' => $config['memory_allocation'] ?? 90, // percentage
            'persistence_mode' => $config['persistence_mode'] ?? true,
            'ECC_enabled' => $config['ECC_enabled'] ?? false,
            'mig_mode' => $config['mig_mode'] ?? false,
            'compute_mode' => $config['compute_mode'] ?? 'exclusive_thread'
        ];

        // Apply settings via nvidia-smi
        $this->applyGPUConfig($optimizations);

        // Update database
        $this->db->update('mod_gpu_instances', [
            'optimization_config' => json_encode($optimizations)
        ], ['id' => $gpuId]);

        return [
            'success' => true,
            'gpu_id' => $gpuId,
            'optimizations' => $optimizations
        ];
    }

    /**
     * Monitor GPU usage
     */
    public function getGPUUsage(int $serviceId): array {
        $gpuInstance = $this->db->select(
            "SELECT * FROM mod_gpu_instances WHERE service_id = ?",
            [$serviceId]
        )[0];

        if (!$gpuInstance) {
            return ['status' => 'not_found', 'service_id' => $serviceId];
        }

        // Query GPU metrics
        $metrics = $this->queryGPUDiagnostics($gpuInstance->vm_id);

        return [
            'gpu_id' => $gpuInstance->id,
            'gpu_type' => $gpuInstance->gpu_type,
            'utilization_percent' => $metrics['utilization'],
            'memory_used_gb' => $metrics['memory_used'],
            'memory_total_gb' => $metrics['memory_total'],
            'temperature_c' => $metrics['temperature'],
            'power_draw_w' => $metrics['power_draw'],
            'fan_speed_percent' => $metrics['fan_speed']
        ];
    }

    /**
     * Install specific driver version
     */
    public function installDriverVersion(string $gpuId, string $version): array {
        $gpu = $this->getGPUInstance($gpuId);

        // Download driver
        $downloadPath = $this->driverManager->downloadDriver($version);

        // Install on VM
        $installResult = $this->driverManager->installOnVM($gpu['vm_id'], $downloadPath, $version);

        // Update database
        $this->db->update('mod_gpu_instances', [
            'driver_version' => $version,
            'driver_installed_at' => date('Y-m-d H:i:s')
        ], ['id' => $gpuId]);

        return [
            'success' => $installResult['success'],
            'gpu_id' => $gpuId,
            'driver_version' => $version
        ];
    }

    /**
     * Setup MIG (Multi-Instance GPU)
     */
    public function setupMIG(string $gpuId, array $migConfig): array {
        $gpu = $this->getGPUInstance($gpuId);

        $migInstances = [];

        foreach ($migConfig['partitions'] as $partition) {
            $migId = $this->driverManager->createMIGPartition($gpu['vm_id'], $partition);

            $migInstances[] = [
                'id' => $migId,
                'gi_count' => $partition['gi_count'],
                'ci_count' => $partition['ci_count'],
                'memory_mb' => $partition['memory_mb']
            ];
        }

        // Store MIG configuration
        $this->db->update('mod_gpu_instances', [
            'mig_config' => json_encode($migInstances),
            'mig_enabled' => true
        ], ['id' => $gpuId]);

        return [
            'success' => true,
            'gpu_id' => $gpuId,
            'mig_instances' => $migInstances
        ];
    }

    // Private helper methods

    private function createGPUVM(array $gpuInstance, array $params): string {
        $vmId = 'gpu-vm-' . bin2hex(random_bytes(8));

        // Create VM with GPU passthrough configuration
        $config = [
            'gpu_type' => $params['gpu_type'],
            'gpu_count' => $params['gpu_count'] ?? 1,
            'pci_passthrough' => true,
            'memory' => $params['memory_gb'] ?? 32,
            'cpus' => $params['cpus'] ?? 8
        ];

        // Hypervisor-specific VM creation
        $this->createVMWithPassthrough($vmId, $config);

        return $vmId;
    }

    private function installGPUDrivers(string $vmId, array $params): void {
        $driverVersion = $params['driver_version'] ?? '535';

        // Install NVIDIA driver
        $installScript = <<<SCRIPT
#!/bin/bash
# NVIDIA Driver Installation

set -e

DRIVER_VERSION="{$driverVersion}"
NVIDIA_GPU="1"

# Disable nouveau
echo "blacklist nouveau" > /etc/modprobe.d/blacklist-nouveau.conf
echo "options nouveau modeset=0" >> /etc/modprobe.d/blacklist-nouveau.conf
update-initramfs -u

# Download driver
wget -O /tmp/nvidia-driver.run "https://download.nvidia.com/XFree86/Linux-x86_64/${DRIVER_VERSION}/NVIDIA-Linux-x86_64-${DRIVER_VERSION}.run"

# Install
chmod +x /tmp/nvidia-driver.run
/tmp/nvidia-driver.run --silent

# Verify installation
nvidia-smi
SCRIPT;

        $this->executeOnVM($vmId, $installScript);
    }

    private function verifyGPUAccess(string $vmId): array {
        $output = $this->executeOnVM($vmId, 'nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv');

        if (empty($output)) {
            return ['success' => false, 'error' => 'GPU not accessible'];
        }

        return [
            'success' => true,
            'gpu_info' => $output
        ];
    }
}

/**
 * GPU Driver Manager
 */
class GPUDriverManager {
    private $supportedDrivers = [
        '525' => '525.147.05',
        '535' => '535.154.05',
        '545' => '545.29.06'
    ];

    public function downloadDriver(string $version): string {
        $driverVersion = $this->supportedDrivers[$version] ?? $version;
        $downloadUrl = "https://download.nvidia.com/XFree86/Linux-x86_64/{$driverVersion}/NVIDIA-Linux-x86_64-{$driverVersion}.run";

        $localPath = "/tmp/nvidia-driver-{$version}.run";

        exec("wget -O {$localPath} {$downloadUrl}");

        return $localPath;
    }

    public function installOnVM(string $vmId, string $driverPath, string $version): array {
        // Copy driver to VM
        $this->copyToVM($vmId, $driverPath, '/tmp/nvidia-driver.run');

        // Install
        $this->executeOnVM($vmId, 'chmod +x /tmp/nvidia-driver.run && /tmp/nvidia-driver.run --silent');

        // Verify
        $result = $this->executeOnVM($vmId, 'nvidia-smi --query-gpu=driver_version --format=csv,noheader');

        return [
            'success' => $result !== '',
            'installed_version' => $result
        ];
    }

    public function createMIGPartition(string $vmId, array $config): string {
        $migId = 'mig-' . bin2hex(random_bytes(6));

        $command = "nvidia-smi mig -cgi {$config['gi_count']},{$config['ci_count']} -C";

        $this->executeOnVM($vmId, $command);

        return $migId;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_gpu_instances` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `vm_id` VARCHAR(100),
  `reservation_id` VARCHAR(100),
  `gpu_type` VARCHAR(50) NOT NULL,
  `gpu_count` INT DEFAULT 1,
  `vram_gb` INT,
  `driver_version` VARCHAR(20),
  `cuda_version` VARCHAR(20),
  `optimization_config` TEXT,
  `mig_enabled` TINYINT(1) DEFAULT 0,
  `mig_config` TEXT,
  `provisioning_status` ENUM('preparing', 'installing', 'ready', 'failed') DEFAULT 'preparing',
  `driver_installed_at` DATETIME,
  `ready_at` DATETIME,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_gpu_types` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(100) NOT NULL,
  `vram_gb` INT NOT NULL,
  `cuda_cores' INT,
  'tensor_cores' INT,
  'memory_bandwidth' VARCHAR(50),
  'tdp_watts' INT,
  'price_per_hour' DECIMAL(10,4) NOT NULL,
  'available_count' INT DEFAULT 0,
  'available' TINYINT(1) DEFAULT 1
);
```

## GPU Types and Pricing

| GPU Type | VRAM | CUDA Cores | Tensor Cores | Price/hr |
|----------|------|------------|--------------|----------|
| NVIDIA T4 | 16 GB | 2560 | 320 | $0.35 |
| NVIDIA V100 | 16 GB | 5120 | 640 | $2.50 |
| NVIDIA A100 | 40 GB | 6912 | 432 | $3.50 |
| NVIDIA A100 (80GB) | 80 GB | 6912 | 432 | $4.50 |
| NVIDIA H100 | 80 GB | 16896 | 528 | $6.50 |

## Best Practices

1. **Driver Management**: Keep drivers updated for security and performance
2. **License Compliance**: Properly manage GPU licensing
3. **Monitoring**: Track GPU utilization and temperature
4. **MIG for Multi-Tenancy**: Use MIG for better resource utilization
5. **Performance Tuning**: Optimize for specific workloads

## Related Skills

- whmcs-provisioning-master
- whmcs-monitoring-agent
- whmcs-resource-quotas
- whmcs-container-runtime
# Module Packaging for Distribution

Proper packaging ensures your WHMCS module can be easily installed, updated, and distributed to clients or the marketplace.

## Package Structure

### Standard Module Package

```
module-name-v1.2.3.zip
├── module.yml           # Module metadata
├── README.md            # Installation and usage guide
├── LICENSE              # License file
├── changelog.md         # Version history
├── config.php           # Configuration (optional)
├── src/
│   ├── Module.php       # Main module class
│   ├── Admin/
│   │   └── Controller.php
│   ├── Client/
│   │   └── Controller.php
│   ├── Services/
│   │   └── Service.php
│   └── ...
├── templates/           # Smarty templates
│   ├── admin/
│   └── client/
├── assets/
│   ├── css/
│   ├── js/
│   └── images/
├── lang/
│   ├── en.php
│   └── ...
└── migrations/          # Database migrations
    └── 1_0_0_initial.php
```

### Module Metadata (module.yml)

```yaml
name: Your Module Name
version: 1.2.3
description: A comprehensive module for WHMCS that does amazing things
author:
  name: Developer Name
  email: developer@example.com
  website: https://example.com
license: proprietary
minimum_whmcs_version: 8.0
php_minimum_version: 8.0

requires:
  modules: []
  extensions:
    - curl
    - json
    - openssl

features:
  - Feature One
  - Feature Two
  - Feature Three

screenshots:
  - path: assets/screenshots/screenshot1.png
    alt: Admin area view
  - path: assets/screenshots/screenshot2.png
    alt: Client portal view

hooks:
  - name: ClientAreaPage
    priority: 1
  - name: AfterModuleCronJob
    priority: 1

permissions:
  admin:
    - module_settings
    - view_reports
```

## Package Creation Script

### Build Script

```php
<?php
/**
 * Module packaging script
 */
class ModulePackager
{
    private string $sourceDir;
    private string $outputDir;
    private string $version;
    private array $excludePatterns = [
        '.git',
        '.gitignore',
        '.DS_Store',
        '*.log',
        'node_modules',
        'vendor',
        '.idea',
        '.vscode',
        'tests',
        '*.md~',
        'composer.lock',
    ];

    public function __construct(string $sourceDir, string $outputDir)
    {
        $this->sourceDir = rtrim($sourceDir, '/');
        $this->outputDir = rtrim($outputDir, '/');
    }

    /**
     * Create module package
     */
    public function package(string $version): string
    {
        $this->version = $version;
        $tempDir = $this->createTempDirectory();

        // Copy files excluding patterns
        $this->copySourceFiles($tempDir);

        // Generate package files
        $this->generatePackageFiles($tempDir);

        // Create ZIP
        $packagePath = $this->createZip($tempDir);

        // Clean up
        $this->removeDirectory($tempDir);

        return $packagePath;
    }

    /**
     * Create package from source
     */
    private function copySourceFiles(string $destDir): void
    {
        $this->copyRecursive($this->sourceDir, $destDir);

        // Remove excluded files
        foreach ($this->excludePatterns as $pattern) {
            $this->removeMatchingFiles($destDir, $pattern);
        }
    }

    /**
     * Generate version-specific files
     */
    private function generatePackageFiles(string $tempDir): void
    {
        // Create version file
        file_put_contents(
            $tempDir . '/VERSION',
            "{$this->version}\n" . date('Y-m-d H:i:s') . "\n"
        );

        // Create manifest
        $manifest = [
            'name' => basename($this->sourceDir),
            'version' => $this->version,
            'created_at' => date('c'),
            'files' => $this->getFileList($tempDir),
        ];

        file_put_contents(
            $tempDir . '/MANIFEST.json',
            json_encode($manifest, JSON_PRETTY_PRINT)
        );

        // Calculate checksums
        $checksums = [];
        foreach ($manifest['files'] as $file) {
            $checksums[$file] = md5_file($tempDir . '/' . $file);
        }

        file_put_contents(
            $tempDir . '/CHECKSUMS.md5',
            implode("\n", array_map(
                fn($f, $c) => "$c  $f",
                array_keys($checksums),
                $checksums
            ))
        );
    }

    /**
     * Create ZIP archive
     */
    private function createZip(string $tempDir): string
    {
        $moduleName = basename($this->sourceDir);
        $outputFile = "{$this->outputDir}/{$moduleName}-{$this->version}.zip";

        // Ensure output directory exists
        if (!is_dir($this->outputDir)) {
            mkdir($this->outputDir, 0755, true);
        }

        $zip = new ZipArchive();
        $zip->open($outputFile, ZipArchive::CREATE | ZipArchive::OVERWRITE);

        $this->addDirectoryToZip($zip, $tempDir, basename($tempDir));

        $zip->close();

        return $outputFile;
    }

    /**
     * Add directory recursively to ZIP
     */
    private function addDirectoryToZip(ZipArchive $zip, string $dir, string $basePath): void
    {
        $files = new RecursiveIteratorIterator(
            new RecursiveDirectoryIterator($dir),
            RecursiveIteratorIterator::SELF_FIRST
        );

        foreach ($files as $file) {
            if ($file->isDir()) {
                continue;
            }

            $filePath = $file->getRealPath();
            $relativePath = $basePath . '/' . substr($filePath, strlen($dir) + 1);

            $zip->addFile($filePath, $relativePath);
        }
    }

    /**
     * Copy directory recursively
     */
    private function copyRecursive(string $src, string $dest): void
    {
        if (!is_dir($dest)) {
            mkdir($dest, 0755, true);
        }

        $items = new RecursiveIteratorIterator(
            new RecursiveDirectoryIterator($src, RecursiveDirectoryIterator::SKIP_DOTS),
            RecursiveIteratorIterator::SELF_FIRST
        );

        foreach ($items as $item) {
            $target = $dest . '/' . $items->getSubPathName();

            if ($item->isDir()) {
                if (!is_dir($target)) {
                    mkdir($target, 0755, true);
                }
            } else {
                $dir = dirname($target);
                if (!is_dir($dir)) {
                    mkdir($dir, 0755, true);
                }
                copy($item->getRealPath(), $target);
            }
        }
    }

    /**
     * Remove matching files
     */
    private function removeMatchingFiles(string $dir, string $pattern): void
    {
        $files = new RecursiveIteratorIterator(
            new RecursiveDirectoryIterator($dir),
            RecursiveIteratorIterator::CHILD_FIRST
        );

        foreach ($files as $file) {
            $path = $file->getPathname();

            if ($this->matchesPattern($path, $pattern)) {
                if ($file->isDir()) {
                    rmdir($path);
                } else {
                    unlink($path);
                }
            }
        }
    }

    private function matchesPattern(string $path, string $pattern): bool
    {
        if (str_starts_with($pattern, '*')) {
            return fnmatch($pattern, basename($path));
        }

        return str_contains($path, $pattern);
    }

    private function getFileList(string $dir): array
    {
        $files = [];
        $iterator = new RecursiveIteratorIterator(
            new RecursiveDirectoryIterator($dir),
            RecursiveIteratorIterator::LEAVES_ONLY
        );

        foreach ($iterator as $file) {
            if ($file->isDir()) {
                continue;
            }

            $files[] = $iterator->getSubPathName();
        }

        return $files;
    }

    private function createTempDirectory(): string
    {
        $tempDir = sys_get_temp_dir() . '/module_package_' . uniqid();
        mkdir($tempDir, 0755, true);
        return $tempDir;
    }

    private function removeDirectory(string $dir): void
    {
        $files = new RecursiveIteratorIterator(
            new RecursiveDirectoryIterator($dir, RecursiveIteratorIterator::CHILD_FIRST),
            RecursiveIteratorIterator::CHILD_FIRST
        );

        foreach ($files as $file) {
            if ($file->isDir()) {
                rmdir($file->getPathname());
            } else {
                unlink($file->getPathname());
            }
        }

        rmdir($dir);
    }
}

// CLI usage
if (php_sapi_name() === 'cli') {
    $sourceDir = $argv[1] ?? __DIR__;
    $outputDir = $argv[2] ?? __DIR__ . '/dist';
    $version = $argv[3] ?? '1.0.0';

    $packager = new ModulePackager($sourceDir, $outputDir);
    $packagePath = $packager->package($version);

    echo "Package created: $packagePath\n";
}
```

## Installation Script

### Module Installer

```php
<?php
/**
 * Module installation handler
 */
class ModuleInstaller
{
    private string $moduleName;
    private string $modulePath;

    public function __construct(string $moduleName)
    {
        $this->moduleName = $moduleName;
        $this->modulePath = ROOTDIR . '/modules/' . $this->getModuleType() . '/' . $moduleName;
    }

    /**
     * Run installation
     */
    public function install(): InstallationResult
    {
        $result = new InstallationResult();

        try {
            // Verify requirements
            $this->verifyRequirements();

            // Create directories
            $this->createDirectories();

            // Copy files
            $this->copyFiles();

            // Run database migrations
            $this->runMigrations();

            // Register hooks
            $this->registerHooks();

            // Set default configuration
            $this->setDefaultConfig();

            // Mark as installed
            $this->markInstalled();

            $result->success = true;
            $result->message = "Module installed successfully";
        } catch (InstallationException $e) {
            $this->rollback();
            $result->success = false;
            $result->message = $e->getMessage();
            $result->errors = [$e];
        }

        return $result;
    }

    /**
     * Run uninstallation
     */
    public function uninstall(): void
    {
        // Unregister hooks
        $this->unregisterHooks();

        // Run downgrade migrations
        $this->runDowngradeMigrations();

        // Remove files
        $this->removeFiles();

        // Remove directories
        $this->removeDirectories();

        // Clean configuration
        $this->cleanConfig();

        // Mark as uninstalled
        $this->markUninstalled();
    }

    /**
     * Verify system requirements
     */
    private function verifyRequirements(): void
    {
        // Check PHP version
        if (version_compare(PHP_VERSION, '8.0', '<')) {
            throw new InstallationException('PHP 8.0 or higher is required');
        }

        // Check required extensions
        $requiredExtensions = ['curl', 'json', 'pdo'];
        foreach ($requiredExtensions as $ext) {
            if (!extension_loaded($ext)) {
                throw new InstallationException("PHP extension '{$ext}' is required");
            }
        }

        // Check WHMCS version
        $whmcsVersion = App::getVersion();
        if (version_compare($whmcsVersion, '8.0', '<')) {
            throw new InstallationException('WHMCS 8.0 or higher is required');
        }

        // Check directory permissions
        $dirsToCheck = [$this->modulePath];
        foreach ($dirsToCheck as $dir) {
            $parentDir = dirname($dir);
            if (!is_writable($parentDir)) {
                throw new InstallationException("Directory not writable: {$parentDir}");
            }
        }
    }

    private function createDirectories(): void
    {
        $directories = [
            $this->modulePath,
            $this->modulePath . '/src',
            $this->modulePath . '/templates',
            $this->modulePath . '/templates/admin',
            $this->modulePath . '/templates/client',
            $this->modulePath . '/assets',
            $this->modulePath . '/assets/css',
            $this->modulePath . '/assets/js',
            $this->modulePath . '/assets/images',
            $this->modulePath . '/lang',
        ];

        foreach ($directories as $dir) {
            if (!is_dir($dir)) {
                mkdir($dir, 0755, true);
            }
        }
    }

    private function copyFiles(): void
    {
        // Copy module files from package
        // Implementation depends on source location
    }

    private function runMigrations(): void
    {
        $runner = new MigrationRunner($this->moduleName, $this->modulePath . '/migrations');
        $result = $runner->run();

        if (!$result->success) {
            throw new InstallationException('Database migration failed');
        }
    }

    private function registerHooks(): void
    {
        $hooks = $this->getHooksFromModule();

        foreach ($hooks as $hook) {
            Capsule::table('mod_hooks')->updateOrInsert(
                ['hook' => $hook['name'], 'module' => $this->moduleName],
                [
                    'hook' => $hook['name'],
                    'module' => $this->moduleName,
                    'priority' => $hook['priority'] ?? 0,
                    'description' => $hook['description'] ?? '',
                ]
            );
        }
    }

    private function setDefaultConfig(): void
    {
        $defaults = $this->getDefaultConfig();

        foreach ($defaults as $key => $value) {
            Capsule::table('mod_configuration')->updateOrInsert(
                ['setting_name' => $key, 'module' => $this->moduleName],
                [
                    'module' => $this->moduleName,
                    'setting_name' => $key,
                    'value' => is_array($value) ? json_encode($value) : $value,
                ]
            );
        }
    }

    private function markInstalled(): void
    {
        Capsule::table('mod_modules')->updateOrInsert(
            ['module_name' => $this->moduleName],
            [
                'module_name' => $this->moduleName,
                'version' => $this->getModuleVersion(),
                'installed_at' => date('Y-m-d H:i:s'),
                'status' => 'active',
            ]
        );
    }

    private function rollback(): void
    {
        // Clean up any partial installation
        $this->uninstall();
    }

    private function getModuleType(): string
    {
        return 'addons'; // Default, can be 'registrars', 'gateways', etc.
    }
}
```

## Version Management

### Version Updater

```php
<?php
/**
 * Module version updater
 */
class ModuleVersionUpdater
{
    private string $moduleName;

    public function __construct(string $moduleName)
    {
        $this->moduleName = $moduleName;
    }

    /**
     * Check and perform updates
     */
    public function update(string $fromVersion, string $toVersion): UpdateResult
    {
        $result = new UpdateResult();

        $migrations = $this->getRequiredMigrations($fromVersion, $toVersion);

        foreach ($migrations as $migration) {
            try {
                $this->runMigration($migration);
                $result->migrated[] = $migration;
            } catch (Exception $e) {
                $result->errors[] = [
                    'migration' => $migration,
                    'error' => $e->getMessage(),
                ];
                $result->success = false;
            }
        }

        $result->success = empty($result->errors);

        if ($result->success) {
            $this->updateVersionInDb($toVersion);
        }

        return $result;
    }

    private function getRequiredMigrations(string $from, string $to): array
    {
        // Return ordered list of migrations to run
        return [];
    }
}
```

## Distribution Checklist

| Item | Description | Status |
|------|-------------|--------|
| module.yml | Complete metadata file | [ ] |
| README.md | Installation and usage guide | [ ] |
| LICENSE | Appropriate license file | [ ] |
| Changelog | Version history | [ ] |
| Screenshots | Marketing screenshots | [ ] |
| Version | Semantic versioned | [ ] |
| Checksums | MD5/SHA256 verification | [ ] |
| Composer | PHP dependencies defined | [ ] |
| Tests | Unit test coverage | [ ] |
| Demo | Demo environment available | [ ] |

## Related Patterns

- [Module Versioning](./module-versioning.md) - Version management
- [Database Migrations](./database-migrations.md) - Schema migrations
- [Module Release Checklist](./module-release-checklist.md) - Release process
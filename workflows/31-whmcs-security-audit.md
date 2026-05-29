# WHMCS Security Audit Workflow

## Overview
This workflow provides a systematic approach to conducting security audits of WHMCS installations and modules.

## Step 1: Security Audit Checklist

```php
<?php
// src/Service/SecurityAuditService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class SecurityAuditService
{
    private $findings = [];
    private $severityLevels = ['critical', 'high', 'medium', 'low', 'info'];

    public function runFullAudit(): array
    {
        $this->findings = [];

        $this->auditFilePermissions();
        $this->auditDirectoryPermissions();
        $this->auditDatabaseSecurity();
        $this->auditPasswordPolicies();
        $this->auditSessionSecurity();
        $this->auditInputValidation();
        $this->auditSqlInjection();
        $this->auditCsrfProtection();
        $this->auditXssProtection();
        $this->auditFileUploads();
        $this->auditApiSecurity();
        $this->auditEncryption();
        $this->auditBackupSecurity();
        $this->auditAccessControls();

        return [
            'timestamp' => date('Y-m-d H:i:s'),
            'total_findings' => count($this->findings),
            'by_severity' => $this->getFindingsBySeverity(),
            'findings' => $this->findings
        ];
    }

    private function addFinding(string $category, string $severity, string $title, string $description, array $details = []): void
    {
        $this->findings[] = [
            'category' => $category,
            'severity' => $severity,
            'title' => $title,
            'description' => $description,
            'details' => $details,
            'remediation' => $this->getRemediation($category, $title)
        ];
    }

    private function auditFilePermissions(): void
    {
        $criticalFiles = [
            dirname(__DIR__, 3) . '/configuration.php',
            dirname(__DIR__, 3) . '/init.php',
            dirname(__DIR__, 3) . '/config.php'
        ];

        foreach ($criticalFiles as $file) {
            if (file_exists($file)) {
                $perms = substr(sprintf('%o', fileperms($file)), -4);
                if ($perms !== '0644') {
                    $this->addFinding(
                        'File Permissions',
                        'critical',
                        'Insecure file permissions',
                        "File $file has permissions $perms instead of 0644",
                        ['current_permissions' => $perms, 'expected' => '0644']
                    );
                }
            }
        }
    }

    private function auditDirectoryPermissions(): void
    {
        $writableDirs = [
            dirname(__DIR__, 3) . '/storage',
            dirname(__DIR__, 3) . '/attachments',
            dirname(__DIR__, 3) . '/downloads'
        ];

        foreach ($writableDirs as $dir) {
            if (is_dir($dir)) {
                $perms = substr(sprintf('%o', fileperms($dir)), -4);
                if ($perms !== '0755') {
                    $this->addFinding(
                        'Directory Permissions',
                        'medium',
                        'Directory permissions not optimal',
                        "Directory $dir has permissions $perms",
                        ['current_permissions' => $perms, 'expected' => '0755']
                    );
                }
            }
        }
    }

    private function auditDatabaseSecurity(): void
    {
        // Check for root access
        $dbUser = Capsule::config('db_username');
        if ($dbUser === 'root') {
            $this->addFinding(
                'Database Security',
                'high',
                'Database root user in use',
                'The database connection uses the root user',
                ['username' => 'root']
            );
        }

        // Check for remote database access
        $dbHost = Capsule::config('db_host');
        if ($dbHost !== 'localhost' && $dbHost !== '127.0.0.1') {
            $this->addFinding(
                'Database Security',
                'medium',
                'Remote database connection',
                'Database is accessed remotely, increasing attack surface',
                ['host' => $dbHost]
            );
        }
    }

    private function auditPasswordPolicies(): void
    {
        $config = Capsule::table('tblconfiguration')
            ->whereIn('setting', ['minpasswordstrength', 'passwordrequire'])
            ->get()
            ->keyBy('setting');

        if (!isset($config['minpasswordstrength']) || $config['minpasswordstrength']->value < 3) {
            $this->addFinding(
                'Password Policy',
                'high',
                'Weak password requirements',
                'Password strength requirement is too low',
                ['current' => $config['minpasswordstrength']->value ?? 0, 'recommended' => 3]
            );
        }
    }

    private function auditSessionSecurity(): void
    {
        $sessionConfig = [
            'session.cookie_httponly',
            'session.cookie_secure',
            'session.use_strict_mode'
        ];

        foreach ($sessionConfig as $setting) {
            $value = ini_get($setting);
            if ($setting === 'session.cookie_httponly' && !$value) {
                $this->addFinding(
                    'Session Security',
                    'high',
                    'Session cookies not HTTPOnly',
                    'Session cookies can be accessed via JavaScript'
                );
            }

            if ($setting === 'session.cookie_secure' && !$value && isset($_SERVER['HTTPS'])) {
                $this->addFinding(
                    'Session Security',
                    'medium',
                    'Session cookies not secure',
                    'Session cookies should be secure on HTTPS sites'
                );
            }
        }
    }

    private function auditInputValidation(): void
    {
        // Check for common XSS vulnerabilities in templates
        $templates = glob(dirname(__DIR__, 3) . '/templates/*/*.tpl');

        foreach ($templates as $template) {
            $content = file_get_contents($template);

            // Check for unescaped output
            if (preg_match('/\{\$[^}]+\}/', $content) && !preg_match('/\{&#36;[^}]+\}/', $content)) {
                // Potential issue - need manual review
                $this->addFinding(
                    'Input Validation',
                    'info',
                    'Template output review needed',
                    "Review $template for proper escaping"
                );
            }
        }
    }

    private function auditSqlInjection(): void
    {
        // Static analysis would be done here
        // For runtime audit, check for queries without proper escaping

        $this->addFinding(
            'SQL Injection',
            'info',
            'SQL injection audit recommended',
            'Run static analysis tools like Psalm or PHPStan to check for SQL injection vulnerabilities'
        );
    }

    private function auditCsrfProtection(): void
    {
        // Check if CSRF tokens are being used
        $hooksDir = dirname(__DIR__, 3) . '/includes/hooks';

        if (is_dir($hooksDir)) {
            $hooks = glob("$hooksDir/*.php");

            foreach ($hooks as $hook) {
                $content = file_get_contents($hook);

                // Check for form submissions without CSRF check
                if (preg_match('/\$_POST/', $content) && !preg_match('/csrfToken|token/', $content)) {
                    $this->addFinding(
                        'CSRF Protection',
                        'medium',
                        'Potential missing CSRF protection',
                        "Review $hook for CSRF token validation",
                        ['file' => basename($hook)]
                    );
                }
            }
        }
    }

    private function auditXssProtection(): void
    {
        // Check for XSS protections in headers
        $headers = $this->getSecurityHeaders();

        if (!isset($headers['X-XSS-Protection'])) {
            $this->addFinding(
                'XSS Protection',
                'medium',
                'X-XSS-Protection header not set',
                'Browser XSS filter not explicitly enabled'
            );
        }

        if (!isset($headers['Content-Security-Policy'])) {
            $this->addFinding(
                'XSS Protection',
                'low',
                'Content-Security-Policy header not set',
                'Consider implementing CSP for additional XSS protection'
            );
        }
    }

    private function auditFileUploads(): void
    {
        // Check upload configuration
        $uploadMaxFilesize = ini_get('upload_max_filesize');
        $postMaxSize = ini_get('post_max_size');

        $uploadBytes = $this->parseSize($uploadMaxFilesize);
        if ($uploadBytes > 10 * 1024 * 1024) { // 10MB
            $this->addFinding(
                'File Uploads',
                'medium',
                'Large file upload limit',
                'File upload limit is high, increasing risk of DoS',
                ['current_limit' => $uploadMaxFilesize]
            );
        }
    }

    private function auditApiSecurity(): void
    {
        // Check API configuration
        $apiAccessLog = Capsule::table('tblactivitylog')
            ->where('description', 'like', '%API%')
            ->where('date', '>=', date('Y-m-d', strtotime('-7 days')))
            ->count();

        if ($apiAccessLog === 0) {
            $this->addFinding(
                'API Security',
                'info',
                'API access logging review',
                'Consider implementing API access logging for security monitoring'
            );
        }
    }

    private function auditEncryption(): void
    {
        // Check SSL/TLS configuration
        if (!isset($_SERVER['HTTPS']) && !isset($_SERVER['HTTP_X_FORWARDED_PROTO'])) {
            $this->addFinding(
                'Encryption',
                'critical',
                'No HTTPS detected',
                'Site is not using HTTPS encryption'
            );
        }
    }

    private function auditBackupSecurity(): void
    {
        $backupDir = dirname(__DIR__, 3) . '/backups';

        if (is_dir($backupDir)) {
            $perms = substr(sprintf('%o', fileperms($backupDir)), -4);
            if ($perms !== '0750') {
                $this->addFinding(
                    'Backup Security',
                    'high',
                    'Insecure backup directory permissions',
                    'Backups could be accessed by unauthorized users',
                    ['permissions' => $perms]
                );
            }
        }
    }

    private function auditAccessControls(): void
    {
        // Check admin password age
        $admins = Capsule::table('tbladmins')
            ->where('disabled', '!=', 1)
            ->get();

        foreach ($admins as $admin) {
            $lastLogin = strtotime($admin->lastlogin);
            $daysSinceLogin = (time() - $lastLogin) / (60 * 60 * 24);

            if ($daysSinceLogin > 90 && $admin->lastlogin !== '0000-00-00 00:00:00') {
                $this->addFinding(
                    'Access Control',
                    'low',
                    'Admin account inactive',
                    "Admin {$admin->username} has not logged in for " . round($daysSinceLogin) . " days",
                    ['username' => $admin->username, 'last_login' => $admin->lastlogin]
                );
            }
        }
    }

    private function getSecurityHeaders(): array
    {
        // In a real implementation, this would check actual headers
        return [];
    }

    private function getFindingsBySeverity(): array
    {
        $bySeverity = [];
        foreach ($this->severityLevels as $level) {
            $bySeverity[$level] = 0;
        }

        foreach ($this->findings as $finding) {
            $bySeverity[$finding['severity']]++;
        }

        return $bySeverity;
    }

    private function getRemediation(string $category, string $title): string
    {
        $remediations = [
            'File Permissions' => 'Use chmod 644 for files and chmod 755 for directories.',
            'Database Security' => 'Create a dedicated database user with limited privileges.',
            'Password Policy' => 'Increase minimum password strength requirement to at least 3.',
            'Session Security' => 'Enable HTTPOnly and Secure flags for session cookies.',
            'CSRF Protection' => 'Implement CSRF token validation in all form submissions.',
            'Encryption' => 'Configure HTTPS and force SSL/TLS encryption.',
            'Backup Security' => 'Restrict backup directory permissions to owner only.'
        ];

        return $remediations[$category] ?? 'Review and fix according to security best practices.';
    }

    private function parseSize(string $size): int
    {
        $unit = preg_replace('/[^a-zA-Z]/', '', $size);
        $value = (int)$size;

        return match (strtoupper($unit)) {
            'G' => $value * 1024 * 1024 * 1024,
            'M' => $value * 1024 * 1024,
            'K' => $value * 1024,
            default => $value
        };
    }
}
```

## Step 2: Generate Audit Report

```php
<?php
// admin/security_report.php

add_hook('AdminAreaPage', 1, function() {
    $auditService = new \WHMCS\Module\Addon\YourModule\Service\SecurityAuditService();
    $report = $auditService->runFullAudit();

    return [
        'templatefile' => 'admin/security_report',
        'vars' => ['security_report' => $report]
    ];
});
```

## Verification Checklist

- [ ] Audit service implemented
- [ ] File permission checks working
- [ ] Database security checks working
- [ ] Password policy checks working
- [ ] Session security checks working
- [ ] CSRF protection checks working
- [ ] XSS protection checks working
- [ ] Encryption checks working
- [ ] Report generation working
- [ ] Audit completed without errors

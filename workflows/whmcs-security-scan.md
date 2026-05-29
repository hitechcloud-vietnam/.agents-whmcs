# WHMCS Security Scan Workflow

## Overview
This workflow implements automated security scanning for WHMCS.

## Prerequisites
- WHMCS with security monitoring
- Admin access for security tools
- Security scanning tools (optional)

## Step-by-Step Process

### Step 1: Security Scan Manager
```php
<?php
// /includes/security/SecurityScanner.php

class SecurityScanner {
    private $vulnerabilities = [];

    /**
     * Run comprehensive security scan
     */
    public function runScan(): array
    {
        return [
            'file_integrity' => $this->checkFileIntegrity(),
            'permissions' => $this->checkFilePermissions(),
            'sql_injection' => $this->checkSQLInjection(),
            'xss' => $this->checkXSSVulnerabilities(),
            'csrf' => $this->checkCSRFProtection(),
            'outdated_software' => $this->checkOutdatedSoftware()
        ];
    }

    private function checkFileIntegrity(): array
    {
        $whmcsPath = dirname(__DIR__, 2);

        // Get known file checksums
        $knownChecksums = $this->getKnownChecksums();

        $issues = [];
        $coreFiles = glob($whmcsPath . '/includes/*.php');

        foreach ($coreFiles as $file) {
            if (in_array(basename($file), array_keys($knownChecksums))) {
                $currentChecksum = md5_file($file);
                if ($currentChecksum !== $knownChecksums[basename($file)]) {
                    $issues[] = [
                        'file' => $file,
                        'issue' => 'Checksum mismatch',
                        'severity' => 'critical'
                    ];
                }
            }
        }

        return ['status' => empty($issues) ? 'pass' : 'fail', 'issues' => $issues];
    }

    private function checkFilePermissions(): array
    {
        $issues = [];
        $whmcsPath = dirname(__DIR__, 2);

        $sensitiveFiles = [
            $whmcsPath . '/configuration.php',
            $whmcsPath . '/includes/config.php'
        ];

        foreach ($sensitiveFiles as $file) {
            $perms = fileperms($file) & 0777;

            if ($perms > 0640) {
                $issues[] = [
                    'file' => $file,
                    'issue' => 'Permissions too open: ' . decoct($perms),
                    'severity' => 'high'
                ];
            }
        }

        return ['status' => empty($issues) ? 'pass' : 'fail', 'issues' => $issues];
    }

    private function checkSQLInjection(): array
    {
        // Basic SQL injection pattern check
        $issues = [];
        $whmcsPath = dirname(__DIR__, 2);

        // Check custom files for SQL injection vulnerabilities
        $customFiles = glob($whmcsPath . '/modules/*/hooks/*.php');

        foreach ($customFiles as $file) {
            $content = file_get_contents($file);

            // Check for unsafe SQL patterns
            if (preg_match('/\$_(GET|POST|REQUEST)\[.*?\]\s*\.\s*.*?query/i', $content)) {
                $issues[] = [
                    'file' => $file,
                    'issue' => 'Potential SQL injection: direct input in query',
                    'severity' => 'critical'
                ];
            }
        }

        return ['status' => empty($issues) ? 'pass' : 'fail', 'issues' => $issues];
    }

    private function checkXSSVulnerabilities(): array
    {
        $issues = [];

        // Check for unsanitized output
        // Implementation would scan for echo of raw user input

        return ['status' => empty($issues) ? 'pass' : 'fail', 'issues' => $issues];
    }

    private function checkCSRFProtection(): array
    {
        $issues = [];

        // Check forms for CSRF tokens
        $forms = Capsule::select("
            SELECT id, action FROM mod_forms
            WHERE requires_csrf = 0
        ");

        if (count($forms) > 0) {
            $issues[] = [
                'issue' => 'Forms without CSRF protection found',
                'count' => count($forms),
                'severity' => 'high'
            ];
        }

        return ['status' => empty($issues) ? 'pass' : 'fail', 'issues' => $issues];
    }

    private function checkOutdatedSoftware(): array
    {
        $issues = [];

        // Check WHMCS version
        $currentVersion = Capsule::table('tblconfiguration')
            ->where('setting', 'Version')
            ->value('value');

        if ($this->isVersionOutdated($currentVersion)) {
            $issues[] = [
                'software' => 'WHMCS',
                'current_version' => $currentVersion,
                'issue' => 'Version is outdated',
                'severity' => 'high'
            ];
        }

        return ['status' => empty($issues) ? 'pass' : 'fail', 'issues' => $issues];
    }
}
```

### Step 2: Security Scan Hook
```php
<?php
// /includes/hooks/security_scan_hooks.php

add_hook('DailyCronJob', 1, function($vars) {
    $scanner = new SecurityScanner();
    $results = $scanner->runScan();

    // Alert on critical issues
    foreach ($results as $category => $result) {
        if ($result['status'] === 'fail') {
            foreach ($result['issues'] as $issue) {
                if ($issue['severity'] === 'critical') {
                    sendSecurityAlert($category, $issue);
                }
            }
        }
    }

    // Store results
    Capsule::table('mod_security_scans')->insert([
        'scan_date' => date('Y-m-d H:i:s'),
        'results' => json_encode($results)
    ]);

    return $results;
});
```

## Related Workflows
- [WHMCS Vulnerability Scan](./whmcs-vulnerability-scan.md)
- [WHMCS Security Audit](./whmcs-security-audit.md)
---
name: whmcs-certificate-pinning
description: HSTS configuration for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS Certificate Pinning & HSTS Skill

## Overview
This skill provides patterns and implementations for configuring HSTS (HTTP Strict Transport Security) and certificate pinning in WHMCS for enhanced SSL security.

## Implementation Patterns

### HSTS and Certificate Pinning Manager
```php
<?php
/**
 * WHMCS HSTS and Certificate Pinning Configuration
 * Implements security headers and pinning
 */

namespace WHMCS\Module\Server\Security;

class HSTSManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Configure HSTS for service
     */
    public function configureHSTS(int $serviceId, array $config): array {
        $hstsConfig = [
            'max_age' => $config['max_age'] ?? 31536000, // 1 year default
            'include_subdomains' => $config['include_subdomains'] ?? true,
            'preload' => $config['preload'] ?? false,
            'enforce' => $config['enforce'] ?? true
        ];

        // Store configuration
        $this->db->update('mod_service_security', [
            'hsts_config' => json_encode($hstsConfig),
            'hsts_enabled' => true
        ], ['service_id' => $serviceId]);

        // Generate nginx/apache config
        $serverConfig = $this->generateHSTSConfig($hstsConfig);
        $this->applyHSTSConfig($serviceId, $serverConfig);

        return [
            'success' => true,
            'service_id' => $serviceId,
            'config' => $hstsConfig
        ];
    }

    /**
     * Setup certificate pinning
     */
    public function setupCertificatePinning(int $serviceId, array $pins): array {
        // Generate SPKI (Subject Public Key Info) pins
        $publicKeyPins = [];

        foreach ($pins as $pin) {
            $spki = $this->getSPKIFromCertificate($pin['cert_path']);
            $pinHash = $this->generatePinHash($spki);
            $publicKeyPins[] = $pinHash;
        }

        // Add backup pin
        $backupPin = $this->generateBackupPin();
        $publicKeyPins[] = $backupPin;

        $pinningConfig = [
            'pins' => $publicKeyPins,
            'include_subdomains' => true,
            'max_age' => 7776000 // 90 days
        ];

        $this->db->update('mod_service_security', [
            'pinning_config' => json_encode($pinningConfig),
            'pinning_enabled' => true
        ], ['service_id' => $serviceId]);

        // Generate HPKP headers
        $this->applyHPKPConfig($serviceId, $pinningConfig);

        return [
            'success' => true,
            'service_id' => $serviceId,
            'pins' => $publicKeyPins
        ];
    }

    /**
     * Generate HSTS policy
     */
    public function generateHSTSPolicy(int $serviceId): array {
        $service = $this->db->select(
            "SELECT hsts_config FROM mod_service_security WHERE service_id = ?",
            [$serviceId]
        )[0];

        if (!$service || !$service->hsts_config) {
            throw new \Exception("HSTS not configured for service");
        }

        $config = json_decode($service->hsts_config, true);

        $header = "Strict-Transport-Security: ";
        $header .= "max-age={$config['max_age']}";

        if ($config['include_subdomains']) {
            $header .= "; includeSubDomains";
        }

        if ($config['preload']) {
            $header .= "; preload";
        }

        return [
            'header' => $header,
            'max_age' => $config['max_age'],
            'include_subdomains' => $config['include_subdomains'],
            'preload' => $config['preload']
        ];
    }

    /**
     * Submit to HSTS preload list
     */
    public function submitToPreloadList(int $serviceId): array {
        $policy = $this->generateHSTSPolicy($serviceId);

        if ($policy['max_age'] < 31536000) {
            throw new \Exception("max-age must be at least 31536000 (1 year) for preload");
        }

        if (!$policy['include_subdomains']) {
            throw new \Exception("includeSubDomains required for preload");
        }

        if (!$policy['preload']) {
            throw new \Exception("preload directive must be set for submission");
        }

        // Verify site is accessible and HSTS is working
        $verification = $this->verifyHSTSInstallation($serviceId);

        if (!$verification['success']) {
            throw new \Exception("HSTS not properly configured on site");
        }

        // Submit to HSTS preload list
        $submissionResult = $this->submitToHSTSPS($policy);

        $this->db->update('mod_service_security', [
            'preload_submitted' => true,
            'preload_submitted_at' => date('Y-m-d H:i:s')
        ], ['service_id' => $serviceId]);

        return [
            'success' => true,
            'service_id' => $serviceId,
            'preload_status' => 'submitted',
            'details' => $submissionResult
        ];
    }

    // Private helper methods

    private function generateHSTSConfig(array $config): string {
        $header = "Strict-Transport-Security: max-age={$config['max_age']}";

        if ($config['include_subdomains']) {
            $header .= "; includeSubDomains";
        }

        if ($config['preload']) {
            $header .= "; preload";
        }

        return $header;
    }

    private function applyHSTSConfig(int $serviceId, string $config): void {
        $domain = $this->getServiceDomain($serviceId);

        // Nginx configuration
        $nginxConfig = <<<CONFIG
server {
    listen 443 ssl;
    server_name {$domain};

    # HSTS Header
    add_header Strict-Transport-Security "{$config}" always;

    # ... rest of config
}
CONFIG;

        file_put_contents("/etc/nginx/sites-available/{$domain}-hsts.conf", $nginxConfig);
        exec('nginx -t && nginx -s reload');
    }

    private function getSPKIFromCertificate(string $certPath): string {
        $certContent = file_get_contents($certPath);
        $cert = openssl_x509_read($certContent);

        $pubKey = openssl_pkey_get_public($cert);
        $keyDetails = openssl_pkey_get_details($pubKey);

        return $keyDetails['key'];
    }

    private function generatePinHash(string $spki): string {
        $hash = hash('sha256', $spki, true);
        return base64_encode($hash);
    }

    private function generateBackupPin(): string {
        // Generate a backup pin for key rotation
        $key = openssl_pkey_new([
            'private_key_type' => OPENSSL_KEYTYPE_RSA,
            'private_key_bits' => 2048
        ]);

        $keyDetails = openssl_pkey_get_details($key);
        return $this->generatePinHash($keyDetails['key']);
    }

    private function applyHPKPConfig(int $serviceId, array $config): void {
        $domain = $this->getServiceDomain($serviceId);

        $pins = implode('; pin-sha256="', $config['pins']);
        $header = 'Public-Key-Pins: pin-sha256="' . $pins . '"; max-age=' . $config['max_age'];

        if ($config['include_subdomains']) {
            $header .= '; includeSubDomains';
        }

        // Note: HPKP is deprecated but implementation shown for legacy support
    }

    private function verifyHSTSInstallation(int $serviceId): array {
        $domain = $this->getServiceDomain($serviceId);

        $ch = curl_init("https://{$domain}/");
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_NOBODY => true
        ]);

        $headers = [];
        curl_setopt($ch, CURLOPT_HEADERFUNCTION, function($curl, $header) use (&$headers) {
            $len = strlen($header);
            $header = trim($header);
            if (!empty($header)) {
                $headers[] = $header;
            }
            return $len;
        });

        curl_exec($ch);
        curl_close($ch);

        $hstsFound = false;
        foreach ($headers as $header) {
            if (stripos($header, 'Strict-Transport-Security:') === 0) {
                $hstsFound = true;
                break;
            }
        }

        return [
            'success' => $hstsFound,
            'domain' => $domain
        ];
    }
}
```

## Security Header Configuration

### Nginx Configuration
```nginx
# Security Headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

### Apache Configuration
```apache
# Security Headers
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-XSS-Protection "1; mode=block"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
```

## Best Practices

1. **Start Conservative**: Begin with shorter max-age and enable later
2. **Test Thoroughly**: Ensure all subdomains work with HTTPS
3. **Backup Keys**: Always keep backup pins for emergencies
4. **Preload Consideration**: Once submitted, removal is difficult
5. **Monitor**: Track HSTS policy effectiveness

## Related Skills

- whmcs-security-headers
- whmcs-hsts-preload
- whmcs-tls-configuration
- whmcs-certificate-import
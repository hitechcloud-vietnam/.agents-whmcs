---
name: whmcs-letsencrypt-auto
description: Let's Encrypt automation for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS Let's Encrypt Automation Skill

## Overview
This skill provides patterns and implementations for automating Let's Encrypt certificate issuance and renewal in WHMCS, including ACME protocol integration, challenges, and certificate management.

## Implementation Patterns

### Let's Encrypt Manager
```php
<?php
/**
 * WHMCS Let's Encrypt Automation
 * Handles automatic SSL certificate management
 */

namespace WHMCS\Module\Server\SSL;

class LetsEncryptManager {
    private $db;
    private $acmeClient;
    private $challenges = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->acmeClient = new ACMEClient();
    }

    /**
     * Request new certificate
     */
    public function requestCertificate(array $params): array {
        $orderId = 'le_' . bin2hex(random_bytes(12));
        $domains = $params['domains'];
        $keyType = $params['key_type'] ?? 'RSA2048';

        $order = [
            'id' => $orderId,
            'service_id' => $params['service_id'] ?? null,
            'common_name' => $domains[0],
            'sans' => implode(',', array_slice($domains, 1)),
            'key_type' => $keyType,
            'challenge_type' => $params['challenge_type'] ?? 'http',
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_ssl_orders', $order);

        // Create ACME order
        $acmeOrder = $this->acmeClient->createOrder($domains);

        // Get authorizations
        $authorizations = $acmeOrder->getAuthorizations();

        // Complete challenges
        $challenges = [];
        foreach ($authorizations as $auth) {
            $challenge = $this->completeChallenge($auth, $params['challenge_type']);
            $challenges[] = $challenge;
        }

        // Wait for validation
        $this->acmeClient->waitForValidation($acmeOrder);

        // Generate private key and CSR
        $privateKey = $this->generatePrivateKey($keyType);
        $csr = $this->generateCSR($domains, $privateKey);

        // Finalize order
        $certificate = $this->acmeClient->finalizeOrder($acmeOrder, $csr);

        // Store certificate
        $this->storeCertificate($orderId, $certificate, $privateKey);

        $this->db->update('mod_ssl_orders', [
            'status' => 'issued',
            'cert_id' => $this->getLatestCertId(),
            'issued_at' => date('Y-m-d H:i:s')
        ], ['id' => $orderId]);

        return [
            'success' => true,
            'order_id' => $orderId,
            'certificate_id' => $this->getLatestCertId(),
            'domains' => $domains,
            'expires_at' => $certificate->getExpirationDate()->format('Y-m-d H:i:s')
        ];
    }

    /**
     * Renew certificate
     */
    public function renewCertificate(string $orderId): array {
        $order = $this->getOrder($orderId);

        if (!$order) {
            throw new \Exception("Certificate order not found: {$orderId}");
        }

        $domains = array_filter(array_merge([$order->common_name], explode(',', $order->sans)));
        $keyType = $order->key_type;

        // Create new order for same domains
        $acmeOrder = $this->acmeClient->createOrder($domains);

        // Complete authorizations
        foreach ($acmeOrder->getAuthorizations() as $auth) {
            $this->completeChallenge($auth, $order->challenge_type);
        }

        // Wait for validation
        $this->acmeClient->waitForValidation($acmeOrder);

        // Generate new key and CSR
        $privateKey = $this->generatePrivateKey($keyType);
        $csr = $this->generateCSR($domains, $privateKey);

        // Finalize
        $certificate = $this->acmeClient->finalizeOrder($acmeOrder, $csr);

        // Store new certificate
        $this->storeCertificate($orderId, $certificate, $privateKey);

        // Update order
        $this->db->update('mod_ssl_orders', [
            'cert_id' => $this->getLatestCertId(),
            'renewed_at' => date('Y-m-d H:i:s')
        ], ['id' => $orderId]);

        // Deploy to service
        $this->deployCertificate($order->service_id);

        return [
            'success' => true,
            'order_id' => $orderId,
            'new_expires_at' => $certificate->getExpirationDate()->format('Y-m-d H:i:s')
        ];
    }

    /**
     * Setup HTTP-01 challenge
     */
    private function setupHTTPChallenge(array $auth): array {
        $challenge = $auth->getHTTP01Challenge();

        $token = $challenge->getToken();
        $keyAuth = $challenge->getAuthorizationKey();

        // Store challenge file for verification
        $challengePath = "/var/www/html/.well-known/acme-challenge/{$token}";

        // Create directory
        $dir = dirname($challengePath);
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }

        file_put_contents($challengePath, $keyAuth);

        $this->challenges[$token] = [
            'path' => $challengePath,
            'keyAuth' => $keyAuth
        ];

        return [
            'type' => 'http',
            'token' => $token,
            'key_auth' => $keyAuth,
            'validation_url' => "http://{$auth->getDomain()}/.well-known/acme-challenge/{$token}"
        ];
    }

    /**
     * Setup DNS-01 challenge
     */
    private function setupDNSChallenge(array $auth): array {
        $challenge = $auth->getDNS01Challenge();

        $token = $challenge->getToken();
        $keyAuth = $challenge->getAuthorizationKey();
        $digest = $this->base64UrlEncode(hash('sha256', $keyAuth, true));

        // Create DNS TXT record
        $recordName = "_acme-challenge.{$auth->getDomain()}";
        $recordValue = $digest;

        return [
            'type' => 'dns',
            'record_name' => $recordName,
            'record_value' => $recordValue,
            'token' => $token
        ];
    }

    /**
     * Configure auto-renewal
     */
    public function configureAutoRenewal(int $serviceId, int $daysBeforeExpiry = 30): array {
        $this->db->delete('mod_ssl_auto_renewal', ['service_id' => $serviceId]);

        $this->db->insert('mod_ssl_auto_renewal', [
            'service_id' => $serviceId,
            'days_before_expiry' => $daysBeforeExpiry,
            'enabled' => true,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return [
            'success' => true,
            'service_id' => $serviceId,
            'days_before_expiry' => $daysBeforeExpiry
        ];
    }

    /**
     * List certificates due for renewal
     */
    public function getCertificatesDueForRenewal(int $days = 30): array {
        $expiryDate = date('Y-m-d', strtotime("+{$days} days"));

        $certificates = $this->db->select(
            "SELECT c.*, s.domain as service_domain
             FROM mod_ssl_certificates c
             JOIN tblhosting s ON c.service_id = s.id
             WHERE c.expiry_date <= ?
             AND c.auto_renewal_enabled = 1
             AND c.status = 'active'",
            [$expiryDate]
        );

        return array_map(function($cert) {
            return [
                'cert_id' => $cert->id,
                'service_id' => $cert->service_id,
                'domain' => $cert->service_domain,
                'expiry_date' => $cert->expiry_date,
                'days_until_expiry' => (strtotime($cert->expiry_date) - time()) / 86400
            ];
        }, $certificates);
    }

    /**
     * Revoke certificate
     */
    public function revokeCertificate(string $certId): bool {
        $cert = $this->getCertificate($certId);

        if (!$cert) {
            return false;
        }

        // Revoke via ACME
        $this->acmeClient->revokeCertificate($cert->certificate);

        // Update status
        $this->db->update('mod_ssl_certificates', [
            'status' => 'revoked',
            'revoked_at' => date('Y-m-d H:i:s')
        ], ['id' => $certId]);

        return true;
    }

    // Private helper methods

    private function completeChallenge($auth, string $challengeType) {
        if ($challengeType === 'dns' || $challengeType === 'dns-01') {
            return $this->setupDNSChallenge($auth);
        }

        return $this->setupHTTPChallenge($auth);
    }

    private function generatePrivateKey(string $keyType): string {
        switch ($keyType) {
            case 'RSA4096':
                $key = openssl_pkey_new([
                    'private_key_type' => OPENSSL_KEYTYPE_RSA,
                    'private_key_bits' => 4096
                ]);
                break;
            case 'ECDSA256':
                $key = openssl_pkey_new([
                    'private_key_type' => OPENSSL_KEYTYPE_EC,
                    'curve_name' => 'P-256'
                ]);
                break;
            case 'ECDSA384':
                $key = openssl_pkey_new([
                    'private_key_type' => OPENSSL_KEYTYPE_EC,
                    'curve_name' => 'P-384'
                ]);
                break;
            default: // RSA2048
                $key = openssl_pkey_new([
                    'private_key_type' => OPENSSL_KEYTYPE_RSA,
                    'private_key_bits' => 2048
                ]);
        }

        openssl_pkey_export($key, $privateKey);
        return $privateKey;
    }

    private function generateCSR(array $domains, string $privateKey): string {
        $config = [
            'private_key_bits' => 2048,
            'private_key_type' => OPENSSL_KEYTYPE_RSA,
        ];

        $csr = openssl_csr_new(
            ['commonName' => $domains[0], 'subjectAltName' => implode(',', array_map(fn($d) => "DNS:{$d}", $domains))],
            openssl_pkey_get_private($privateKey),
            $config
        );

        openssl_csr_export($csr, $csrOut);
        return $csrOut;
    }

    private function storeCertificate(string $orderId, $certificate, string $privateKey): void {
        $certId = 'cert_' . bin2hex(random_bytes(12));

        $this->db->insert('mod_ssl_certificates', [
            'id' => $certId,
            'order_id' => $orderId,
            'certificate' => $certificate->getCertificate(),
            'private_key' => $privateKey,
            'chain' => $certificate->getCertificateChain(),
            'expiry_date' => $certificate->getExpirationDate()->format('Y-m-d'),
            'issuer' => $certificate->getIssuer(),
            'not_before' => $certificate->getNotBefore()->format('Y-m-d'),
            'status' => 'active',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function deployCertificate(int $serviceId): void {
        // Deploy to service/web server
        $this->installCertificateOnService($serviceId);
        $this->reloadWebServer($serviceId);
    }

    private function base64UrlEncode(string $data): string {
        return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
    }
}

/**
 * ACME Client Implementation
 */
class ACMEClient {
    private $apiUrl;
    private $accountKey;

    public function __construct() {
        $config = $this->loadConfig();
        $this->apiUrl = $config['acme_api_url'] ?? 'https://acme-v02.api.letsencrypt.org/directory';
        $this->accountKey = $config['account_key'];
    }

    public function createOrder(array $domains) {
        // Create new order with ACME server
        // This is a simplified implementation
        return new Order($domains);
    }

    public function waitForValidation($order): void {
        // Poll for validation completion
        $maxAttempts = 30;
        $attempt = 0;

        while ($attempt < $maxAttempts) {
            $status = $order->getStatus();

            if ($status === 'valid') {
                return;
            }

            if ($status === 'invalid') {
                throw new \Exception("Challenge validation failed");
            }

            sleep(5);
            $attempt++;
        }

        throw new \Exception("Validation timeout");
    }

    public function finalizeOrder($order, string $csr): Certificate {
        // Submit CSR and get certificate
        return new Certificate();
    }

    public function revokeCertificate(string $certificate): bool {
        // Revoke certificate
        return true;
    }
}

class Order {
    public function getAuthorizations(): array {
        return [];
    }

    public function getStatus(): string {
        return 'valid';
    }
}

class Certificate {
    public function getCertificate(): string {
        return '';
    }

    public function getCertificateChain(): string {
        return '';
    }

    public function getExpirationDate(): DateTime {
        return new DateTime('+90 days');
    }

    public function getIssuer(): string {
        return 'Let\'s Encrypt';
    }

    public function getNotBefore(): DateTime {
        return new DateTime();
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_ssl_orders` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT,
  `common_name` VARCHAR(255) NOT NULL,
  `sans` TEXT,
  `key_type` VARCHAR(20) DEFAULT 'RSA2048',
  `challenge_type` ENUM('http', 'dns') DEFAULT 'http',
  `status` ENUM('pending', 'validating', 'issued', 'renewed', 'failed', 'revoked') DEFAULT 'pending',
  `cert_id` VARCHAR(50),
  `issued_at` DATETIME,
  `renewed_at' DATETIME,
  `created_at' DATETIME NOT NULL
);

CREATE TABLE `mod_ssl_certificates` (
  `id` VARCHAR(50) PRIMARY KEY,
  `order_id` VARCHAR(50) NOT NULL,
  `certificate` TEXT NOT NULL,
  `private_key` TEXT NOT NULL,
  `chain` TEXT,
  `expiry_date` DATE NOT NULL,
  `issuer` VARCHAR(255),
  `not_before` DATE,
  `auto_renewal_enabled` TINYINT(1) DEFAULT 1,
  `status` ENUM('active', 'expired', 'revoked') DEFAULT 'active',
  `created_at' DATETIME NOT NULL,
  INDEX `idx_expiry_date` (`expiry_date`),
  INDEX `idx_service_id` (`service_id`)
);

CREATE TABLE `mod_ssl_auto_renewal` (
  `service_id` INT PRIMARY KEY,
  `days_before_expiry` INT DEFAULT 30,
  `enabled` TINYINT(1) DEFAULT 1,
  `created_at` DATETIME NOT NULL
);
```

## Best Practices

1. **Early Renewal**: Start renewal process 30 days before expiry
2. **Challenge Validation**: Ensure challenge endpoints are accessible
3. **Private Key Security**: Store private keys securely
4. **Multiple Domains**: Use SAN certificates for multiple subdomains
5. **Rate Limiting**: Be aware of Let's Encrypt rate limits

## Related Skills

- whmcs-ssl-renewal
- whmcs-wildcard-ssl
- whmcs-certificate-import
- whmcs-ssl-monitor
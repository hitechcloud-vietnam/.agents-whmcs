---
name: whmcs-multi-domain-ssl
description: SAN certificate setup for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS Multi-Domain (SAN) Certificate Skill

## Overview
This skill provides patterns and implementations for managing Subject Alternative Name (SAN) certificates in WHMCS, including multi-domain coverage, additional domain management, and unified renewal.

## Implementation Patterns

### Multi-Domain Certificate Manager
```php
<?php
/**
 * WHMCS Multi-Domain SSL (SAN Certificates)
 * Handles SAN/UCC certificates
 */

namespace WHMCS\Module\Server\SSL;

class MultiDomainCertificateManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Request SAN certificate
     */
    public function requestSANCertificate(array $params): array {
        $orderId = 'san_' . bin2hex(random_bytes(12));

        $primaryDomain = $params['primary_domain'];
        $additionalDomains = $params['additional_domains'] ?? [];
        $allDomains = array_unique(array_merge([$primaryDomain], $additionalDomains));

        $order = [
            'id' => $orderId,
            'service_id' => $params['service_id'],
            'primary_domain' => $primaryDomain,
            'all_domains' => json_encode($allDomains),
            'domain_count' => count($allDomains),
            'key_type' => $params['key_type'] ?? 'RSA2048',
            'challenge_type' => $params['challenge_type'] ?? 'http',
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_san_certificates', $order);

        // Create certificate order
        $acmeClient = new ACMEClient();
        $acmeOrder = $acmeClient->createOrder($allDomains);

        // Complete challenges for all domains
        $challenges = [];
        foreach ($acmeOrder->getAuthorizations() as $auth) {
            $challenge = $auth->getHTTP01Challenge();
            $token = $challenge->getToken();
            $keyAuth = $challenge->getAuthorizationKey();

            // Place challenge file
            $this->placeChallengeFile($auth->getDomain(), $token, $keyAuth);

            $challenges[] = [
                'domain' => $auth->getDomain(),
                'type' => 'http-01',
                'token' => $token
            ];
        }

        $this->db->update('mod_san_certificates', [
            'challenges' => json_encode($challenges)
        ], ['id' => $orderId]);

        // Wait for validation
        $acmeClient->waitForValidation($acmeOrder);

        // Generate CSR
        $privateKey = $this->generatePrivateKey($params['key_type'] ?? 'RSA2048');
        $csr = $this->generateCSR($allDomains, $privateKey);

        // Finalize order
        $certificate = $acmeClient->finalizeOrder($acmeOrder, $csr);

        // Store certificate
        $certId = $this->storeCertificate($orderId, $certificate, $privateKey);

        $this->db->update('mod_san_certificates', [
            'cert_id' => $certId,
            'status' => 'issued',
            'issued_at' => date('Y-m-d H:i:s')
        ], ['id' => $orderId]);

        return [
            'success' => true,
            'order_id' => $orderId,
            'cert_id' => $certId,
            'primary_domain' => $primaryDomain,
            'total_domains' => count($allDomains),
            'domains' => $allDomains
        ];
    }

    /**
     * Add domain to existing SAN certificate
     */
    public function addDomain(string $orderId, string $newDomain): array {
        $sanCert = $this->getSANCertificate($orderId);

        if (!$sanCert) {
            throw new \Exception("SAN certificate not found: {$orderId}");
        }

        if ($sanCert->status !== 'issued') {
            throw new \Exception("Cannot add domain to non-issued certificate");
        }

        $existingDomains = json_decode($sanCert->all_domains, true);

        if (in_array($newDomain, $existingDomains)) {
            throw new \Exception("Domain already included in certificate");
        }

        // Reissue with new domain
        $allDomains = array_merge($existingDomains, [$newDomain]);

        // Create new order
        $acmeClient = new ACMEClient();
        $acmeOrder = $acmeClient->createOrder($allDomains);

        // Complete challenges
        foreach ($acmeOrder->getAuthorizations() as $auth) {
            $challenge = $auth->getHTTP01Challenge();
            $this->placeChallengeFile($auth->getDomain(), $challenge->getToken(), $challenge->getAuthorizationKey());
        }

        $acmeClient->waitForValidation($acmeOrder);

        // Generate new CSR
        $privateKey = $this->generatePrivateKey($sanCert->key_type);
        $csr = $this->generateCSR($allDomains, $privateKey);
        $certificate = $acmeClient->finalizeOrder($acmeOrder, $csr);

        // Store new certificate
        $newCertId = $this->storeCertificate($orderId, $certificate, $privateKey);

        // Update order
        $this->db->update('mod_san_certificates', [
            'cert_id' => $newCertId,
            'all_domains' => json_encode($allDomains),
            'domain_count' => count($allDomains),
            'reissued_at' => date('Y-m-d H:i:s')
        ], ['id' => $orderId]);

        return [
            'success' => true,
            'order_id' => $orderId,
            'new_cert_id' => $newCertId,
            'domains' => $allDomains
        ];
    }

    /**
     * Remove domain from SAN certificate
     */
    public function removeDomain(string $orderId, string $domain): array {
        $sanCert = $this->getSANCertificate($orderId);

        if (!$sanCert) {
            throw new \Exception("SAN certificate not found");
        }

        $existingDomains = json_decode($sanCert->all_domains, true);

        if (($key = array_search($domain, $existingDomains)) !== false) {
            unset($existingDomains[$key]);
        }

        if (count($existingDomains) < 1) {
            throw new \Exception("SAN certificate must have at least one domain");
        }

        // Reissue with remaining domains
        $acmeClient = new ACMEClient();
        $acmeOrder = $acmeClient->createOrder(array_values($existingDomains));

        foreach ($acmeOrder->getAuthorizations() as $auth) {
            $challenge = $auth->getHTTP01Challenge();
            $this->placeChallengeFile($auth->getDomain(), $challenge->getToken(), $challenge->getAuthorizationKey());
        }

        $acmeClient->waitForValidation($acmeOrder);

        $privateKey = $this->generatePrivateKey($sanCert->key_type);
        $csr = $this->generateCSR(array_values($existingDomains), $privateKey);
        $certificate = $acmeClient->finalizeOrder($acmeOrder, $csr);

        $newCertId = $this->storeCertificate($orderId, $certificate, $privateKey);

        $this->db->update('mod_san_certificates', [
            'cert_id' => $newCertId,
            'all_domains' => json_encode(array_values($existingDomains)),
            'domain_count' => count($existingDomains),
            'reissued_at' => date('Y-m-d H:i:s')
        ], ['id' => $orderId]);

        return [
            'success' => true,
            'order_id' => $orderId,
            'remaining_domains' => array_values($existingDomains)
        ];
    }

    /**
     * Get SAN certificate details
     */
    public function getDetails(string $orderId): array {
        $sanCert = $this->getSANCertificate($orderId);

        if (!$sanCert) {
            throw new \Exception("SAN certificate not found");
        }

        $domains = json_decode($sanCert->all_domains, true);

        return [
            'order_id' => $orderId,
            'primary_domain' => $sanCert->primary_domain,
            'all_domains' => $domains,
            'domain_count' => $sanCert->domain_count,
            'status' => $sanCert->status,
            'expiry_date' => $sanCert->expiry_date,
            'issued_at' => $sanCert->issued_at
        ];
    }

    // Private helper methods

    private function placeChallengeFile(string $domain, string $token, string $keyAuth): void {
        $path = "/var/www/vhosts/{$domain}/.well-known/acme-challenge/{$token}";

        $dir = dirname($path);
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }

        file_put_contents($path, $keyAuth);
    }

    private function generateCSR(array $domains, string $privateKey): string {
        $config = [
            'private_key_bits' => 2048,
            'private_key_type' => OPENSSL_KEYTYPE_RSA
        ];

        $sans = array_map(fn($d) => "DNS:{$d}", $domains);

        $csr = openssl_csr_new(
            ['commonName' => $domains[0], 'subjectAltName' => implode(',', $sans)],
            openssl_pkey_get_private($privateKey),
            $config
        );

        openssl_csr_export($csr, $csrOut);
        return $csrOut;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_san_certificates` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `primary_domain` VARCHAR(255) NOT NULL,
  `all_domains` TEXT NOT NULL,
  `domain_count` INT NOT NULL,
  `key_type` VARCHAR(20) DEFAULT 'RSA2048',
  `challenges` TEXT,
  `cert_id` VARCHAR(50),
  `status` ENUM('pending', 'issued', 'reissued') DEFAULT 'pending',
  `issued_at` DATETIME,
  `reissued_at` DATETIME,
  `created_at' DATETIME NOT NULL,
  INDEX `idx_service_id` (`service_id`)
);
```

## Best Practices

1. **Plan Ahead**: Include all known domains in initial request
2. **Cost Efficiency**: Multiple domains in one cert is cheaper than separate certs
3. **Add/Remove**: Manage domains dynamically as needed
4. **Coverage**: Ensure all subdomains are included
5. **Unified Management**: Single renewal for all domains

## Related Skills

- whmcs-letsencrypt-auto
- whmcs-wildcard-ssl
- whmcs-ssl-renewal
- whmcs-certificate-import
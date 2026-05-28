# WHMCS SSL Offload Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build SSL certificate offload/termination modules for load balancers.

## SSL Offload Module Structure

```php
<?php
/**
 * SSL Offload Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'SSL Offload',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'Parameters' => ['api_key', 'region'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'CertificateType' => [
            'Type' => 'dropdown',
            'Options' => 'shared,dedicated,wildcard',
            'Default' => 'shared',
        ],
        'TLSVersion' => [
            'Type' => 'dropdown',
            'Options' => '1.0,1.1,1.2,1.3',
            'Default' => '1.2',
        ],
        'HSTSEnabled' => [
            'Type' => 'yesno',
            'Description' => 'Enable HSTS header',
        ],
        'OCSPStapling' => [
            'Type' => 'yesno',
            'Description' => 'Enable OCSP stapling',
        ],
    ];
}
```

## SSL Offload Operations

```php
function {module}_CreateAccount(array $params): string {
    $offload = $this->api->createSSLOffload([
        'certificate_type' => $params['configoption1'],
        'tls_versions' => explode(',', $params['configoption2']),
        'hsts' => $params['configoption3'] === 'on',
        'ocsp_stapling' => $params['configoption4'] === 'on',
    ]);

    Capsule::table('mod_ssl_offload')->insert([
        'service_id' => $params['serviceid'],
        'offload_id' => $offload['id'],
        'frontend_ip' => $offload['frontend_ip'],
        'certificate_type' => $params['configoption1'],
        'tls_versions' => $params['configoption2'],
        'hsts' => $params['configoption3'] === 'on',
        'ocsp_stapling' => $params['configoption4'] === 'on',
        'status' => 'active',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $offload = Capsule::table('mod_ssl_offload')
        ->where('service_id', $params['serviceid'])
        ->first();

    if ($offload) {
        $this->api->deleteSSLOffload($offload->offload_id);
        Capsule::table('mod_ssl_offload')
            ->where('id', $offload->id)
            ->delete();
    }

    return 'success';
}
```

## Certificate Management

```php
public function uploadCertificate(int $serviceId, string $cert, string $key, ?string $chain = null): bool {
    $offload = Capsule::table('mod_ssl_offload')
        ->where('service_id', $serviceId)
        ->first();

    $result = $this->api->uploadCertificate($offload->offload_id, [
        'certificate' => $cert,
        'private_key' => $key,
        'ca_bundle' => $chain,
    ]);

    Capsule::table('mod_ssl_offload')
        ->where('service_id', $serviceId)
        ->update([
            'certificate_id' => $result['cert_id'],
            'expires_at' => $result['expires_at'],
        ]);

    return true;
}

public function getCertificateInfo(int $serviceId): array {
    $offload = Capsule::table('mod_ssl_offload')
        ->where('service_id', $serviceId)
        ->first();

    if (!$offload->certificate_id) {
        return ['type' => 'shared', 'valid' => true];
    }

    $cert = $this->api->getCertificate($offload->certificate_id);

    return [
        'subject' => $cert['subject'],
        'issuer' => $cert['issuer'],
        'valid_from' => $cert['valid_from'],
        'valid_until' => $cert['valid_until'],
        'days_remaining' => $cert['days_remaining'],
        'serial' => $cert['serial'],
    ];
}

public function autoRenew(int $serviceId): bool {
    $offload = Capsule::table('mod_ssl_offload')
        ->where('service_id', $serviceId)
        ->first();

    $certInfo = $this->getCertificateInfo($serviceId);

    if ($certInfo['days_remaining'] <= 30) {
        $newCert = $this->api->renewCertificate($offload->certificate_id);

        Capsule::table('mod_ssl_offload')
            ->where('service_id', $serviceId)
            ->update([
                'certificate_id' => $newCert['id'],
                'expires_at' => $newCert['expires_at'],
            ]);

        return true;
    }

    return false;
}
```

## Client Area

```php
function {module}_ClientArea(array $params): array {
    $offload = Capsule::table('mod_ssl_offload')
        ->where('service_id', $params['serviceid'])
        ->first();

    $certInfo = $this->getCertificateInfo($params['serviceid']);

    return [
        'pagetitle' => 'SSL Offload',
        'templatefile' => 'templates/ssl_offload_clientarea',
        'vars' => [
            'offload' => $offload,
            'certificate' => $certInfo,
            'config' => [
                'tls_versions' => $offload->tls_versions,
                'hsts' => $offload->hsts,
                'ocsp_stapling' => $offload->ocsp_stapling,
            ],
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-load-balancer
- whmcs-ssl-certificate-module
# WHMCS SSL Renewal Workflow

## Purpose

Automated procedures for managing SSL certificate lifecycle in WHMCS, including automatic renewal, installation, and verification of SSL certificates.

## Prerequisites

- WHMCS provisioning module for SSL
- SSL provider API credentials (Let's Encrypt, DigiCert, etc.)
- Server access for certificate installation
- Automated cron configured

## Workflow Steps

### Step 1: Configure SSL Provisioning Module

Set up SSL certificate module:

```php
<?php
// modules/servers/ssl_provider/ssl_provider.php

class SSL_Provider_Module
{
    public function getConfigArray()
    {
        return [
            'APIKey' => [
                'Type' => 'text',
                'Size' => '40',
                'Description' => 'Enter your SSL Provider API Key',
            ],
            'APIEndpoint' => [
                'Type' => 'text',
                'Size' => '60',
                'Default' => 'https://api.sslprovider.com/v1',
                'Description' => 'API endpoint URL',
            ],
            'AutoVerify' => [
                'Type' => 'yesno',
                'Description' => 'Automatically verify certificates after issuance',
            ],
            'WebRoot' => [
                'Type' => 'text',
                'Size' => '60',
                'Default' => '/var/www',
                'Description' => 'Web root path for ACME challenges',
            ],
        ];
    }
    
    public function provision($params)
    {
        $orderParams = [
            'product' => $params['configoption1'], // e.g., 'positivessl'
            'fqdn' => $params['domain'],
            'period' => $params['configoptions']['Period'] ?? 12,
            'csr' => $params['csr'],
            'approver_email' => $params['approveremail'],
        ];
        
        // Create order with provider
        $response = $this->apiCall('orders/create', $orderParams);
        
        if ($response['success']) {
            return [
                'success' => true,
                'order_id' => $response['order_id'],
                'status' => 'pending_verification',
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['error'] ?? 'Order creation failed',
        ];
    }
    
    public function requestCertificate($params)
    {
        $response = $this->apiCall('certificates/request', [
            'order_id' => $params['order_id'],
            'verification_method' => 'http',
            'verification_data' => [
                'http_path' => '/.well-known/pki-validation/',
                'http_body' => $params['dcv_token'],
            ],
        ]);
        
        return [
            'success' => $response['success'],
            'crt' => $response['certificate'] ?? null,
            'ca' => $response['ca_bundle'] ?? null,
        ];
    }
    
    public function renew($params)
    {
        // Renew certificate before expiry
        $response = $this->apiCall('orders/renew', [
            'order_id' => $params['order_id'],
        ]);
        
        return [
            'success' => $response['success'],
            'renewal_order_id' => $response['renewal_order_id'] ?? null,
        ];
    }
    
    public function revoke($params)
    {
        $response = $this->apiCall('certificates/revoke', [
            'certificate_id' => $params['certificate_id'],
            'reason' => $params['revocation_reason'] ?? 'unspecified',
        ]);
        
        return ['success' => $response['success']];
    }
}
```

### Step 2: Create Automatic Renewal Hook

Automate certificate renewals:

```php
<?php
// modules/custom/hooks/ssl_renewal_hooks.php

use WHMCS\Carbon;
use WHMCS\Billing\Certificate;

// Hook: Before certificate renewal
add_hook('SSLCertificateRenewalPre', 1, function($params) {
    $service = $params['service'];
    $domain = $service->domain;
    
    // Check if renewal is needed
    $expiryDate = Carbon::parse($service->nextduedate);
    $daysUntilExpiry = Carbon::now()->diffInDays($expiryDate);
    
    // Renew 30 days before expiry
    if ($daysUntilExpiry > 30) {
        return [
            'abort' => true,
            'message' => "Renewal not yet needed. {$daysUntilExpiry} days until expiry.",
        ];
    }
    
    // Check for payment issues
    $hasUnpaidInvoices = Capsule::table('tblinvoiceitems')
        ->join('tblinvoices', 'tblinvoiceitems.invoiceid', '=', 'tblinvoices.id')
        ->where('tblinvoiceitems.relid', $service->id)
        ->where('tblinvoices.status', 'Unpaid')
        ->exists();
    
    if ($hasUnpaidInvoices) {
        return [
            'abort' => true,
            'message' => 'Cannot renew: unpaid invoices exist',
        ];
    }
    
    return ['abort' => false];
});

// Hook: After certificate issuance
add_hook('SSLCertificateIssued', 1, function($params) {
    $certificate = $params['certificate'];
    $service = $params['service'];
    
    logActivity("SSL Certificate issued for {$service->domain}");
    
    // Send notification to client
    sendEmail(
        $service->userid,
        'SSL Certificate Issued',
        [
            'certificate_id' => $certificate['id'],
            'domain' => $service->domain,
            'expiry_date' => $certificate['expiry_date'],
            'download_url' => getCertificateDownloadUrl($certificate['id']),
        ]
    );
    
    // Update service due date
    Capsule::table('tblhosting')
        ->where('id', $service->id)
        ->update([
            'nextduedate' => Carbon::parse($certificate['expiry_date'])->format('Y-m-d'),
        ]);
});

// Hook: After certificate installation
add_hook('SSLCertificateInstalled', 1, function($params) {
    $domain = $params['domain'];
    $server = $params['server'];
    
    // Verify installation
    $verification = verifySSLCertificate($domain);
    
    if (!$verification['valid']) {
        logActivity("SSL verification failed for {$domain}: " . $verification['error']);
        
        sendAdminNotification(
            'SSL Installation Verification Failed',
            [
                'domain' => $domain,
                'server' => $server['hostname'],
                'error' => $verification['error'],
            ]
        );
    }
});
```

### Step 3: Set Up Renewal Cron Job

Automated renewal processing:

```php
<?php
// modules/custom/ssl_renewal_cron.php
// Run daily via cron: 0 2 * * *

require_once __DIR__ . '/init.php';

class SSLRenewalCron
{
    private $renewalDaysBefore = 30;
    private $processed = 0;
    private $errors = [];
    
    public function run()
    {
        echo "Starting SSL renewal check...\n";
        
        $certificates = $this->getCertificatesNeedingRenewal();
        echo "Found " . count($certificates) . " certificates to check\n";
        
        foreach ($certificates as $certificate) {
            $this->processCertificate($certificate);
        }
        
        $this->report();
    }
    
    private function getCertificatesNeedingRenewal()
    {
        $renewalDate = Carbon::now()->addDays($this->renewalDaysBefore)->toDateString();
        
        return Capsule::table('tblhosting')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->join('tblssl', 'tblhosting.id', '=', 'tblssl.relid')
            ->where('tblproducts.type', 'ssl')
            ->where('tblhosting.domainstatus', 'Active')
            ->where('tblssl.expirydate', '<=', $renewalDate)
            ->whereNotIn('tblhosting.domainstatus', ['Suspended', 'Terminated', 'Cancelled'])
            ->select([
                'tblhosting.id as service_id',
                'tblhosting.domain',
                'tblhosting.server',
                'tblclients.id as client_id',
                'tblclients.email',
                'tblssl.id as ssl_id',
                'tblssl.expirydate',
            ])
            ->get();
    }
    
    private function processCertificate($certificate)
    {
        try {
            echo "Processing {$certificate->domain}...\n";
            
            // Check if renewal is already in progress
            $pendingRenewal = Capsule::table('tblsslorders')
                ->where('service_id', $certificate->service_id)
                ->whereIn('status', ['pending', 'processing'])
                ->exists();
            
            if ($pendingRenewal) {
                echo "Renewal already in progress for {$certificate->domain}\n";
                return;
            }
            
            // Create renewal order
            $result = $this->createRenewalOrder($certificate);
            
            if ($result['success']) {
                $this->processed++;
                echo "Renewal initiated for {$certificate->domain}\n";
            } else {
                $this->errors[] = [
                    'domain' => $certificate->domain,
                    'error' => $result['error'],
                ];
            }
            
        } catch (\Exception $e) {
            $this->errors[] = [
                'domain' => $certificate->domain,
                'error' => $e->getMessage(),
            ];
            logActivity("SSL renewal error for {$certificate->domain}: " . $e->getMessage());
        }
    }
    
    private function createRenewalOrder($certificate)
    {
        // Create invoice for renewal
        $invoiceId = createInvoice([
            'client_id' => $certificate->client_id,
            'items' => [
                [
                    'description' => "SSL Certificate Renewal - {$certificate->domain}",
                    'amount' => getServiceRenewalPrice($certificate->service_id),
                ]
            ]
        ]);
        
        // Auto-pay if client has valid payment method
        $autoPayResult = autoPayInvoice($invoiceId);
        
        if ($autoPayResult['paid']) {
            // Provision immediately
            return $this->provisionRenewal($certificate);
        }
        
        // Wait for payment
        return [
            'success' => true,
            'message' => 'Renewal invoice created, awaiting payment',
            'invoice_id' => $invoiceId,
        ];
    }
    
    private function provisionRenewal($certificate)
    {
        $params = [
            'serviceid' => $certificate->service_id,
            'model' => Capsule::table('tblhosting')
                ->where('id', $certificate->service_id)
                ->first(),
        ];
        
        $result = Provisioning\Module::call(
            $certificate->server_type,
            'Renew',
            $params
        );
        
        return ['success' => $result['success']];
    }
    
    private function report()
    {
        echo "\n=== SSL Renewal Summary ===\n";
        echo "Processed: {$this->processed}\n";
        echo "Errors: " . count($this->errors) . "\n";
        
        if (!empty($this->errors)) {
            echo "\nErrors:\n";
            foreach ($this->errors as $error) {
                echo "  - {$error['domain']}: {$error['error']}\n";
            }
            
            // Send error report
            sendAdminNotification(
                'SSL Renewal Errors',
                ['errors' => $this->errors]
            );
        }
    }
}

// Run the cron
$renewal = new SSLRenewalCron();
$renewal->run();
```

### Step 4: Implement Let's Encrypt Integration

Free SSL with ACME protocol:

```php
<?php
// modules/custom/letsencrypt/LetsEncryptClient.php

namespace WHMCS\Custom\LetsEncrypt;

class LetsEncryptClient
{
    private $apiUrl = 'https://acme-v02.api.letsencrypt.org/directory';
    private $accountKey;
    
    public function __construct($accountKeyPath = null)
    {
        if ($accountKeyPath && file_exists($accountKeyPath)) {
            $this->accountKey = openssl_pkey_get_private(
                file_get_contents($accountKeyPath)
            );
        } else {
            $this->accountKey = $this->generateKeyPair();
        }
    }
    
    public function requestCertificate($domains, $webRoot)
    {
        // Step 1: Get certificate order
        $order = $this->createOrder($domains);
        
        // Step 2: Complete HTTP-01 challenges
        foreach ($order['challenges'] as $challenge) {
            if ($challenge['type'] === 'http-01') {
                $this->completeHttpChallenge($challenge, $webRoot);
            }
        }
        
        // Step 3: Poll for validation
        $this->pollValidation($order['orderUrl']);
        
        // Step 4: Generate CSR and finalize order
        $csr = $this->generateCSR($domains);
        $this->finalizeOrder($order['orderUrl'], $csr);
        
        // Step 5: Download certificate
        return $this->downloadCertificate($order['orderUrl']);
    }
    
    private function createOrder($domains)
    {
        $payload = [
            'identifiers' => array_map(function($domain) {
                return ['type' => 'dns', 'value' => $domain];
            }, $domains),
        ];
        
        $response = $this->signedRequest(
            $this->apiUrl . '/new-order',
            $payload
        );
        
        return [
            'orderUrl' => $response['url'],
            'challenges' => $response['authorizations'][0]['challenges'],
        ];
    }
    
    private function completeHttpChallenge($challenge, $webRoot)
    {
        $token = $challenge['token'];
        $keyAuth = $this->getKeyAuthorization($token);
        
        // Create challenge file
        $challengeDir = $webRoot . '/.well-known/acme-challenge/';
        if (!is_dir($challengeDir)) {
            mkdir($challengeDir, 0755, true);
        }
        
        file_put_contents($challengeDir . $token, $keyAuth);
        
        // Notify LE that we're ready
        $this->signedRequest($challenge['url'], [
            'keyAuthorization' => $keyAuth,
        ]);
    }
    
    private function getKeyAuthorization($token)
    {
        $thumbprint = $this->getJWKThumbprint();
        return "{$token}.{$thumbprint}";
    }
    
    private function generateCSR($domains)
    {
        $domain = $domains[0];
        $altNames = array_slice($domains, 1);
        
        $config = [
            'private_key_bits' => 2048,
            'private_key_type' => OPENSSL_KEYTYPE_RSA,
        ];
        
        $keyResource = openssl_pkey_new($config);
        
        $csrConfig = [
            'digest_alg' => 'sha256',
            'private_key_bits' => 2048,
            'private_key_type' => OPENSSL_KEYTYPE_RSA,
            'req_extensions' => 'v3_req',
            'config' => '/etc/ssl/openssl.cnf',
        ];
        
        $subject = [
            'commonName' => $domain,
        ];
        
        $altNameString = 'DNS:' . implode(',DNS:', $altNames);
        
        $csr = openssl_csr_new($subject, $keyResource, $csrConfig);
        openssl_csr_export($csr, $csrString);
        
        return $csrString;
    }
    
    private function downloadCertificate($orderUrl)
    {
        $response = $this->signedRequest($orderUrl . '/certificate', null, true);
        
        return [
            'certificate' => $response,
            'expires' => $this->getCertificateExpiry($response),
        ];
    }
    
    private function signedRequest($url, $payload, $binary = false)
    {
        // Implement ACME signed request
        // This is a simplified version
    }
}
```

### Step 5: Create SSL Status Dashboard

Admin monitoring interface:

```php
<?php
// admin/ssl_dashboard.php

require_once __DIR__ . '/../init.php';

if (!checkPermission('View SSL Certificates', true)) {
    exit('Access Denied');
}

$stats = getSSLStatistics();
$expiring = getExpiringCertificates(30);
$recentlyIssued = getRecentlyIssuedCertificates(7);

echo $twig->render('admin/ssl_dashboard.html', [
    'stats' => $stats,
    'expiring' => $expiring,
    'recently_issued' => $recentlyIssued,
]);

function getSSLStatistics()
{
    $total = Capsule::table('tblssl')->count();
    $active = Capsule::table('tblssl')->where('status', 'Active')->count();
    $expiring7 = getExpiringCount(7);
    $expiring30 = getExpiringCount(30);
    
    return [
        'total' => $total,
        'active' => $active,
        'expiring_7_days' => $expiring7,
        'expiring_30_days' => $expiring30,
    ];
}

function getExpiringCertificates($days)
{
    $date = Carbon::now()->addDays($days)->toDateString();
    
    return Capsule::table('tblssl')
        ->join('tblhosting', 'tblssl.relid', '=', 'tblhosting.id')
        ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
        ->where('tblssl.expirydate', '<=', $date)
        ->where('tblssl.expirydate', '>=', Carbon::now()->toDateString())
        ->select([
            'tblssl.id',
            'tblhosting.domain',
            'tblssl.expirydate',
            'tblclients.email',
        ])
        ->orderBy('tblssl.expirydate', 'asc')
        ->get();
}
```

## Verification Checklist

- [ ] SSL provisioning module configured
- [ ] API credentials set up
- [ ] Automatic renewal hooks implemented
- [ ] Daily cron job scheduled
- [ ] Certificate verification working
- [ ] Email notifications configured
- [ ] Admin dashboard accessible
- [ ] Failed renewal handling tested
- [ ] Installation verification active

## Related Skills and Documentation

- [WHMCS Webhook Automation](whmcs-webhook-automation-workflow.md)
- [WHMCS Provisioning Automation](whmcs-provisioning-automation-workflow.md)
- Let's Encrypt Documentation: https://letsencrypt.org/docs/
- ACME Protocol: https://datatracker.ietf.org/doc/html/rfc8555

## Notes

- Set renewal reminders at 30, 14, and 7 days
- Always verify certificate installation
- Monitor for failed verifications
- Keep logs of all renewal attempts
- Test renewal process quarterly
- Consider auto-installation on supported servers

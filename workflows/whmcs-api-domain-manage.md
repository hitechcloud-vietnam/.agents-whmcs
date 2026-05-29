# WHMCS API Domain Manage Workflow

## Purpose
Guide developers through managing domains via WHMCS API.

## Prerequisites
- WHMCS installation
- Domain/Registrar API access
- Registrar module configured
- PHP skills

## Steps

### Phase 1: Domain Operations

1. Domain registration
   ```php
   function registerDomain($clientId, $domain, $years): array {
       return localAPI('RegisterDomain', [
           'clientid' => $clientId,
           'domain' => $domain,
           'regperiod' => $years,
           'nameserver1' => 'ns1.example.com',
           'nameserver2' => 'ns2.example.com',
           'paymentmethod' => 'paypal',
       ]);
   }
   ```

2. Domain transfer
   ```php
   function transferDomain($clientId, $domain, $eppCode): array {
       return localAPI('TransferDomain', [
           'clientid' => $clientId,
           'domain' => $domain,
           'eppcode' => $eppCode,
           'transferopt' => 'preservelock',
           'paymentmethod' => 'paypal',
       ]);
   }
   ```

### Phase 2: Domain Management

1. Get nameservers
   ```php
   $nameservers = localAPI('GetNameservers', [
       'domain' => 'example.com',
   ]);
   ```

2. Update nameservers
   ```php
   localAPI('SaveNameservers', [
       'domain' => 'example.com',
       'ns1' => 'ns1.newhost.com',
       'ns2' => 'ns2.newhost.com',
   ]);
   ```

3. Domain lock
   ```php
   localAPI('SaveRegistrarLock', [
       'domain' => 'example.com',
       'lockenabled' => 'locked',
   ]);
   ```

### Phase 3: DNS Management

1. Get DNS records
   ```php
   $dnsRecords = localAPI('GetDNS', [
       'domain' => 'example.com',
   ]);
   ```

2. Save DNS records
   ```php
   localAPI('SaveDNS', [
       'domain' => 'example.com',
       'records' => [
           ['hostname' => '', 'type' => 'A', 'address' => '192.168.1.1', 'priority' => 0],
           ['hostname' => 'www', 'type' => 'A', 'address' => '192.168.1.1', 'priority' => 0],
       ],
   ]);
   ```

## Related Workflows
- whmcs-api-service-create
- whmcs-api-integration
- whmcs-api-domain-sync

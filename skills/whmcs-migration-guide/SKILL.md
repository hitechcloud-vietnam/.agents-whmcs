# WHMCS Migration Guide Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for migrating existing systems to WHMCS modules.

## When to Use

- Converting from another billing system
- Building import capabilities
- Moving from manual to automated

## Migration Patterns

### Data Import

```php
function importClientsFromCsv(string $filePath): array {
    $handle = fopen($filePath, 'r');
    $results = ['success' => 0, 'failed' => 0, 'errors' => []];

    while (($row = fgetcsv($handle)) !== false) {
        try {
            $clientData = [
                'firstname' => $row[0],
                'lastname' => $row[1],
                'email' => $row[2],
                'companyname' => $row[3] ?? '',
                'address1' => $row[4] ?? '',
                'city' => $row[5] ?? '',
                'country' => $row[6] ?? 'VN',
            ];

            $result = localAPI('CreateClient', $clientData);

            if ($result['result'] === 'success') {
                $results['success']++;
            } else {
                $results['failed']++;
                $results['errors'][] = $result['message'];
            }
        } catch (\Exception $e) {
            $results['failed']++;
            $results['errors'][] = $e->getMessage();
        }
    }

    fclose($handle);
    return $results;
}
```

### Service Migration

```php
function importServices(array $services): array {
    $results = ['success' => 0, 'failed' => 0];

    foreach ($services as $service) {
        try {
            // Create service via API
            $serviceParams = [
                'clientid' => $service['client_id'],
                'pid' => $service['product_id'],
                'domain' => $service['domain'],
                'billingcycle' => 'Monthly',
                'regdate' => date('Y-m-d'),
                'nextduedate' => date('Y-m-d'),
            ];

            $result = localAPI('CreateOrder', $serviceParams);

            if ($result['result'] === 'success') {
                $results['success']++;

                // Store external ID mapping
                saveMigrationMapping($service['external_id'], $result['orderid']);
            } else {
                $results['failed']++;
            }
        } catch (\Exception $e) {
            $results['failed']++;
        }
    }

    return $results;
}
```

### Domain Migration

```php
function importDomains(array $domains): array {
    foreach ($domains as $domain) {
        $domainParams = [
            'clientid' => $domain['client_id'],
            'domain' => $domain['domain'],
            'regperiod' => $domain['period'],
            'registrar' => 'modulename',
            'regdate' => $domain['registered_date'],
            'nextduedate' => $domain['expiry_date'],
        ];

        localAPI('RegisterDomain', $domainParams);
    }
}
```

## Checklist

- [ ] CSV import capability
- [ ] API import for bulk operations
- [ ] Progress tracking
- [ ] Error handling with rollback
- [ ] Mapping old IDs to new IDs

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-database-design
- whmcs-testing-qa
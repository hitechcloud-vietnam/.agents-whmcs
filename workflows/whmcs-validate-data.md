# WHMCS Data Validation Workflow

## Purpose
Validate data integrity and quality in WHMCS.

## Prerequisites
- WHMCS installation
- Admin access

## Step-by-Step Process

### Step 1: Create Data Validator

**Create hooks/data_validation.php:**
```php
<?php
/**
 * WHMCS Data Validation System
 */

use WHMCS\Database\Capsule;

class DataValidator {
    
    private $validationResults = [];
    
    /**
     * Run all validations
     */
    public function runAllValidations() {
        return [
            'clients' => $this->validateClients(),
            'services' => $this->validateServices(),
            'domains' => $this->validateDomains(),
            'invoices' => $this->validateInvoices(),
            'orphaned' => $this->findOrphanedRecords(),
            'integrity' => $this->checkDataIntegrity()
        ];
    }
    
    /**
     * Validate clients
     */
    public function validateClients() {
        $issues = [];
        
        // Check for missing required fields
        $requiredIssues = Capsule::select("
            SELECT id, email, firstname, lastname 
            FROM tblclients 
            WHERE email IS NULL 
               OR email = '' 
               OR firstname IS NULL 
               OR firstname = '' 
               OR lastname IS NULL 
               OR lastname = ''
        ");
        
        foreach ($requiredIssues as $issue) {
            $issues[] = [
                'type' => 'missing_required',
                'client_id' => $issue->id,
                'email' => $issue->email,
                'message' => 'Missing required fields'
            ];
        }
        
        // Check for invalid emails
        $invalidEmails = Capsule::select("
            SELECT id, email 
            FROM tblclients 
            WHERE email NOT LIKE '%@%.%'
        ");
        
        foreach ($invalidEmails as $issue) {
            $issues[] = [
                'type' => 'invalid_email',
                'client_id' => $issue->id,
                'email' => $issue->email,
                'message' => 'Invalid email format'
            ];
        }
        
        // Check for duplicate emails
        $duplicates = Capsule::select("
            SELECT email, COUNT(*) as count 
            FROM tblclients 
            GROUP BY email 
            HAVING COUNT(*) > 1
        ");
        
        foreach ($duplicates as $dup) {
            $clients = Capsule::table('tblclients')
                ->where('email', $dup->email)
                ->get(['id', 'email']);
            
            foreach ($clients as $client) {
                $issues[] = [
                    'type' => 'duplicate_email',
                    'client_id' => $client->id,
                    'email' => $client->email,
                    'message' => 'Duplicate email: ' . $dup->count . ' occurrences'
                ];
            }
        }
        
        // Check for invalid countries
        $validCountries = $this->getValidCountries();
        $invalidCountries = Capsule::select("
            SELECT id, country 
            FROM tblclients 
            WHERE country NOT IN ('" . implode("','", $validCountries) . "')
        ");
        
        foreach ($invalidCountries as $issue) {
            $issues[] = [
                'type' => 'invalid_country',
                'client_id' => $issue->id,
                'country' => $issue->country,
                'message' => 'Invalid country code'
            ];
        }
        
        return [
            'total_checked' => Capsule::table('tblclients')->count(),
            'issues_found' => count($issues),
            'issues' => $issues
        ];
    }
    
    /**
     * Validate services
     */
    public function validateServices() {
        $issues = [];
        
        // Check for services without client
        $orphanServices = Capsule::select("
            SELECT h.id, h.domain, h.userid 
            FROM tblhosting h 
            LEFT JOIN tblclients c ON h.userid = c.id 
            WHERE c.id IS NULL AND h.userid > 0
        ");
        
        foreach ($orphanServices as $issue) {
            $issues[] = [
                'type' => 'orphan_service',
                'service_id' => $issue->id,
                'domain' => $issue->domain,
                'message' => 'Service has invalid client ID'
            ];
        }
        
        // Check for services without product
        $noProduct = Capsule::select("
            SELECT h.id, h.domain, h.packageid 
            FROM tblhosting h 
            LEFT JOIN tblproducts p ON h.packageid = p.id 
            WHERE p.id IS NULL AND h.packageid > 0
        ");
        
        foreach ($noProduct as $issue) {
            $issues[] = [
                'type' => 'missing_product',
                'service_id' => $issue->id,
                'domain' => $issue->domain,
                'message' => 'Service references non-existent product'
            ];
        }
        
        // Check for negative amounts
        $negativeAmounts = Capsule::select("
            SELECT id, domain, firstpaymentamount, recurringamount 
            FROM tblhosting 
            WHERE firstpaymentamount < 0 OR recurringamount < 0
        ");
        
        foreach ($negativeAmounts as $issue) {
            $issues[] = [
                'type' => 'negative_amount',
                'service_id' => $issue->id,
                'domain' => $issue->domain,
                'message' => 'Service has negative payment amount'
            ];
        }
        
        // Check for invalid status
        $validStatuses = ['Active', 'Pending', 'Suspended', 'Cancelled', 'Terminated', 'FRAUD'];
        $invalidStatus = Capsule::select("
            SELECT id, domain, domainstatus 
            FROM tblhosting 
            WHERE domainstatus NOT IN ('" . implode("','", $validStatuses) . "')
        ");
        
        foreach ($invalidStatus as $issue) {
            $issues[] = [
                'type' => 'invalid_status',
                'service_id' => $issue->id,
                'domain' => $issue->domain,
                'status' => $issue->domainstatus,
                'message' => 'Invalid service status'
            ];
        }
        
        return [
            'total_checked' => Capsule::table('tblhosting')->count(),
            'issues_found' => count($issues),
            'issues' => $issues
        ];
    }
    
    /**
     * Validate domains
     */
    public function validateDomains() {
        $issues = [];
        
        // Check for domains without client
        $orphanDomains = Capsule::select("
            SELECT d.id, d.domain, d.userid 
            FROM tbldomains d 
            LEFT JOIN tblclients c ON d.userid = c.id 
            WHERE c.id IS NULL AND d.userid > 0
        ");
        
        foreach ($orphanDomains as $issue) {
            $issues[] = [
                'type' => 'orphan_domain',
                'domain_id' => $issue->id,
                'domain' => $issue->domain,
                'message' => 'Domain has invalid client ID'
            ];
        }
        
        // Check for invalid domain names
        $domains = Capsule::table('tbldomains')->get(['id', 'domain']);
        
        foreach ($domains as $domain) {
            if (!filter_var('http://' . $domain->domain, FILTER_VALIDATE_URL) && 
                !preg_match('/^[a-zA-Z0-9][a-zA-Z0-9-]*\.[a-zA-Z]{2,}$/', $domain->domain)) {
                $issues[] = [
                    'type' => 'invalid_domain',
                    'domain_id' => $domain->id,
                    'domain' => $domain->domain,
                    'message' => 'Invalid domain name format'
                ];
            }
        }
        
        return [
            'total_checked' => Capsule::table('tbldomains')->count(),
            'issues_found' => count($issues),
            'issues' => $issues
        ];
    }
    
    /**
     * Validate invoices
     */
    public function validateInvoices() {
        $issues = [];
        
        // Check for invoices without client
        $orphanInvoices = Capsule::select("
            SELECT i.id, i.invoicenumber, i.userid 
            FROM tblinvoices i 
            LEFT JOIN tblclients c ON i.userid = c.id 
            WHERE c.id IS NULL AND i.userid > 0
        ");
        
        foreach ($orphanInvoices as $issue) {
            $issues[] = [
                'type' => 'orphan_invoice',
                'invoice_id' => $issue->id,
                'number' => $issue->invoicenumber,
                'message' => 'Invoice has invalid client ID'
            ];
        }
        
        // Check for negative totals
        $negativeTotals = Capsule::select("
            SELECT id, invoicenumber, total 
            FROM tblinvoices 
            WHERE total < 0
        ");
        
        foreach ($negativeTotals as $issue) {
            $issues[] = [
                'type' => 'negative_total',
                'invoice_id' => $issue->id,
                'number' => $issue->invoicenumber,
                'message' => 'Invoice has negative total'
            ];
        }
        
        // Check for inconsistent totals
        $invoices = Capsule::select("
            SELECT i.id, i.invoicenumber, i.subtotal, i.tax, i.total,
                   COALESCE(i.tax2, 0) as tax2,
                   (i.subtotal + i.tax + COALESCE(i.tax2, 0)) as calculated_total
            FROM tblinvoices i
        ");
        
        foreach ($invoices as $inv) {
            $calculatedTotal = round($inv->subtotal + $inv->tax + $inv->tax2, 2);
            if (abs($calculatedTotal - $inv->total) > 0.01) {
                $issues[] = [
                    'type' => 'inconsistent_total',
                    'invoice_id' => $inv->id,
                    'number' => $inv->invoicenumber,
                    'stored_total' => $inv->total,
                    'calculated_total' => $calculatedTotal,
                    'message' => 'Invoice total does not match sum of components'
                ];
            }
        }
        
        return [
            'total_checked' => Capsule::table('tblinvoices')->count(),
            'issues_found' => count($issues),
            'issues' => $issues
        ];
    }
    
    /**
     * Find orphaned records
     */
    public function findOrphanedRecords() {
        $orphans = [];
        
        // Orphan services
        $orphans['services'] = Capsule::select("
            SELECT COUNT(*) as count 
            FROM tblhosting h 
            LEFT JOIN tblclients c ON h.userid = c.id 
            WHERE c.id IS NULL AND h.userid > 0
        ")[0]->count ?? 0;
        
        // Orphan domains
        $orphans['domains'] = Capsule::select("
            SELECT COUNT(*) as count 
            FROM tbldomains d 
            LEFT JOIN tblclients c ON d.userid = c.id 
            WHERE c.id IS NULL AND d.userid > 0
        ")[0]->count ?? 0;
        
        // Orphan invoices
        $orphans['invoices'] = Capsule::select("
            SELECT COUNT(*) as count 
            FROM tblinvoices i 
            LEFT JOIN tblclients c ON i.userid = c.id 
            WHERE c.id IS NULL AND i.userid > 0
        ")[0]->count ?? 0;
        
        // Orphan orders
        $orphans['orders'] = Capsule::select("
            SELECT COUNT(*) as count 
            FROM tblorders o 
            LEFT JOIN tblclients c ON o.userid = c.id 
            WHERE c.id IS NULL AND o.userid > 0
        ")[0]->count ?? 0;
        
        return $orphans;
    }
    
    /**
     * Check data integrity
     */
    public function checkDataIntegrity() {
        $checks = [];
        
        // Check table counts
        $checks['table_counts'] = [
            'clients' => Capsule::table('tblclients')->count(),
            'services' => Capsule::table('tblhosting')->count(),
            'domains' => Capsule::table('tbldomains')->count(),
            'invoices' => Capsule::table('tblinvoices')->count()
        ];
        
        // Check for UUIDs
        $noUuid = Capsule::table('tblclients')
            ->whereNull('uuid')
            ->orWhere('uuid', '')
            ->count();
        $checks['missing_uuid'] = $noUuid;
        
        return $checks;
    }
    
    /**
     * Get valid country codes
     */
    private function getValidCountries() {
        return [
            'US', 'CA', 'GB', 'AU', 'DE', 'FR', 'ES', 'IT', 'NL', 'BE',
            'AT', 'CH', 'IE', 'NZ', 'SG', 'HK', 'JP', 'KR', 'IN', 'CN',
            'BR', 'MX', 'AR', 'CL', 'CO', 'PE', 'VE', 'ZA', 'EG', 'NG',
            'KE', 'AE', 'SA', 'IL', 'TR', 'PL', 'CZ', 'HU', 'RO', 'RU',
            'UA', 'SE', 'NO', 'DK', 'FI', 'PT', 'GR', 'TH', 'VN', 'MY',
            'PH', 'ID', 'TW'
        ];
    }
    
    /**
     * Fix validation issues
     */
    public function autoFixIssues($issueType, $fixType = 'default') {
        switch ($issueType) {
            case 'missing_uuid':
                return $this->fixMissingUuids();
            case 'invalid_country':
                return $this->fixInvalidCountries();
            default:
                return ['fixed' => 0, 'message' => 'No auto-fix available'];
        }
    }
    
    /**
     * Fix missing UUIDs
     */
    private function fixMissingUuids() {
        $fixed = 0;
        
        $clients = Capsule::table('tblclients')
            ->whereNull('uuid')
            ->orWhere('uuid', '')
            ->get(['id']);
        
        foreach ($clients as $client) {
            Capsule::table('tblclients')
                ->where('id', $client->id)
                ->update(['uuid' => Capsule::raw('UUID()')]);
            $fixed++;
        }
        
        return ['fixed' => $fixed, 'message' => 'UUIDs generated for ' . $fixed . ' clients'];
    }
}
```

### Step 2: Execute Validation

```php
<?php
/**
 * Run data validation
 */
$validator = new DataValidator();

$results = $validator->runAllValidations();

echo "Data Validation Report\n";
echo "======================\n\n";

foreach ($results as $type => $result) {
    echo ucfirst($type) . ":\n";
    echo "  Checked: " . ($result['total_checked'] ?? 'N/A') . "\n";
    echo "  Issues: " . $result['issues_found'] . "\n";
    
    if (!empty($result['issues']) && count($result['issues']) > 0) {
        echo "  Details:\n";
        foreach (array_slice($result['issues'], 0, 5) as $issue) {
            echo "    - " . $issue['message'] . "\n";
        }
    }
    echo "\n";
}
```

## Best Practices
- Run validation regularly
- Fix critical issues first
- Backup before auto-fixes
- Review all issues manually
- Test fixes on staging
- Document validation results
- Set up automated checks
- Monitor data quality
- Train staff on data entry
- Implement validation rules

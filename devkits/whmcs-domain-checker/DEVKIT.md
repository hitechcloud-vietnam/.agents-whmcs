# WHMCS Domain Checker Module

```php
<?php
/**
 * WHMCS Domain Checker Module
 * 
 * Provides domain availability checking with pricing tiers,
 * bulk lookup, and transfer support.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access prohibited");
}

function domainchecker_MetaData()
{
    return array(
        'DisplayName' => 'Domain Checker',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    );
}

function domainchecker_ConfigArray()
{
    return array(
        'FriendlyName' => array(
            'Type' => 'System',
            'Value' => 'Domain Checker',
        ),
        'DefaultTlds' => array(
            'Type' => 'text',
            'Size' => '100',
            'Default' => '.com,.net,.org,.io,.co',
            'Description' => 'Default TLDs to check (comma-separated)',
        ),
        'EnableBulkCheck' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable bulk domain checking',
        ),
        'MaxBulkCheck' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '10',
            'Description' => 'Maximum domains per bulk check',
        ),
        'EnableTransferCheck' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Check transfer eligibility',
        ),
        'EnablePricing' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Show domain pricing',
        ),
        'CacheTTL' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '3600',
            'Description' => 'Cache TTL in seconds for availability',
        ),
    );
}

function domainchecker_activate()
{
    try {
        if (!function_exists('createTable')) {
            require_once dirname(__FILE__) . '/../../includes/modulefunctions.php';
        }
        
        // TLD pricing table
        $pricingTable = 'mod_domainchecker_pricing';
        $pricingSchema = "
            CREATE TABLE `{$pricingTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `tld` VARCHAR(50) NOT NULL,
                `registration_price` DECIMAL(10,2) NOT NULL,
                `renewal_price` DECIMAL(10,2) NOT NULL,
                `transfer_price` DECIMAL(10,2) DEFAULT NULL,
                `registration_period` INT DEFAULT 1,
                `currency` VARCHAR(10) DEFAULT 'USD',
                `is_active` TINYINT(1) DEFAULT 1,
                `category` VARCHAR(50) DEFAULT 'standard',
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_tld` (`tld`, `currency`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($pricingTable, $pricingSchema);
        
        // Domain cache table
        $cacheTable = 'mod_domainchecker_cache';
        $cacheSchema = "
            CREATE TABLE `{$cacheTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `domain` VARCHAR(255) NOT NULL,
                `tld` VARCHAR(50) NOT NULL,
                `is_available` TINYINT(1) DEFAULT NULL,
                `checked_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `expires_at` DATETIME NOT NULL,
                UNIQUE KEY `unique_domain` (`domain`),
                INDEX `idx_expires` (`expires_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($cacheTable, $cacheSchema);
        
        // Search logs table
        $logsTable = 'mod_domainchecker_logs';
        $logsSchema = "
            CREATE TABLE `{$logsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `search_query` VARCHAR(255) NOT NULL,
                `tlds_checked` INT DEFAULT 1,
                `results_found` INT DEFAULT 0,
                `domains_purchased` INT DEFAULT 0,
                `ip_address` VARCHAR(45) NULL,
                `user_id` INT NULL,
                `searched_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_searched_at` (`searched_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($logsTable, $logsSchema);
        
        // Insert default TLDs with pricing
        $defaultTlds = array(
            array('tld' => '.com', 'reg' => 9.99, 'ren' => 12.99, 'trans' => 9.99, 'cat' => 'popular'),
            array('tld' => '.net', 'reg' => 11.99, 'ren' => 14.99, 'trans' => 11.99, 'cat' => 'popular'),
            array('tld' => '.org', 'reg' => 10.99, 'ren' => 13.99, 'trans' => 10.99, 'cat' => 'popular'),
            array('tld' => '.io', 'reg' => 29.99, 'ren' => 34.99, 'trans' => 29.99, 'cat' => 'premium'),
            array('tld' => '.co', 'reg' => 19.99, 'ren' => 24.99, 'trans' => 19.99, 'cat' => 'premium'),
            array('tld' => '.app', 'reg' => 14.99, 'ren' => 17.99, 'trans' => null, 'cat' => 'new'),
            array('tld' => '.dev', 'reg' => 14.99, 'ren' => 17.99, 'trans' => null, 'cat' => 'new'),
            array('tld' => '.xyz', 'reg' => 2.99, 'ren' => 9.99, 'trans' => 2.99, 'cat' => 'budget'),
            array('tld' => '.online', 'reg' => 3.99, 'ren' => 12.99, 'trans' => 3.99, 'cat' => 'new'),
            array('tld' => '.site', 'reg' => 3.99, 'ren' => 12.99, 'trans' => 3.99, 'cat' => 'new'),
        );
        
        foreach ($defaultTlds as $tld) {
            Capsule::table('mod_domainchecker_pricing')->insert(array(
                'tld' => $tld['tld'],
                'registration_price' => $tld['reg'],
                'renewal_price' => $tld['ren'],
                'transfer_price' => $tld['trans'],
                'category' => $tld['cat'],
            ));
        }
        
        return array(
            'status' => 'success',
            'description' => 'Domain Checker module activated with default TLDs.',
        );
    } catch (\Exception $e) {
        return array(
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        );
    }
}

function domainchecker_deactivate()
{
    return array('status' => 'success', 'description' => 'Module deactivated.');
}

/**
 * Check domain availability
 */
function domainchecker_CheckDomain($domain, $tlds = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    // Parse domain
    $domain = strtolower(trim($domain));
    $domain = preg_replace('/^https?:\/\//', '', $domain);
    $domain = preg_replace('/^www\./', '', $domain);
    $domain = preg_replace('/\..*$/', '', $domain);
    
    if (empty($domain)) {
        return array('success' => false, 'error' => 'Invalid domain name');
    }
    
    // Default TLDs
    if (empty($tlds)) {
        $tlds = array('.com');
    }
    
    $results = array();
    
    foreach ($tlds as $tld) {
        $tld = '.' . ltrim($tld, '.');
        $fullDomain = $domain . $tld;
        
        // Check cache first
        $cached = domainchecker_GetCachedResult($fullDomain);
        if ($cached !== null) {
            $results[$fullDomain] = $cached;
            continue;
        }
        
        // Check availability (simulated - replace with real API)
        $isAvailable = domainchecker_QueryRegistry($fullDomain);
        
        // Cache result
        domainchecker_CacheResult($fullDomain, $isAvailable);
        
        // Get pricing
        $pricing = domainchecker_GetPricing($tld);
        
        $results[$fullDomain] = array(
            'domain' => $fullDomain,
            'tld' => $tld,
            'is_available' => $isAvailable,
            'pricing' => $pricing,
        );
    }
    
    // Log search
    domainchecker_LogSearch($domain, count($tlds), count(array_filter($results, function($r) {
        return $r['is_available'];
    })));
    
    return array('success' => true, 'results' => $results);
}

/**
 * Query domain registry (placeholder - implement real API)
 */
function domainchecker_QueryRegistry($domain)
{
    // This is a placeholder implementation
    // Replace with actual registry API call (e.g., Nominet, Verisign, etc.)
    
    // Simulate availability based on hash
    $hash = crc32($domain);
    return ($hash % 3) !== 0; // ~66% available
}

/**
 * Bulk domain check
 */
function domainchecker_BulkCheck($domains, $tlds = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    // Limit check
    $maxCheck = Capsule::table('mod_domainchecker_config')
        ->where('setting', 'MaxBulkCheck')
        ->first();
    
    $limit = $maxCheck ? (int)$maxCheck->value : 10;
    
    if (count($domains) > $limit) {
        return array(
            'success' => false,
            'error' => "Maximum {$limit} domains per bulk check",
        );
    }
    
    $allResults = array();
    
    foreach ($domains as $domain) {
        $result = domainchecker_CheckDomain($domain, $tlds);
        if ($result['success']) {
            $allResults = array_merge($allResults, $result['results']);
        }
    }
    
    return array('success' => true, 'results' => $allResults);
}

/**
 * Check transfer eligibility
 */
function domainchecker_CheckTransfer($domain)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $domain = strtolower(trim($domain));
    
    // Get TLD pricing
    $parts = explode('.', $domain, 2);
    $tld = '.' . ($parts[1] ?? 'com');
    
    $pricing = domainchecker_GetPricing($tld);
    
    // Check if transfer is supported
    if (empty($pricing['transfer_price'])) {
        return array(
            'success' => false,
            'error' => 'Transfer not supported for this TLD',
        );
    }
    
    // Check domain status (simulated)
    $canTransfer = domainchecker_CheckTransferLock($domain);
    $domainAge = domainchecker_GetDomainAge($domain);
    
    return array(
        'success' => true,
        'domain' => $domain,
        'can_transfer' => $canTransfer,
        'is_locked' => !$canTransfer,
        'domain_age_days' => $domainAge,
        'pricing' => $pricing,
        'requirements' => array(
            'unlock_domain' => !$canTransfer,
            'wait_period_passed' => $domainAge >= 60,
            'authorization_code_needed' => true,
        ),
    );
}

/**
 * Check if domain is locked
 */
function domainchecker_CheckTransferLock($domain)
{
    // Placeholder - implement actual WHOIS lookup
    $hash = crc32($domain);
    return ($hash % 5) !== 0; // 80% unlocked
}

/**
 * Get domain age in days
 */
function domainchecker_GetDomainAge($domain)
{
    // Placeholder - implement actual WHOIS lookup
    $hash = crc32($domain);
    return ($hash % 365) + 30; // Random age between 30-395 days
}

/**
 * Get cached result
 */
function domainchecker_GetCachedResult($domain)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $cached = Capsule::table('mod_domainchecker_cache')
        ->where('domain', $domain)
        ->where('expires_at', '>', date('Y-m-d H:i:s'))
        ->first();
    
    if ($cached) {
        return array(
            'is_available' => (bool) $cached->is_available,
            'cached' => true,
            'checked_at' => $cached->checked_at,
        );
    }
    
    return null;
}

/**
 * Cache result
 */
function domainchecker_CacheResult($domain, $isAvailable)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $ttl = 3600; // Default 1 hour
    
    Capsule::table('mod_domainchecker_cache')->updateOrInsert(
        array('domain' => $domain),
        array(
            'is_available' => $isAvailable ? 1 : 0,
            'checked_at' => date('Y-m-d H:i:s'),
            'expires_at' => date('Y-m-d H:i:s', time() + $ttl),
        )
    );
}

/**
 * Get TLD pricing
 */
function domainchecker_GetPricing($tld)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $pricing = Capsule::table('mod_domainchecker_pricing')
        ->where('tld', $tld)
        ->where('is_active', 1)
        ->first();
    
    if ($pricing) {
        return array(
            'registration' => (float) $pricing->registration_price,
            'renewal' => (float) $pricing->renewal_price,
            'transfer' => $pricing->transfer_price ? (float) $pricing->transfer_price : null,
            'period' => $pricing->registration_period,
            'category' => $pricing->category,
        );
    }
    
    return null;
}

/**
 * Set TLD pricing
 */
function domainchecker_SetPricing($tld, $pricing)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $data = array(
            'registration_price' => $pricing['registration'],
            'renewal_price' => $pricing['renewal'],
            'transfer_price' => $pricing['transfer'] ?? null,
            'registration_period' => $pricing['period'] ?? 1,
            'category' => $pricing['category'] ?? 'standard',
        );
        
        Capsule::table('mod_domainchecker_pricing')->updateOrInsert(
            array('tld' => $tld),
            $data
        );
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get all available TLDs with pricing
 */
function domainchecker_GetTlds($category = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_domainchecker_pricing')
        ->where('is_active', 1)
        ->orderBy('registration_price', 'asc');
    
    if ($category) {
        $query->where('category', $category);
    }
    
    $tlds = $query->get();
    
    $result = array();
    foreach ($tlds as $tld) {
        $result[$tld->tld] = array(
            'registration' => (float) $tld->registration_price,
            'renewal' => (float) $tld->renewal_price,
            'transfer' => $tld->transfer_price ? (float) $tld->transfer_price : null,
            'category' => $tld->category,
        );
    }
    
    return $result;
}

/**
 * Get TLD categories
 */
function domainchecker_GetCategories()
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $categories = Capsule::table('mod_domainchecker_pricing')
        ->select('category')
        ->distinct()
        ->get();
    
    return array_column($categories, 'category');
}

/**
 * Log search
 */
function domainchecker_LogSearch($query, $tldsChecked, $resultsFound)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_domainchecker_logs')->insert(array(
            'search_query' => $query,
            'tlds_checked' => $tldsChecked,
            'results_found' => $resultsFound,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
        ));
    } catch (\Exception $e) {
        // Silently fail logging
    }
}

/**
 * Get search statistics
 */
function domainchecker_GetStats($days = 30)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    
    $stats = Capsule::table('mod_domainchecker_logs')
        ->where('searched_at', '>=', $since)
        ->selectRaw("
            COUNT(*) as total_searches,
            SUM(tlds_checked) as total_tlds_checked,
            SUM(results_found) as total_available,
            COUNT(CASE WHEN domains_purchased > 0 THEN 1 END) as conversions
        ")
        ->first();
    
    $topTlds = Capsule::table('mod_domainchecker_logs')
        ->where('searched_at', '>=', $since)
        ->selectRaw("search_query, COUNT(*) as count")
        ->groupBy('search_query')
        ->orderBy('count', 'desc')
        ->limit(10)
        ->get();
    
    return array(
        'total_searches' => (int) $stats->total_searches,
        'total_tlds_checked' => (int) $stats->total_tlds_checked,
        'total_available' => (int) $stats->total_available,
        'conversions' => (int) $stats->conversions,
        'conversion_rate' => $stats->total_searches > 0 
            ? round(($stats->conversions / $stats->total_searches) * 100, 2) 
            : 0,
        'top_searches' => $topTlds,
    );
}

/**
 * Generate domain suggestions
 */
function domainchecker_GetSuggestions($domain)
{
    $domain = strtolower(trim($domain));
    $domain = preg_replace('/^https?:\/\//', '', $domain);
    $domain = preg_replace('/^www\./', '', $domain);
    $baseName = preg_replace('/\..*$/', '', $domain);
    $currentTld = preg_replace('/^' . preg_quote($baseName, '/') . '/', '', $domain);
    
    $suggestions = array();
    
    // Add prefixes
    $prefixes = array('get', 'try', 'go', 'my', 'the', 'buy', 'get');
    foreach ($prefixes as $prefix) {
        $suggestions[] = $prefix . $baseName;
    }
    
    // Add suffixes
    $suffixes = array('er', 'ly', 'ify', 'hub', 'zone', 'lab');
    foreach ($suffixes as $suffix) {
        $suggestions[] = $baseName . $suffix;
    }
    
    // Alternative TLDs
    $altTlds = array('.co', '.io', '.app', '.dev');
    foreach ($altTlds as $tld) {
        if ($currentTld !== $tld) {
            $suggestions[] = $baseName . $tld;
        }
    }
    
    // Check suggestions
    $results = array();
    foreach ($suggestions as $suggestion) {
        $results[$suggestion] = domainchecker_CheckDomain($suggestion, array());
    }
    
    return $results;
}

/**
 * Render domain checker HTML
 */
function domainchecker_RenderChecker($options = array())
{
    $tlds = domainchecker_GetTlds();
    $defaultTlds = isset($options['default_tlds']) ? $options['default_tlds'] : array('.com', '.net', '.org');
    
    $html = '<div class="domain-checker">';
    $html .= '<form class="domain-search-form" method="post">';
    $html .= '<div class="search-input-group">';
    $html .= '<input type="text" name="domain" class="domain-input" placeholder="Enter domain name">';
    $html .= '<button type="submit" class="btn-search">Check</button>';
    $html .= '</div>';
    
    // TLD checkboxes
    $html .= '<div class="tld-selector">';
    $html .= '<span class="tld-label">Extensions:</span>';
    foreach ($tlds as $tld => $pricing) {
        $checked = in_array($tld, $defaultTlds) ? ' checked' : '';
        $html .= '<label class="tld-option">';
        $html .= '<input type="checkbox" name="tlds[]" value="' . htmlspecialchars($tld) . '"' . $checked . '>';
        $html .= '<span class="tld-name">' . htmlspecialchars($tld) . '</span>';
        $html .= '<span class="tld-price">$' . number_format($pricing['registration'], 2) . '</span>';
        $html .= '</label>';
    }
    $html .= '</div>';
    
    $html .= '</form>';
    $html .= '<div class="search-results"></div>';
    $html .= '</div>';
    
    return $html;
}

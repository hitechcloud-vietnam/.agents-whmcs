# WHMCS Domain Checker Module

A comprehensive domain availability checker with pricing tiers, bulk lookup, transfer support, and suggestions.

## Features

- Domain availability checking
- Multiple TLD support with pricing
- Bulk domain checking
- Transfer eligibility checking
- Cached results
- Domain suggestions
- Search analytics
- Category-based TLD organization
- Custom pricing per TLD

## Installation

1. Copy the module to your WHMCS installation:
   ```
   /path/to/whmcs/modules/servers/domainchecker/
   ```

2. Activate through WHMCS Admin:
   - Go to **Setup > Addon Modules**
   - Find "Domain Checker"
   - Click **Activate**
   - Configure settings

3. Configure TLDs and pricing through the admin interface.

## Usage

### Checking Domain Availability

```php
// Check single domain with specific TLDs
$result = domainchecker_CheckDomain('example', array('.com', '.net', '.org'));

if ($result['success']) {
    foreach ($result['results'] as $domain => $data) {
        echo $domain . ': ' . ($data['is_available'] ? 'Available' : 'Taken');
        if ($data['is_available'] && $data['pricing']) {
            echo ' - $' . $data['pricing']['registration'] . '/year';
        }
    }
}
```

### Bulk Checking

```php
// Check multiple domains
$domains = array('domain1', 'domain2', 'domain3');
$tlds = array('.com', '.net');

$result = domainchecker_BulkCheck($domains, $tlds);

if ($result['success']) {
    foreach ($result['results'] as $domain => $data) {
        // Process each result
    }
}
```

### Transfer Checking

```php
// Check if domain can be transferred
$result = domainchecker_CheckTransfer('example.com');

if ($result['success']) {
    echo 'Domain: ' . $result['domain'];
    echo 'Can Transfer: ' . ($result['can_transfer'] ? 'Yes' : 'No');
    echo 'Is Locked: ' . ($result['is_locked'] ? 'Yes' : 'No');
    echo 'Requirements:';
    print_r($result['requirements']);
}
```

### Pricing Management

```php
// Get pricing for a TLD
$pricing = domainchecker_GetPricing('.com');
// Returns: registration, renewal, transfer, period, category

// Set pricing for a TLD
domainchecker_SetPricing('.app', array(
    'registration' => 14.99,
    'renewal' => 17.99,
    'transfer' => 14.99,
    'category' => 'new',
));

// Get all TLDs
$tlds = domainchecker_GetTlds();
// Returns array of all active TLDs with pricing

// Get TLDs by category
$tlds = domainchecker_GetTlds('premium');

// Get TLD categories
$categories = domainchecker_GetCategories();
```

### Suggestions

```php
// Get domain suggestions
$suggestions = domainchecker_GetSuggestions('mysite');

// Check which suggestions are available
foreach ($suggestions as $domain => $result) {
    if ($result['success'] && $result['results'][$domain]['is_available']) {
        echo $domain . ' is available!';
    }
}
```

### Statistics

```php
// Get search statistics
$stats = domainchecker_GetStats(30);
// Returns: total_searches, total_tlds_checked, total_available, conversions, conversion_rate, top_searches
```

### Rendering

```php
// Render domain checker form
$html = domainchecker_RenderChecker(array(
    'default_tlds' => array('.com', '.net', '.org'),
));

echo $html;
```

## Default TLDs

The module comes with default pricing for popular TLDs:

| TLD | Registration | Renewal | Category |
|-----|-------------|---------|----------|
| .com | $9.99 | $12.99 | popular |
| .net | $11.99 | $14.99 | popular |
| .org | $10.99 | $13.99 | popular |
| .io | $29.99 | $34.99 | premium |
| .co | $19.99 | $24.99 | premium |
| .app | $14.99 | $17.99 | new |
| .dev | $14.99 | $17.99 | new |
| .xyz | $2.99 | $9.99 | budget |
| .online | $3.99 | $12.99 | new |
| .site | $3.99 | $12.99 | new |

## Database Tables

- `mod_domainchecker_pricing` - TLD pricing
- `mod_domainchecker_cache` - Availability cache
- `mod_domainchecker_logs` - Search logs

## API Functions

| Function | Description |
|----------|-------------|
| `domainchecker_CheckDomain()` | Check domain availability |
| `domainchecker_BulkCheck()` | Bulk domain checking |
| `domainchecker_CheckTransfer()` | Check transfer eligibility |
| `domainchecker_GetPricing()` | Get TLD pricing |
| `domainchecker_SetPricing()` | Set TLD pricing |
| `domainchecker_GetTlds()` | Get all TLDs |
| `domainchecker_GetCategories()` | Get TLD categories |
| `domainchecker_GetSuggestions()` | Get domain suggestions |
| `domainchecker_GetStats()` | Get search statistics |
| `domainchecker_RenderChecker()` | Render checker form |

## Registry Integration

To integrate with a real domain registry, replace the `domainchecker_QueryRegistry()` function with actual API calls to your registrar or WHOIS server.

## Version History

- **1.0.0** - Initial release
  - Domain availability checking
  - TLD pricing management
  - Bulk checking
  - Transfer eligibility
  - Domain suggestions
  - Search analytics
  - Result caching

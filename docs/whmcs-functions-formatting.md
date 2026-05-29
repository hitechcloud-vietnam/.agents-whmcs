# WHMCS Formatting Functions

Complete reference for data formatting functions in WHMCS.

## Overview

WHMCS provides formatting utilities for displaying currencies, dates, numbers, and other data in user-friendly formats.

## Currency Formatting

### formatCurrency()

Formats a monetary amount.

```php
/**
 * Format currency amount
 * 
 * @param float $amount Amount to format
 * @param int $currencyId Currency ID
 * @param string $symbol Position ('before' or 'after')
 * @return string Formatted amount
 */
function formatCurrency(float $amount, int $currencyId = 1, string $symbol = 'before'): string
{
    $currency = Capsule::table('tblcurrencies')
        ->where('id', $currencyId)
        ->first();
    
    if (!$currency) {
        $currency = (object) [
            'code' => 'USD',
            'prefix' => '$',
            'suffix' => '',
            'format' => 1
        ];
    }
    
    $amount = number_format($amount, $currency->format ?? 2, '.', ',');
    
    if ($symbol === 'after') {
        return $amount . ' ' . $currency->suffix;
    }
    
    return $currency->prefix . $amount;
}
```

**Example:**
```php
echo formatCurrency(1234.56, 1);
// Output: $1,234.56

echo formatCurrency(1234.56, 2, 'after');
// Output: 1.234,56 EUR
```

### formatMoney()

Formats money with specific symbol.

```php
/**
 * Format money with symbol
 * 
 * @param float $amount Amount
 * @param string $symbol Currency symbol
 * @param int $decimals Decimal places
 * @return string Formatted amount
 */
function formatMoney(float $amount, string $symbol = '$', int $decimals = 2): string
{
    return $symbol . number_format($amount, $decimals, '.', ',');
}
```

### parseCurrency()

Parses currency string to float.

```php
/**
 * Parse currency string to amount
 * 
 * @param string $currencyString Currency string
 * @return float Amount
 */
function parseCurrency(string $currencyString): float
{
    // Remove currency symbols and formatting
    $amount = preg_replace('/[^0-9.-]/', '', $currencyString);
    
    return (float) $amount;
}
```

## Number Formatting

### formatNumber()

Formats a number.

```php
/**
 * Format number
 * 
 * @param float|int $number Number to format
 * @param int $decimals Decimal places
 * @param string $decPoint Decimal separator
 * @param string $thousandsSep Thousands separator
 * @return string Formatted number
 */
function formatNumber(
    $number,
    int $decimals = 0,
    string $decPoint = '.',
    string $thousandsSep = ','
): string {
    return number_format($number, $decimals, $decPoint, $thousandsSep);
}
```

**Example:**
```php
echo formatNumber(1234567, 2);
// Output: 1,234,567.00

echo formatNumber(1234.5, 2, ',', ' ');
// Output: 1 234,50
```

### formatBytes()

Formats bytes to human-readable.

```php
/**
 * Format bytes to human readable
 * 
 * @param int $bytes Bytes
 * @param int $precision Precision
 * @return string Formatted size
 */
function formatBytes(int $bytes, int $precision = 2): string
{
    $units = ['B', 'KB', 'MB', 'GB', 'TB', 'PB'];
    
    $bytes = max($bytes, 0);
    $pow = floor(($bytes ? log($bytes) : 0) / log(1024));
    $pow = min($pow, count($units) - 1);
    
    $bytes /= (1 << (10 * $pow));
    
    return round($bytes, $precision) . ' ' . $units[$pow];
}
```

**Example:**
```php
echo formatBytes(1024);
// Output: 1.00 KB

echo formatBytes(1048576);
// Output: 1.00 MB

echo formatBytes(1234567890);
// Output: 1.15 GB
```

### formatPercentage()

Formats percentage.

```php
/**
 * Format percentage
 * 
 * @param float $value Value (0-100 or 0-1)
 * @param int $decimals Decimal places
 * @param bool $isDecimal True if value is 0-1
 * @return string Formatted percentage
 */
function formatPercentage(float $value, int $decimals = 1, bool $isDecimal = false): string
{
    if ($isDecimal) {
        $value *= 100;
    }
    
    return number_format($value, $decimals) . '%';
}
```

## Date Formatting

### formatDate()

Formats date for display.

```php
/**
 * Format date
 * 
 * @param string $date Date string
 * @param string $format Output format
 * @return string Formatted date
 */
function formatDate(string $date, string $format = 'Y-m-d'): string
{
    if (empty($date)) {
        return '';
    }
    
    return date($format, strtotime($date));
}
```

**Example:**
```php
echo formatDate('2024-01-15', 'M j, Y');
// Output: Jan 15, 2024

echo formatDate('2024-01-15 14:30:00', 'F j, Y g:i A');
// Output: January 15, 2024 2:30 PM
```

### formatDateTime()

Formats date and time.

```php
/**
 * Format date and time
 * 
 * @param string $datetime DateTime string
 * @param string $format Output format
 * @return string Formatted datetime
 */
function formatDateTime(string $datetime, string $format = 'Y-m-d H:i:s'): string
{
    if (empty($datetime)) {
        return '';
    }
    
    return date($format, strtotime($datetime));
}
```

### fromMySQLDate()

Converts MySQL date to display format.

```php
/**
 * Convert MySQL date to display format
 * 
 * @param string $date MySQL date
 * @param bool $includeTime Include time
 * @return string Formatted date
 */
function fromMySQLDate(string $date, bool $includeTime = false): string
{
    if (empty($date)) {
        return '';
    }
    
    $format = Config\Setting::getValue('DateFormat') ?: 'Y-m-d';
    
    if ($includeTime) {
        $format .= ' H:i';
    }
    
    return date($format, strtotime($date));
}
```

### toMySQLDate()

Converts display date to MySQL format.

```php
/**
 * Convert display date to MySQL format
 * 
 * @param string $date Display date
 * @return string MySQL date
 */
function toMySQLDate(string $date): string
{
    $timestamp = strtotime($date);
    
    return $timestamp ? date('Y-m-d', $timestamp) : '';
}
```

## Time Formatting

### formatTime()

Formats time.

```php
/**
 * Format time
 * 
 * @param string $time Time string
 * @param string $format Output format
 * @return string Formatted time
 */
function formatTime(string $time, string $format = 'H:i'): string
{
    return date($format, strtotime($time));
}
```

### formatDuration()

Formats duration in seconds to readable.

```php
/**
 * Format duration
 * 
 * @param int $seconds Duration in seconds
 * @param string $format Format string
 * @return string Formatted duration
 */
function formatDuration(int $seconds, string $format = 'auto'): string
{
    if ($format === 'auto') {
        if ($seconds < 60) {
            return $seconds . ' sec';
        }
        
        if ($seconds < 3600) {
            $mins = floor($seconds / 60);
            return $mins . ' min';
        }
        
        if ($seconds < 86400) {
            $hours = floor($seconds / 3600);
            $mins = floor(($seconds % 3600) / 60);
            return $hours . 'h ' . $mins . 'm';
        }
        
        $days = floor($seconds / 86400);
        $hours = floor(($seconds % 86400) / 3600);
        return $days . 'd ' . $hours . 'h';
    }
    
    $hours = floor($seconds / 3600);
    $mins = floor(($seconds % 3600) / 60);
    $secs = $seconds % 60;
    
    $format = str_replace(['H', 'm', 's'], [$hours, $mins, $secs], $format);
    
    return $format;
}
```

**Example:**
```php
echo formatDuration(3665);
// Output: 1h 1m

echo formatDuration(3665, 'H:m:s');
// Output: 1:01:05
```

## Text Formatting

### truncate()

Truncates text with ellipsis.

```php
/**
 * Truncate text
 * 
 * @param string $text Text to truncate
 * @param int $length Maximum length
 * @param string $ellipsis Ellipsis string
 * @param bool $break Break words
 * @return string Truncated text
 */
function truncate(string $text, int $length = 100, string $ellipsis = '...', bool $break = false): string
{
    if (mb_strlen($text) <= $length) {
        return $text;
    }
    
    if (!$break) {
        $text = preg_replace('/\s+?(\S+)?$/', '', mb_substr($text, 0, $length + 1));
    }
    
    return mb_substr($text, 0, $length) . $ellipsis;
}
```

### capitalize()

Capitalizes text.

```php
/**
 * Capitalize text
 * 
 * @param string $text Text to capitalize
 * @param bool $eachWord Capitalize each word
 * @return string Capitalized text
 */
function capitalize(string $text, bool $eachWord = false): string
{
    if ($eachWord) {
        return ucwords(strtolower($text));
    }
    
    return ucfirst(strtolower($text));
}
```

### titleCase()

Converts text to title case.

```php
/**
 * Convert to title case
 * 
 * @param string $text Text to convert
 * @return string Title case text
 */
function titleCase(string $text): string
{
    $small = ['a', 'an', 'the', 'and', 'but', 'or', 'for', 'nor', 'on', 'at', 'to', 'by', 'of'];
    $words = explode(' ', strtolower($text));
    
    foreach ($words as $key => $word) {
        if ($key === 0 || !in_array($word, $small)) {
            $words[$key] = ucfirst($word);
        }
    }
    
    return implode(' ', $words);
}
```

## Phone Formatting

### formatPhone()

Formats phone number.

```php
/**
 * Format phone number
 * 
 * @param string $phone Phone number
 * @param string $country Country code
 * @return string Formatted phone
 */
function formatPhone(string $phone, string $country = 'US'): string
{
    // Remove non-digits
    $phone = preg_replace('/[^0-9]/', '', $phone);
    
    switch ($country) {
        case 'US':
        case 'CA':
            if (strlen($phone) === 10) {
                return '(' . substr($phone, 0, 3) . ') ' . 
                       substr($phone, 3, 3) . '-' . 
                       substr($phone, 6);
            }
            if (strlen($phone) === 11 && $phone[0] === '1') {
                return '+1 (' . substr($phone, 1, 3) . ') ' . 
                       substr($phone, 4, 3) . '-' . 
                       substr($phone, 7);
            }
            break;
            
        case 'UK':
            if (strlen($phone) === 10) {
                return substr($phone, 0, 4) . ' ' . substr($phone, 4, 3) . ' ' . substr($phone, 7);
            }
            break;
    }
    
    return $phone;
}
```

## Address Formatting

### formatAddress()

Formats address.

```php
/**
 * Format address
 * 
 * @param array $address Address data
 * @param string $format Output format
 * @param string $newline Newline character
 * @return string Formatted address
 */
function formatAddress(array $address, string $format = 'multi', string $newline = "\n"): string
{
    $lines = [];
    
    // Name/Company
    if (!empty($address['companyname'])) {
        $lines[] = $address['companyname'];
    }
    
    // Address lines
    if (!empty($address['address1'])) {
        $lines[] = $address['address1'];
    }
    if (!empty($address['address2'])) {
        $lines[] = $address['address2'];
    }
    
    // City, State, Postal
    $cityLine = [];
    if (!empty($address['city'])) {
        $cityLine[] = $address['city'];
    }
    if (!empty($address['state'])) {
        $cityLine[] = $address['state'];
    }
    if (!empty($address['postcode'])) {
        $cityLine[] = $address['postcode'];
    }
    
    if (!empty($cityLine)) {
        $lines[] = implode(', ', $cityLine);
    }
    
    // Country
    if (!empty($address['country'])) {
        $countryName = getCountryName($address['country']);
        $lines[] = $countryName;
    }
    
    return implode($newline, array_filter($lines));
}
```

## Credit Card Formatting

### formatCreditCard()

Formats credit card number.

```php
/**
 * Format credit card number
 * 
 * @param string $cardNumber Card number
 * @param string $mask Mask character
 * @return string Formatted card
 */
function formatCreditCard(string $cardNumber, string $mask = 'X'): string
{
    // Remove spaces
    $cardNumber = preg_replace('/\s+/', '', $cardNumber);
    
    // Mask all but last 4
    $length = strlen($cardNumber);
    
    if ($length > 4) {
        $masked = str_repeat($mask, $length - 4) . substr($cardNumber, -4);
        
        // Add spaces every 4 digits
        return implode(' ', str_split($masked, 4));
    }
    
    return str_repeat($mask, $length);
}
```

**Example:**
```php
echo formatCreditCard('4111111111111111');
// Output: XXXX XXXX XXXX 1111

echo formatCreditCard('4111111111111111', '*');
// Output: **** **** **** 1111
```

## JSON Formatting

### formatJson()

Formats JSON output.

```php
/**
 * Format JSON
 * 
 * @param mixed $data Data to format
 * @param bool $pretty Pretty print
 * @return string Formatted JSON
 */
function formatJson($data, bool $pretty = true): string
{
    if ($pretty) {
        return json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES);
    }
    
    return json_encode($data);
}
```

## List Formatting

### formatList()

Formats array as list.

```php
/**
 * Format list
 * 
 * @param array $items List items
 * @param string $separator Separator
 * @param string $lastSeparator Last separator
 * @return string Formatted list
 */
function formatList(array $items, string $separator = ', ', string $lastSeparator = ' and '): string
{
    if (count($items) === 0) {
        return '';
    }
    
    if (count($items) === 1) {
        return $items[0];
    }
    
    $last = array_pop($items);
    
    return implode($separator, $items) . $lastSeparator . $last;
}
```

**Example:**
```php
echo formatList(['apple', 'banana', 'orange']);
// Output: apple, banana and orange

echo formatList(['one', 'two'], ', ', ' or ');
// Output: one or two
```

## Best Practices

1. **Use locale-aware formatting** - Consider user's location
2. **Consistent date formats** - Use standard formats across the system
3. **Handle null values** - Gracefully handle empty dates/numbers
4. **Preserve precision** - Don't truncate in calculations
5. **Use proper ellipsis** - Use appropriate ellipsis characters

## Related Functions

- [whmcs-functions-date-time.md](whmcs-functions-date-time.md) - Date/time functions
- [whmcs-functions-utility.md](whmcs-functions-utility.md) - Utility functions
# WHMCS Date/Time Functions

Complete reference for date and time manipulation functions in WHMCS.

## Overview

WHMCS provides comprehensive date/time utilities for handling dates, times, timezones, and date calculations.

## Core Date Functions

### now()

Returns current datetime.

```php
/**
 * Get current datetime
 * 
 * @param bool $asObject Return as DateTime object
 * @param string $timezone Timezone
 * @return string|DateTime
 */
function now(bool $asObject = false, string $timezone = ''): mixed
{
    $date = new DateTime();
    
    if ($timezone) {
        $date->setTimezone(new DateTimeZone($timezone));
    }
    
    return $asObject ? $date : $date->format('Y-m-d H:i:s');
}
```

**Example:**
```php
echo now(); // Output: 2024-01-15 14:30:45

$dateObj = now(true); // Returns DateTime object
```

### today()

Returns today's date.

```php
/**
 * Get today's date
 * 
 * @param string $format Output format
 * @return string Date
 */
function today(string $format = 'Y-m-d'): string
{
    return date($format);
}
```

### currentTime()

Returns current time.

```php
/**
 * Get current time
 * 
 * @param string $format Output format
 * @return string Time
 */
function currentTime(string $format = 'H:i:s'): string
{
    return date($format);
}
```

## Date Parsing

### parseDate()

Parses date string.

```php
/**
 * Parse date string
 * 
 * @param string $date Date string
 * @param string|null $timezone Timezone
 * @return DateTime|null
 */
function parseDate(string $date, ?string $timezone = null): ?DateTime
{
    if (empty($date)) {
        return null;
    }
    
    try {
        $dt = new DateTime($date);
        
        if ($timezone) {
            $dt->setTimezone(new DateTimeZone($timezone));
        }
        
        return $dt;
    } catch (Exception $e) {
        return null;
    }
}
```

**Example:**
```php
$dt = parseDate('2024-01-15 14:30:00', 'America/New_York');
```

### parseMySQLDate()

Parses MySQL date format.

```php
/**
 * Parse MySQL date
 * 
 * @param string $date MySQL date
 * @return DateTime|null
 */
function parseMySQLDate(string $date): ?DateTime
{
    return parseDate($date);
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
 * @param string $timezone Timezone
 * @return string Formatted date
 */
function formatDate(string $date, string $format = 'Y-m-d', string $timezone = ''): string
{
    if (empty($date)) {
        return '';
    }
    
    $dt = new DateTime($date);
    
    if ($timezone) {
        $dt->setTimezone(new DateTimeZone($timezone));
    }
    
    return $dt->format($format);
}
```

**Example:**
```php
echo formatDate('2024-01-15', 'F j, Y');
// Output: January 15, 2024

echo formatDate('2024-01-15', 'M j, Y');
// Output: Jan 15, 2024

echo formatDate('2024-01-15 14:30:00', 'g:i A');
// Output: 2:30 PM
```

### formatTimeAgo()

Formats date as time ago.

```php
/**
 * Format as time ago
 * 
 * @param string $date Date string
 * @param string|null $referenceDate Reference date (null = now)
 * @param string $suffix Suffix ('ago' or 'from now')
 * @return string Time ago string
 */
function formatTimeAgo(string $date, ?string $referenceDate = null, string $suffix = 'ago'): string
{
    $reference = $referenceDate ? new DateTime($referenceDate) : new DateTime();
    $target = new DateTime($date);
    
    $diff = $reference->diff($target);
    
    $intervals = [
        'y' => 'year',
        'm' => 'month',
        'd' => 'day',
        'h' => 'hour',
        'i' => 'minute',
        's' => 'second'
    ];
    
    foreach ($intervals as $unit => $name) {
        if ($diff->$unit > 0) {
            $value = $diff->$unit;
            $plural = $value > 1 ? 's' : '';
            
            if ($suffix === 'from now') {
                return 'in ' . $value . ' ' . $name . $plural;
            }
            
            return $value . ' ' . $name . $plural . ' ' . $suffix;
        }
    }
    
    return 'just now';
}
```

**Example:**
```php
echo formatTimeAgo('2024-01-10'); // Output: 5 days ago
echo formatTimeAgo('2024-01-20', null, 'from now'); // Output: in 5 days
```

## Date Calculations

### addDays()

Adds days to a date.

```php
/**
 * Add days to date
 * 
 * @param string $date Date string
 * @param int $days Number of days
 * @param string $format Output format
 * @return string New date
 */
function addDays(string $date, int $days, string $format = 'Y-m-d'): string
{
    $dt = new DateTime($date);
    $dt->add(new DateInterval('P' . abs($days) . 'D'));
    
    return $dt->format($format);
}
```

### subtractDays()

Subtracts days from date.

```php
/**
 * Subtract days from date
 * 
 * @param string $date Date string
 * @param int $days Number of days
 * @param string $format Output format
 * @return string New date
 */
function subtractDays(string $date, int $days, string $format = 'Y-m-d'): string
{
    $dt = new DateTime($date);
    $dt->sub(new DateInterval('P' . abs($days) . 'D'));
    
    return $dt->format($format);
}
```

**Example:**
```php
echo addDays('2024-01-15', 30); // Output: 2024-02-14
echo subtractDays('2024-01-15', 7); // Output: 2024-01-08
```

### addMonths()

Adds months to date.

```php
/**
 * Add months to date
 * 
 * @param string $date Date string
 * @param int $months Number of months
 * @param string $format Output format
 * @return string New date
 */
function addMonths(string $date, int $months, string $format = 'Y-m-d'): string
{
    $dt = new DateTime($date);
    $dt->add(new DateInterval('P' . abs($months) . 'M'));
    
    return $dt->format($format);
}
```

### addYears()

Adds years to date.

```php
/**
 * Add years to date
 * 
 * @param string $date Date string
 * @param int $years Number of years
 * @param string $format Output format
 * @return string New date
 */
function addYears(string $date, int $years, string $format = 'Y-m-d'): string
{
    $dt = new DateTime($date);
    $dt->add(new DateInterval('P' . abs($years) . 'Y'));
    
    return $dt->format($format);
}
```

## Date Differences

### dateDiff()

Calculates difference between dates.

```php
/**
 * Calculate date difference
 * 
 * @param string $date1 First date
 * @param string $date2 Second date
 * @param string $unit Unit (days, hours, minutes, etc.)
 * @return int Difference
 */
function dateDiff(string $date1, string $date2, string $unit = 'days'): int
{
    $dt1 = new DateTime($date1);
    $dt2 = new DateTime($date2);
    
    $diff = $dt1->diff($dt2);
    
    switch ($unit) {
        case 'years':
            return $diff->y;
        case 'months':
            return ($diff->y * 12) + $diff->m;
        case 'days':
            return $diff->days;
        case 'hours':
            return $diff->days * 24 + $diff->h;
        case 'minutes':
            return ($diff->days * 24 + $diff->h) * 60 + $diff->i;
        case 'seconds':
            return (($diff->days * 24 + $diff->h) * 60 + $diff->i) * 60 + $diff->s;
        default:
            return $diff->days;
    }
}
```

**Example:**
```php
echo dateDiff('2024-01-15', '2024-02-15'); // Output: 31
echo dateDiff('2024-01-15 10:00', '2024-01-15 12:30', 'hours'); // Output: 2
```

### daysUntil()

Days until a date.

```php
/**
 * Get days until date
 * 
 * @param string $date Target date
 * @return int Days (negative if past)
 */
function daysUntil(string $date): int
{
    return dateDiff(date('Y-m-d'), $date);
}
```

### daysSince()

Days since a date.

```php
/**
 * Get days since date
 * 
 * @param string $date Source date
 * @return int Days (negative if future)
 */
function daysSince(string $date): int
{
    return dateDiff($date, date('Y-m-d'));
}
```

## Date Range Functions

### getDateRange()

Gets array of dates in range.

```php
/**
 * Get date range
 * 
 * @param string $startDate Start date
 * @param string $endDate End date
 * @param string $interval Interval (day, week, month)
 * @return array Dates
 */
function getDateRange(string $startDate, string $endDate, string $interval = 'day'): array
{
    $period = new DatePeriod(
        new DateTime($startDate),
        new DateInterval('P1' . strtoupper($interval[0])),
        new DateTime($endDate)
    );
    
    $dates = [];
    foreach ($period as $date) {
        $dates[] = $date->format('Y-m-d');
    }
    
    // Add end date if not included
    $last = end($dates);
    if ($last !== $endDate) {
        $dates[] = $endDate;
    }
    
    return $dates;
}
```

**Example:**
```php
$weekDates = getDateRange('2024-01-01', '2024-01-07');
// Output: ['2024-01-01', '2024-01-02', ..., '2024-01-07']

$monthDates = getDateRange('2024-01-01', '2024-01-31', 'day');
```

### getMonthDates()

Gets all dates in a month.

```php
/**
 * Get all dates in month
 * 
 * @param int $year Year
 * @param int $month Month
 * @return array Dates
 */
function getMonthDates(int $year, int $month): array
{
    $start = sprintf('%d-%02d-01', $year, $month);
    $end = date('Y-m-t', strtotime($start));
    
    return getDateRange($start, $end);
}
```

## Timezone Functions

### getTimezones()

Gets available timezones.

```php
/**
 * Get available timezones
 * 
 * @param string|null $region Filter by region
 * @return array Timezones
 */
function getTimezones(?string $region = null): array
{
    $timezones = DateTimeZone::listIdentifiers();
    
    if ($region) {
        $timezones = array_filter($timezones, function($tz) use ($region) {
            return strpos($tz, $region) === 0;
        });
    }
    
    return array_values($timezones);
}
```

**Example:**
```php
$timezones = getTimezones('America');
// Output: ['America/Adak', 'America/Anchorage', ...]

$allTimezones = getTimezones();
```

### convertTimezone()

Converts datetime between timezones.

```php
/**
 * Convert timezone
 * 
 * @param string $datetime DateTime string
 * @param string $fromTimezone Source timezone
 * @param string $toTimezone Target timezone
 * @param string $format Output format
 * @return string Converted datetime
 */
function convertTimezone(
    string $datetime,
    string $fromTimezone,
    string $toTimezone,
    string $format = 'Y-m-d H:i:s'
): string {
    $dt = new DateTime($datetime, new DateTimeZone($fromTimezone));
    $dt->setTimezone(new DateTimeZone($toTimezone));
    
    return $dt->format($format);
}
```

**Example:**
```php
echo convertTimezone('2024-01-15 10:00:00', 'America/New_York', 'Europe/London');
// Output: 2024-01-15 15:00:00
```

### getUserTimezone()

Gets user's timezone.

```php
/**
 * Get user timezone
 * 
 * @param int $userId User ID
 * @return string Timezone
 */
function getUserTimezone(int $userId): string
{
    $client = Capsule::table('tblclients')
        ->where('id', $userId)
        ->first();
    
    return $client->timezone ?? 'America/New_York';
}
```

### formatInTimezone()

Formats datetime in specific timezone.

```php
/**
 * Format datetime in timezone
 * 
 * @param string $datetime DateTime string
 * @param string $timezone Target timezone
 * @param string $format Output format
 * @return string Formatted datetime
 */
function formatInTimezone(
    string $datetime,
    string $timezone,
    string $format = 'Y-m-d H:i:s'
): string {
    return convertTimezone($datetime, 'UTC', $timezone, $format);
}
```

## Billing Date Functions

### calculateNextBillingDate()

Calculates next billing date.

```php
/**
 * Calculate next billing date
 * 
 * @param string $currentDate Current date
 * @param string $billingCycle Billing cycle
 * @return string Next billing date
 */
function calculateNextBillingDate(string $currentDate, string $billingCycle): string
{
    $intervals = [
        'One Time' => 'P0D',
        'Monthly' => 'P1M',
        'Quarterly' => 'P3M',
        'Semi-Annually' => 'P6M',
        'Annually' => 'P1Y',
        'Biennially' => 'P2Y',
        'Triennially' => 'P3Y'
    ];
    
    $interval = $intervals[$billingCycle] ?? 'P1M';
    
    $dt = new DateTime($currentDate);
    $dt->add(new DateInterval($interval));
    
    return $dt->format('Y-m-d');
}
```

**Example:**
```php
echo calculateNextBillingDate('2024-01-15', 'Monthly'); // Output: 2024-02-15
echo calculateNextBillingDate('2024-01-15', 'Annually'); // Output: 2025-01-15
```

### getBillingAnniversary()

Gets billing anniversary date.

```php
/**
 * Get billing anniversary
 * 
 * @param string $startDate Start date
 * @param string $billingCycle Billing cycle
 * @param int $count Anniversary number
 * @return string Anniversary date
 */
function getBillingAnniversary(string $startDate, string $billingCycle, int $count = 1): string
{
    $currentDate = $startDate;
    
    for ($i = 0; $i < $count; $i++) {
        $currentDate = calculateNextBillingDate($currentDate, $billingCycle);
    }
    
    return $currentDate;
}
```

## Date Validation

### isValidDate()

Validates date format.

```php
/**
 * Validate date
 * 
 * @param string $date Date string
 * @param string $format Expected format
 * @return bool Valid
 */
function isValidDate(string $date, string $format = 'Y-m-d'): bool
{
    $dt = DateTime::createFromFormat($format, $date);
    
    return $dt && $dt->format($format) === $date;
}
```

**Example:**
```php
isValidDate('2024-01-15'); // true
isValidDate('01/15/2024', 'm/d/Y'); // true
isValidDate('not-a-date'); // false
```

### isFutureDate()

Checks if date is in future.

```php
/**
 * Check if future date
 * 
 * @param string $date Date string
 * @return bool Is future
 */
function isFutureDate(string $date): bool
{
    return strtotime($date) > time();
}
```

### isPastDate()

Checks if date is in past.

```php
/**
 * Check if past date
 * 
 * @param string $date Date string
 * @return bool Is past
 */
function isPastDate(string $date): bool
{
    return strtotime($date) < time();
}
```

## Unix Timestamp Functions

### toTimestamp()

Converts to Unix timestamp.

```php
/**
 * Convert to timestamp
 * 
 * @param string $date Date string
 * @return int Timestamp
 */
function toTimestamp(string $date): int
{
    return strtotime($date);
}
```

### fromTimestamp()

Creates date from timestamp.

```php
/**
 * Create date from timestamp
 * 
 * @param int $timestamp Unix timestamp
 * @param string $format Output format
 * @return string Date
 */
function fromTimestamp(int $timestamp, string $format = 'Y-m-d H:i:s'): string
{
    return date($format, $timestamp);
}
```

## Best Practices

1. **Use UTC internally** - Store all dates in UTC
2. **Convert for display** - Convert to user's timezone for display
3. **Use DateTime objects** - For complex date operations
4. **Handle timezones properly** - Always consider timezone differences
5. **Be consistent with formats** - Use standard formats across the system
6. **Handle edge cases** - Consider month boundaries, leap years, etc.

## Related Functions

- [whmcs-functions-formatting.md](whmcs-functions-formatting.md) - Date formatting
- [whmcs-functions-utility.md](whmcs-functions-utility.md) - General utilities
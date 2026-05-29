# WHMCS Prorate Calculator

## Overview

The prorate calculator in WHMCS determines the exact amount to charge or credit when making mid-cycle changes to services. This tool ensures accurate billing adjustments for upgrades, downgrades, and other service modifications.

## Accessing Prorate Calculator

### Admin Area
**Configuration > Billing > Prorate Calculator**

### URL
```
/admin/configbillingprorate.php
```

## Calculator Parameters

### Required Inputs

| Field | Description | Example |
|-------|-------------|---------|
| Current Price | Existing service price | $20.00 |
| New Price | Target service price | $30.00 |
| Billing Cycle | Current billing period | Monthly |
| Start Date | Current cycle start | 2024-05-01 |
| End Date | Current cycle end | 2024-05-31 |
| Change Date | Date of modification | 2024-05-15 |

### Optional Inputs

| Field | Description |
|-------|-------------|
| Include Add-ons | Factor add-on pricing |
| Tax Rate | Apply tax to calculation |
| Round Decimals | Decimal place rounding |

## Calculation Methods

### Daily Rate Method

```php
// Most common method
$daysInCycle = date('t', strtotime($startDate));
$dailyRate = $monthlyPrice / $daysInCycle;
$prorateAmount = $dailyRate * $remainingDays;
```

### Hourly Rate Method

```php
// More precise calculation
$hoursInCycle = 30 * 24;  // 720 hours
$hourlyRate = $monthlyPrice / $hoursInCycle;
$remainingHours = $daysRemaining * 24;
$prorateAmount = $hourlyRate * $remainingHours;
```

### Actual Days Method

```php
// Calendar-accurate
$startTime = strtotime($startDate);
$endTime = strtotime($endDate);
$totalDays = ($endTime - $startTime) / 86400;

$remainingDays = ($endTime - strtotime($changeDate)) / 86400;
$prorateAmount = ($monthlyPrice / $totalDays) * $remainingDays;
```

## Calculation Examples

### Upgrade Calculation

```php
// Input
$currentPrice = 20.00;      // Basic plan
$newPrice = 30.00;          // Pro plan
$billingCycle = 'monthly';
$changeDate = '2024-05-15';
$cycleEnd = '2024-05-31';
$daysInMay = 31;
$daysRemaining = 17;

// Calculation
$dailyRate = 30.00 / 31;    // $0.9677 per day
$oldDailyRate = 20.00 / 31; // $0.6452 per day

$newProrate = 30.00 * (17 / 31);  // $16.45
$oldProrate = 20.00 * (17 / 31);  // $10.97

$upgradeCharge = $newProrate - $oldProrate;  // $5.48
```

### Downgrade Calculation

```php
// Input
$currentPrice = 30.00;      // Pro plan
$newPrice = 20.00;          // Basic plan

// Calculation
$oldProrate = 30.00 * (17 / 31);  // $16.45
$newProrate = 20.00 * (17 / 31);  // $10.97

$creditAmount = $oldProrate - $newProrate;  // $5.48
```

### With Tax Calculation

```php
// Input
$prorateAmount = 5.48;
$taxRate = 20;  // 20%

// Calculation
$taxAmount = $prorateAmount * (20 / 100);  // $1.10
$totalCharge = $prorateAmount + $taxAmount; // $6.58
```

## Prorate Formula Reference

### Monthly Prorate

```
Prorate = (Price / Days in Month) * Remaining Days

Example: $30/month, 17 days remaining in May (31 days)
Prorate = ($30 / 31) * 17 = $16.45
```

### Quarterly Prorate

```
Prorate = (Price / 90) * Remaining Days
Prorate = ($90 / 90) * 17 = $17.00
```

### Annual Prorate

```
Prorate = (Price / 365) * Remaining Days
Prorate = ($300 / 365) * 17 = $13.97
```

## Calculator Interface

### Admin Tool Interface

```html
<!-- Prorate Calculator Form -->
<form method="post" action="configbillingprorate.php">
    <label>Current Price</label>
    <input type="number" name="current_price" step="0.01" value="20.00">
    
    <label>New Price</label>
    <input type="number" name="new_price" step="0.01" value="30.00">
    
    <label>Billing Cycle</label>
    <select name="billing_cycle">
        <option value="monthly">Monthly</option>
        <option value="quarterly">Quarterly</option>
        <option value="annually">Annually</option>
    </select>
    
    <label>Change Date</label>
    <input type="date" name="change_date" value="2024-05-15">
    
    <label>Cycle End Date</label>
    <input type="date" name="cycle_end" value="2024-05-31">
    
    <label>Include Tax</label>
    <input type="checkbox" name="include_tax">
    
    <button type="submit">Calculate</button>
</form>
```

### Results Display

```html
<!-- Calculation Results -->
<div class="prorate-results">
    <h3>Prorate Calculation Results</h3>
    
    <table>
        <tr>
            <td>Current Price (monthly):</td>
            <td>$20.00</td>
        </tr>
        <tr>
            <td>New Price (monthly):</td>
            <td>$30.00</td>
        </tr>
        <tr>
            <td>Change Date:</td>
            <td>May 15, 2024</td>
        </tr>
        <tr>
            <td>Days Remaining:</td>
            <td>17</td>
        </tr>
        <tr>
            <td>Days in Month:</td>
            <td>31</td>
        </tr>
        <tr>
            <td><strong>Upgrade Charge:</strong></td>
            <td><strong>$5.48</strong></td>
        </tr>
        <tr>
            <td>Tax (20%):</td>
            <td>$1.10</td>
        </tr>
        <tr>
            <td><strong>Total Charge:</strong></td>
            <td><strong>$6.58</strong></td>
        </tr>
    </table>
</div>
```

## API Integration

### CalculateProrate API

```php
// Local API call
$params = [
    'action' => 'CalculateProrate',
    'currentprice' => 20.00,
    'newprice' => 30.00,
    'billingcycle' => 'monthly',
    'changedate' => '2024-05-15',
    'cycleenddate' => '2024-05-31',
    'includetax' => true,
    'taxrate' => 20
];

$result = localAPI('CalculateProrate', $params);
```

### Response

```json
{
    "result": "success",
    "currentprice": 20.00,
    "newprice": 30.00,
    "dailyrate": 0.9677,
    "daysremaining": 17,
    "prorateamount": 16.45,
    "oldprorateamount": 10.97,
    "difference": 5.48,
    "tax": 1.10,
    "total": 6.58
}
```

## Custom Calculator Implementation

### Standalone Calculator

```php
<?php
// custom-prorate-calculator.php

function calculateProrate($currentPrice, $newPrice, $changeDate, $cycleEnd, $billingCycle = 'monthly') {
    $daysInCycle = daysBetween($changeDate, $cycleEnd);
    
    switch ($billingCycle) {
        case 'monthly':
            $totalDays = date('t', strtotime($changeDate));
            break;
        case 'quarterly':
            $totalDays = 90;
            break;
        case 'annually':
            $totalDays = 365;
            break;
        default:
            $totalDays = 30;
    }
    
    $dailyRate = $newPrice / $totalDays;
    $newProrate = $dailyRate * $daysInCycle;
    
    $oldDailyRate = $currentPrice / $totalDays;
    $oldProrate = $oldDailyRate * $daysInCycle;
    
    return [
        'new_prorate' => round($newProrate, 2),
        'old_prorate' => round($oldProrate, 2),
        'difference' => round($newProrate - $oldProrate, 2),
        'days_remaining' => $daysInCycle
    ];
}

function daysBetween($start, $end) {
    $startTime = strtotime($start);
    $endTime = strtotime($end);
    return ($endTime - $startTime) / 86400;
}
?>
```

### JavaScript Calculator

```javascript
function calculateProrate() {
    const currentPrice = parseFloat(document.getElementById('current_price').value);
    const newPrice = parseFloat(document.getElementById('new_price').value);
    const changeDate = new Date(document.getElementById('change_date').value);
    const cycleEnd = new Date(document.getElementById('cycle_end').value);
    
    const daysInMonth = new Date(changeDate.getFullYear(), changeDate.getMonth() + 1, 0).getDate();
    const totalDays = (cycleEnd - changeDate) / (1000 * 60 * 60 * 24) + 1;
    
    const newDailyRate = newPrice / daysInMonth;
    const oldDailyRate = currentPrice / daysInMonth;
    
    const newProrate = newDailyRate * totalDays;
    const oldProrate = oldDailyRate * totalDays;
    
    const difference = newProrate - oldProrate;
    
    document.getElementById('result').innerHTML = `
        New Prorate: $${newProrate.toFixed(2)}<br>
        Old Prorate: $${oldProrate.toFixed(2)}<br>
        Difference: $${difference.toFixed(2)}
    `;
}
```

## Common Calculations

### Monthly to Monthly

```
Input: $20 -> $30, Day 15 of 31-day month
Output: $5.48 charge
```

### Monthly to Quarterly

```
Input: $20/mo -> $50/qtr, Day 15 of 31-day month
Consider: Convert first, then prorate
```

### Annual Reduction

```
Input: $100/year, downgrade after 180 days
Credit: $50 - (days remaining impact)
```

## Edge Case Handling

### Month-End Dates

```php
// Day 31 handling for short months
$day = 31;
$month = 2;  // February

if ($day > daysInMonth($month, $year)) {
    $day = daysInMonth($month, $year);
}
```

### Leap Years

```php
// February in leap year
$isLeapYear = date('L', strtotime($year . '-02-01'));
$daysInFeb = $isLeapYear ? 29 : 28;
```

## Troubleshooting

### Calculator Shows Zero

- Check date format (Y-m-d)
- Verify days remaining calculation
- Confirm prices are non-zero

### Unexpected Amounts

- Verify billing cycle selection
- Check for price override
- Review tax configuration

## Related Documentation

- [Pro-Rated Billing](./whmcs-pro-rated-billing.md)
- [Service Upgrade](./whmcs-service-upgrade.md)
- [Service Downgrade](./whmcs-service-downgrade.md)
- [Invoice Generation](./whmcs-invoice-generation.md)
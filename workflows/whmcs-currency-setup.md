# WHMCS Currency Configuration Workflow

## Purpose
Set up multiple currencies in WHMCS

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Currency Settings

Navigate to: Setup > Payments > Currencies

## Step 2: Add Base Currency

1. Click "Add Currency"
2. Configure:
   ```
   Currency Code: USD
   Currency Symbol: $
   Format: $1,000.00
   Base Currency: Yes
   ```
3. Save

## Step 3: Add Additional Currencies

### Add EUR
```
Currency Code: EUR
Currency Symbol: €
Format: €1.000,00
Base Currency: No
```

### Add GBP
```
Currency Code: GBP
Currency Symbol: £
Format: £1,000.00
Base Currency: No
```

## Step 4: Configure Exchange Rates

Navigate to: Setup > Payments > Currencies

### Manual Rates
1. Edit currency
2. Set rate relative to base:
   ```
   1 USD = 0.85 EUR
   1 USD = 0.72 GBP
   ```

### Auto-Update
```
Auto-Update Exchange Rates: Yes
Update Frequency: Daily
API Provider: Open Exchange Rates / Fixer.io
API Key: [your-key]
```

## Step 5: Set Currency Display Options

Navigate to: Setup > Payments > Payment Gateway Settings

```
Display Currencies: All / Base Only
Client Currency Selection: Enabled
Default Currency: [select]
```

## Step 6: Configure Per-Product Pricing

Navigate to: Setup > Products/Services > Products/Services

1. Edit product
2. Go to "Pricing" tab
3. Enter prices for each currency

## Step 7: Set Up Currency Conversion Fees

Navigate to: Setup > Payments > Currencies

```
Currency Conversion Fee: 2%
Round to Nearest: 0.01
```

## Currency Checklist

- [ ] Base currency set
- [ ] Additional currencies added
- [ ] Exchange rates configured
- [ ] Auto-update enabled (optional)
- [ ] Product pricing set per currency
- [ ] Conversion fees configured

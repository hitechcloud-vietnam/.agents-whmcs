# WHMCS Tax Rule Setup Workflow

## Purpose
Configure tax rules for invoices and products

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Tax Settings

Navigate to: Setup > Payments > Tax Rules

## Step 2: Enable Tax

```
Enable Tax: Yes
Tax Level: Single / Dual Level
```

## Step 3: Configure First Tax Level

1. Click "Add Tax Rule"
2. Configure:
   ```
   Tax Name: VAT
   Tax Rate: 20%
   Country: All Countries / Select Specific
   State/Region: All / Select Specific
   Applies To: Both / Products Only / Services Only
   ```
3. Save

## Step 4: Configure Dual Tax (Optional)

For US-style sales tax:

### Tax 1: State Tax
```
Tax Name: State Tax
Tax Rate: 6%
Applies To: Products and Services
```

### Tax 2: Local Tax
```
Tax Name: Local Tax
Tax Rate: 2%
Applies To: Products and Services
```

## Step 5: Set Tax Exemption Rules

Navigate to: Setup > Payments > Tax Rules > Exemptions

### Add Exemption
1. Click "Add Tax Exemption"
2. Configure:
   ```
   Type: Country / State / EU VAT Number
   Country: Germany
   Proof Required: Yes
   ```
3. Save

## Step 6: Configure EU VAT

Navigate to: Setup > Payments > Tax Rules > EU VAT Settings

```
Enable EU VAT: Yes
Apply Digital Services Tax: Yes
Threshold: €10,000
Verify VAT Numbers: Yes (VIES)
```

## Step 7: Set Tax Display

Navigate to: Setup > Payments > Invoice Settings

```
Show Tax ID Field: Yes
Tax Invoice Prefix: INV-
Separate Tax Lines: Yes
Include Tax in Prices: No / Yes
```

## Step 8: Configure Tax on Fees

Navigate to: Setup > Payments > Tax Rules

```
Apply Tax to Setup Fees: Yes
Apply Tax to Cancellation Fees: Yes
Apply Tax to Late Fees: Yes
Apply Tax to Domain Renewals: Yes
```

## Tax Rule Checklist

- [ ] Tax enabled
- [ ] Tax levels configured
- [ ] Country/state rules set
- [ ] Exemptions configured
- [ ] EU VAT configured
- [ ] Display settings set
- [ ] Fee taxation configured

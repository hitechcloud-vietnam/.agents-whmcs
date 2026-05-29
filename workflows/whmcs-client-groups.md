# WHMCS Client Groups Setup Workflow

## Purpose
Configure client groups for segmentation and pricing

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Client Groups

Navigate to: Setup > Clients > Client Groups

## Step 2: Create Client Group

### Create VIP Group
1. Click "Create New Group"
2. Configure:
   ```
   Group Name: VIP
   Group Colour: Gold (#FFD700)
   Group Description: Premium customers with priority support
   ```
3. Save

### Create Reseller Group
```
Group Name: Reseller
Group Colour: Blue (#0073aa)
Group Description: Reseller accounts with bulk pricing
Discount: 10%
```

### Create Government/Education
```
Group Name: Education
Group Colour: Green (#46b450)
Group Description: Educational institutions
Discount: 15%
```

## Step 3: Configure Group Discounts

Navigate to: Edit Group > Discounts

```
Default Discount: 5%
Discount Type: Percentage / Fixed Amount
Apply to: Products / Addons / Both
Excluded Products: [select]
```

## Step 4: Set Group-Specific Pricing

Navigate to: Setup > Products/Services > Products/Services

1. Edit product
2. Go to "Pricing" tab
3. Set prices per client group:
   ```
   Default: $10.00
   VIP: $9.00
   Reseller: $8.00
   Education: $7.00
   ```

## Step 5: Configure Group Auto-Assignment

Navigate to: Setup > Clients > Client Groups > Auto Assignment

### Create Rule
```
Rule Name: VIP Upgrade
Conditions:
  - Total Spent > $1000
  - Orders Count > 10
Action: Move to VIP Group
```

## Step 6: Set Group Permissions

Navigate to: Edit Group > Permissions

```
Can Order: Yes
Can Access Downloads: Yes
Can Access Support: Yes
Payment Terms: Net 30
Custom Suspension Fee: $0
```

## Step 7: Configure Group Branding

Navigate to: Edit Group > Branding

```
Custom Welcome Email: Yes
Custom Invoice Footer: Yes
Custom Support Priority: High
```

## Step 8: Assign Clients to Groups

Navigate to: Clients > Client Name > Edit

```
Client Group: VIP
```

### Bulk Assignment
Navigate to: Configuration > System Settings > Tools > Bulk Client Update

1. Select clients
2. Choose action: "Change Client Group"
3. Select target group

## Client Groups Checklist

- [ ] Groups created
- [ ] Discounts configured
- [ ] Group pricing set
- [ ] Auto-assignment rules created
- [ ] Permissions configured
- [ ] Branding customized
- [ ] Clients assigned

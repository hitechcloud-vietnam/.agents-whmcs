# WHMCS Custom Fields Setup Workflow

## Purpose
Create custom fields for clients and products

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Custom Fields

Navigate to: Setup > Clients > Custom Fields

## Step 2: Create Client Custom Fields

### Field 1: Company Name
```
Field Name: Company Name
Field Type: Text
Required: No
Show on Invoice: No
Registration Only: No
Field Order: 1
```

### Field 2: VAT Number
```
Field Name: VAT Number
Field Type: Text
Validation: VAT Number format
Required: No
Show on Invoice: Yes
Field Order: 2
```

### Field 3: How Did You Hear About Us
```
Field Name: How did you hear about us?
Field Type: Dropdown
Required: No
Options:
  - Google Search
  - Social Media
  - Friend Referral
  - Other
Field Order: 3
```

### Field 4: Preferred Contact Method
```
Field Name: Preferred Contact
Field Type: Radio
Required: No
Options: Email, Phone, Both
```

## Step 3: Configure Product Custom Fields

Navigate to: Setup > Products/Services > Custom Fields

### Create Field
```
Field Name: Server Location
Field Type: Dropdown
Options:
  - US East (Virginia)
  - US West (California)
  - EU (Frankfurt)
Required: No
Show on Invoice: No
Admin Only: No
```

## Step 4: Set Field Display Rules

Navigate to: Edit Custom Field > Display Rules

### Create Rule
```
If: [Product Group] equals [VPS]
Then: Show [Server Location] field
```

## Step 5: Configure Field Validation

Navigate to: Edit Custom Field > Validation

### Regex Validation Examples
```
Phone: ^\+?[1-9]\d{1,14}$
Email: [valid email regex]
VAT: ^[A-Z]{2}[0-9A-Z]{2,10}$
Postcode: [country-specific regex]
```

## Step 6: Set Field Permissions

Navigate to: Edit Custom Field > Permissions

```
Show to Clients: Yes
Editable by Clients: Yes
Required for Registration: No
Admin Only: No
```

## Step 7: Use Custom Fields in Templates

### In Email Templates
```smarty
Client Custom Field: {$customfield_vatnumber}
```

### In Invoice Templates
```smarty
{$customfield_vatnumber}
```

### In Products/Services
```smarty
Service Custom Field: {$servicecustomfield_serverlocation}
```

## Step 8: Export/Import Custom Fields

Navigate to: Setup > Clients > Custom Fields > Import/Export

Export format: CSV
Fields: Name, Type, Required, Options, Order

## Custom Fields Checklist

- [ ] Client fields created
- [ ] Product fields created
- [ ] Display rules configured
- [ ] Validation set
- [ ] Permissions configured
- [ ] Templates updated

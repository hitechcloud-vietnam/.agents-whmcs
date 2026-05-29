# WHMCS Service Groups Setup Workflow

## Purpose
Organize services/products into groups for management

## Prerequisites
- WHMCS installed
- Products created
- Admin access

## Step 1: Navigate to Service Groups

Navigate to: Setup > Products/Services > Service Groups

## Step 2: Create Service Group

### Create Hosting Group
1. Click "Create New Group"
2. Configure:
   ```
   Group Name: Web Hosting
   Display Name: Hosting Services
   Description: All hosting plans and packages
   ```
3. Save

### Create Servers Group
```
Group Name: Virtual Servers
Display Name: VPS Services
Description: Virtual private servers
```

## Step 3: Configure Group Display

Navigate to: Edit Group > Display Settings

```
Show Group in Client Area: Yes
Show Products as: List / Grid
Sort Products by: Name / Price / Popularity
Default Sort: Name
```

## Step 4: Set Group Icon/Image

Navigate to: Edit Group > Appearance

```
Icon: [upload icon]
Image: [upload header image]
Color Theme: Blue
```

## Step 5: Assign Products to Group

Navigate to: Edit Product > General

```
Product Group: Web Hosting
```

## Step 6: Configure Group Ordering

Navigate to: Edit Group > Order Settings

```
Allow Ordering: Yes
Featured Group: Yes
Show in Homepage: Yes
```

## Step 7: Set Group-Specific Options

Navigate to: Edit Group > Options

```
Require Domain: Yes
Show Domain Selection: Yes
Default Billing Cycle: Monthly
```

## Service Groups Checklist

- [ ] Groups created
- [ ] Display configured
- [ ] Icons/images set
- [ ] Products assigned
- [ ] Order settings configured
- [ ] Group options set

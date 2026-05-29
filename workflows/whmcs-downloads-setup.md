# WHMCS Downloads Module Setup Workflow

## Purpose
Configure file downloads module for client resources

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Downloads

Navigate to: Setup > Support > Downloads

## Step 2: Enable Downloads Module

```
Enable Downloads: Yes
Show in Client Area: Yes
```

## Step 3: Create Download Categories

Navigate to: Setup > Support > Downloads > Categories

### Create Category
1. Click "Add Category"
2. Configure:
   ```
   Category Name: Documentation
   Parent Category: None
   Description: Product documentation
   Display Order: 1
   ```
3. Save

### Recommended Categories
- Product Documentation
- Software Downloads
- Legal Documents
- Marketing Materials
- API Documentation

## Step 4: Set Category Permissions

For each category:
```
Access Control: Public / Clients Only / Specific Products
Client Groups: [select allowed groups]
Required Products: [if applicable]
```

## Step 5: Upload Downloads

Navigate to: Setup > Support > Downloads > Downloads

### Upload File
1. Click "Upload Download"
2. Configure:
   ```
   Title: User Manual v2.0
   Category: Documentation
   Description: Complete user guide
   File: [upload PDF]
   Downloads Allowed: Everyone
   ```
3. Save

## Step 6: Configure Download Options

### File Settings
```
Download Type: File Upload / Remote URL
Download Limit: 0 (unlimited)
Show Download Count: Yes
Require Agree to Terms: No
```

### Security Settings
```
Download Method: Direct / Force Download
Hotlink Protection: Yes
Verify User Ownership: No
Log Downloads: Yes
```

## Step 7: Set Up Download Groups

Navigate to: Setup > Support > Downloads > Download Groups

Create groups for organizing related files:
```
Group: Hosting
  - cPanel Guide
  - Email Setup
  - FTP Instructions
```

## Step 8: Configure Download Templates

Navigate to: Setup > Support > Downloads > Template Settings

```
Show Category Description: Yes
Show File Descriptions: Yes
Show Download Buttons: Yes
Columns: Title | Description | Size | Downloads
```

## Step 9: Add Product Associations

Navigate to: Setup > Support > Downloads > Downloads

1. Edit download
2. Set "Related Products":
   ```
   Products: Starter Hosting, Business Hosting
   ```
3. Downloads show in client product area

## Step 10: Configure Download Notifications

Navigate to: Setup > Support > Downloads > Notification Settings

```
New Download Alert: No
Popular Download Alert: No (after X downloads)
```

## Downloads Module Checklist

- [ ] Downloads enabled
- [ ] Categories created
- [ ] Permissions set
- [ ] Files uploaded
- [ ] Security configured
- [ ] Product associations set
- [ ] Template customized

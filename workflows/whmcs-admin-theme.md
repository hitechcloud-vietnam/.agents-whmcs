# WHMCS Admin Area Customization Workflow

## Purpose
Customize WHMCS admin area appearance and functionality

## Prerequisites
- WHMCS installed
- Admin access
- Development knowledge (optional)

## Step 1: Access Admin Settings

Navigate to: Configuration > System Settings > General

## Step 2: Configure Admin Theme

Navigate to: Configuration > System Settings > Admin Appearance

```
Admin Theme: Default / Dark Mode
Sidebar Style: Collapsed / Expanded
Dashboard View: Compact / Standard
```

## Step 3: Customize Admin Branding

Navigate to: Configuration > System Settings > Admin Customisation

```
Admin Area Logo: [upload logo]
Admin Area Title: WHMCS Admin
Show Version Number: Yes/No
```

## Step 4: Configure Admin Sidebar

Navigate to: Configuration > System Settings > Admin Sidebar

```
Default Navigation: Standard
Quick Navigation Shortcuts: [configure]
Recent Activity: Enabled
```

## Step 5: Set Up Admin Dashboard Widgets

Navigate to: Configuration > System Settings > Admin Home

### Available Widgets
- System Health
- Recent Orders
- Pending Tickets
- Invoice Summary
- Activity Log
- Quick Add
- Network Status

Configure widget layout and visibility.

## Step 6: Configure Admin Notifications

Navigate to: Configuration > System Settings > Admin Notifications

```
New Order Notification: Yes
New Ticket Notification: Yes
Payment Received Notification: Yes
Domain Expiry Notification: Yes
Invoice Overdue Notification: Yes
```

## Step 7: Customize Admin Emails

Navigate to: Configuration > System Settings > Admin Email Templates

Edit notification templates:
- New Admin Login Alert
- Admin Password Reset
- Ticket Assigned Alert

## Step 8: Set Up Admin Access Logs

Navigate to: Configuration > System Settings > Admin Activity Log

```
Log Admin Logins: Yes
Log Admin Actions: Yes
Log Retention: 90 days
```

## Step 9: Configure Admin Security

Navigate to: Configuration > System Settings > Admin Security

```
Require Two-Factor: Yes
Session Timeout: 60 minutes
IP Restriction: [optional]
Failed Login Limit: 5
```

## Step 10: Create Custom Admin CSS

```bash
nano /var/www/whmcs/admin/style/custom.css
```

```css
/* Custom Admin Styles */
.admin-logo {
    max-height: 40px;
}

.sidebar {
    background-color: #23282d;
}

.btn-admin {
    background-color: #0073aa;
    border-radius: 4px;
}
```

## Admin Customization Checklist

- [ ] Admin theme configured
- [ ] Branding customized
- [ ] Sidebar configured
- [ ] Dashboard widgets set
- [ ] Notifications enabled
- [ ] Email templates customized
- [ ] Access logs configured
- [ ] Security settings applied

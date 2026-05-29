# WHMCS Maintenance Mode Workflow

## Purpose
Enable and configure WHMCS maintenance mode

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Enable Maintenance Mode

Navigate to: Setup > General Settings > Maintenance Mode

### Settings
```
Enable Maintenance Mode: Yes
Maintenance Message: System maintenance in progress. Please check back soon.
Allowed IP Addresses: [your IP for testing]
Show Maintenance Page: Yes
```

## Step 2: Customize Maintenance Page

Create custom template at: `/var/www/whmcs/templates/[theme]/maintenance.tpl`

```smarty
<!DOCTYPE html>
<html>
<head>
    <title>Maintenance - {$companyname}</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #0073aa, #00a0d2);
            color: white;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
            text-align: center;
        }
        .container {
            max-width: 600px;
            padding: 40px;
        }
        h1 {
            font-size: 48px;
            margin-bottom: 20px;
        }
        p {
            font-size: 18px;
            line-height: 1.6;
        }
        .countdown {
            font-size: 24px;
            margin-top: 30px;
        }
        .contact {
            margin-top: 40px;
            font-size: 14px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Under Maintenance</h1>
        <p>{$maintenance_message}</p>
        <p>We apologize for any inconvenience.</p>
        <div class="contact">
            <p>Need urgent assistance?</p>
            <p>Email: {$company_email}</p>
            <p>Phone: {$company_phone}</p>
        </div>
    </div>
</body>
</html>
```

## Step 3: Allow Admin Access During Maintenance

Navigate to: Setup > General Settings > Maintenance Mode

Add IP addresses:
```
Allowed IPs:
- 192.168.1.1
- 203.0.113.50
```

## Step 4: Test Maintenance Mode

1. Visit WHMCS from allowed IP
2. Verify admin area accessible
3. Check maintenance page from other IP

## Step 5: Schedule Maintenance Window

### Plan Maintenance
```
Duration: 30 minutes
Start Time: 2:00 AM UTC
End Time: 2:30 AM UTC
```

### Notify Users
1. Send email notification
2. Post on status page
3. Update social media

## Step 6: Perform Maintenance Tasks

Common tasks:
- Database updates
- Plugin updates
- Security patches
- System upgrades

## Step 7: Disable Maintenance Mode

Navigate to: Setup > General Settings > Maintenance Mode

```
Enable Maintenance Mode: No
```

## Step 8: Verify System Restored

1. Test client area
2. Test admin area
3. Run test order
4. Verify cron running

## Maintenance Mode Checklist

- [ ] Maintenance enabled
- [ ] Custom page created
- [ ] Admin IPs allowed
- [ ] Maintenance tested
- [ ] Maintenance tasks completed
- [ ] Maintenance disabled
- [ ] System verified

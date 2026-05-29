# WHMCS Access Audit Workflow

## Purpose
Audit and review system access

## Prerequisites
- Admin access
- SSH access

## Step 1: Review Admin Accounts

Navigate to: Configuration > System Settings > Administrators

Review:
- All admin accounts
- Last login date
- Permission levels
- Account status

## Step 2: Identify Unused Accounts

Check admin accounts with:
- No recent login (90+ days)
- Unnecessary permissions
- Duplicate access

## Step 3: Review API Credentials

Navigate to: Setup > System Settings > API Credentials

Review:
- Active API keys
- IP restrictions
- Permission levels
- Last used date

## Step 4: Review Admin Activity

Navigate to: Configuration > System Settings > Admin Activity Log

Check:
- Recent admin actions
- Bulk operations
- Failed access attempts

## Step 5: Review Server Access

```bash
# Check SSH access
grep "sshd" /var/log/auth.log | tail -50

# Check FTP access
grep "ftpd" /var/log/proftpd/auth.log | tail -50
```

## Step 6: Check File Permissions

```bash
ls -la /var/www/whmcs/configuration.php
ls -la /var/www/whmcs/admin/
```

## Step 7: Review Firewall Rules

```bash
# UFW
ufw status numbered

# firewalld
firewall-cmd --list-all
```

## Step 8: Document Access Summary

Create report:
```
Admin Accounts: X active, Y inactive
API Keys: X active
Last Audit: [date]
Issues Found: [list]
Recommendations: [list]
```

## Access Audit Checklist

- [ ] Admin accounts reviewed
- [ ] Unused accounts identified
- [ ] API credentials reviewed
- [ ] Activity log reviewed
- [ ] Server access checked
- [ ] Permissions verified
- [ ] Firewall rules reviewed
- [ ] Summary documented

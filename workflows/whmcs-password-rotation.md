# WHMCS Password Rotation Workflow

## Purpose
Implement and manage password rotation policy

## Prerequisites
- Admin access
- SSH access

## Step 1: Review Current Password Policy

Navigate to: Setup > General Settings > Security

Check current settings:
- Minimum password length
- Password requirements
- Password expiration

## Step 2: Configure Password Policy

### Admin Passwords
Navigate to: Configuration > System Settings > Admin Security

```
Minimum Password Length: 16
Require Uppercase: Yes
Require Lowercase: Yes
Require Number: Yes
Require Special Character: Yes
Password Expiration: 90 days
```

### Client Passwords
Navigate to: Setup > General Settings > Security

```
Minimum Password Length: 12
Password Requirements: Standard
Force Periodic Reset: No (or configure)
```

## Step 3: Update Admin Passwords

Navigate to: Configuration > System Settings > Administrators

For each admin:
1. Click username
2. Generate new secure password
3. Update password
4. Store securely (password manager)

## Step 4: Update Database Password

### Generate New Password
```bash
openssl rand -base64 32
```

### Update MySQL User
```sql
ALTER USER 'whmcs_user'@'localhost' IDENTIFIED BY 'new_password';
FLUSH PRIVILEGES;
```

### Update WHMCS Configuration
```bash
nano /var/www/whmcs/configuration.php
```

Update:
```php
$mysql_password = 'new_password';
```

## Step 5: Update Integration Passwords

Review and rotate:
- Payment gateway API keys
- Domain registrar credentials
- Server module passwords
- SMTP credentials

## Step 6: Document Password Changes

Maintain password log:
```
Date: [date]
Password Type: [type]
Updated By: [name]
Next Rotation: [date]
```

## Password Rotation Checklist

- [ ] Password policy reviewed
- [ ] Policy configured
- [ ] Admin passwords updated
- [ ] Database password rotated
- [ ] Integration passwords updated
- [ ] Changes documented

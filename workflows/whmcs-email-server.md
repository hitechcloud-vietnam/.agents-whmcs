# WHMCS Email Server Configuration Workflow

## Description
Configure dedicated email server for WHMCS to improve email deliverability and reliability.

## Prerequisites
- Dedicated mail server or email service
- SSH access
- Domain with proper DNS records
- SSL certificates for SMTP

## Architecture
```
[WHMCS Server]  --->  [Mail Server]  --->  [Recipient Servers]
   10.0.0.10          10.0.0.30           Gmail, Outlook, etc.
```

## Steps

### Step 1: Install Mail Transfer Agent
```bash
# Option A: Postfix (recommended for dedicated server)
apt update && apt install -y postfix postfix-mysql

# Choose "Internet Site" when prompted
# Or configure manually:
cat > /etc/postfix/main.cf << 'EOF'
myhostname = mail.example.com
myorigin = /etc/mailname
mydestination = localhost.localdomain, localhost
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
EOF

# Option B: Exim
apt install -y exim4-daemon-heavy
```

### Step 2: Configure SMTP Authentication
```bash
# Install SASL for SMTP authentication
apt install -y libsasl2-modules

# Configure Postfix for SASL
cat >> /etc/postfix/main.cf << 'EOF'
# SASL Authentication
smtpd_sasl_auth_enable = yes
smtpd_sasl_path = private/auth
smtpd_sasl_type = dovecot
smtpd_sasl_authenticated_header = yes
smtpd_sasl_security_options = noanonymous
broken_sasl_auth_clients = yes

# TLS
smtpd_tls_cert_file = /etc/ssl/certs/mail.pem
smtpd_tls_key_file = /etc/ssl/private/mail.key
smtpd_tls_security_level = may
smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.crt
smtp_tls_security_level = may
smtp_tls_loglevel = 1
EOF

# Configure submission (port 587)
cat > /etc/postfix/master.cf << 'EOF'
submission inet n - y - - smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
EOF
```

### Step 3: Setup Dovecot for Authentication
```bash
# Install Dovecot
apt install -y dovecot-core dovecot-imapd

# Configure Dovecot
cat > /etc/dovecot/conf.d/10-auth.conf << 'EOF'
disable_plaintext_auth = no
auth_mechanisms = plain login
!include auth-system.conf.ext
EOF

cat > /etc/dovecot/conf.d/10-mail.conf << 'EOF'
mail_location = maildir:/var/mail/vhosts/%d/%n
mail_privileged_group = mail
EOF

cat > /etc/dovecot/conf.d/10-ssl.conf << 'EOF'
ssl = required
ssl_cert = </etc/ssl/certs/mail.pem
ssl_key = </etc/ssl/private/mail.key
EOF

# Create virtual mail users
groupadd -g 5000 vmail
useradd -g vmail -u 5000 vmail -d /var/mail -s /sbin/nologin

# Restart services
systemctl restart postfix
systemctl restart dovecot
systemctl enable postfix dovecot
```

### Step 4: Configure DNS Records
```bash
# Add DNS records for your mail server
# SPF Record
TXT record: v=spf1 mx a:mail.example.com ~all

# DKIM Record (for email authentication)
# Install OpenDKIM
apt install -y opendkim opendkim-tools

# Create DKIM keys
mkdir -p /etc/opendkim/keys/example.com
opendkim-genkey -D /etc/opendkim/keys/example.com -d example.com -s mail
mv /etc/opendkim/keys/example.com/mail.private /etc/opendkim/keys/example.com/default

# Configure OpenDKIM
cat > /etc/opendkim.conf << 'EOF'
AutoRestart         Yes
AutoRestartRate     10/1M
Background          Yes
Canonicalization    relaxed/simple
ExternalIgnoreList  refile:/etc/opendkim/TrustedHosts
InternalHosts       refile:/etc/opendkim/TrustedHosts
KeyTable            refile:/etc/opendkim/KeyTable
LogWhyMessages      Yes
MinimumKeyBits      1024
Modes               sv
OversignHeaders     From
Selector            default
Socket              inet:8891@localhost
Syslog              Yes
SyslogSuccess       Yes
UserID              opendkim:opendkim
EOF

# Add trusted hosts and key table
cat > /etc/opendkim/TrustedHosts << 'EOF'
127.0.0.1
localhost
*.example.com
EOF

cat > /etc/opendkim/KeyTable << 'EOF'
default._domainkey.example.com example.com:default:/etc/opendkim/keys/example.com/default
EOF

# Configure Postfix to use OpenDKIM
cat >> /etc/postfix/main.cf << 'EOF'
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
EOF

systemctl restart postfix opendkim
```

### Step 5: Configure WHMCS Email Settings
```php
// In configuration.php - Use remote SMTP server

$whmcs_config = [
    'mail_type' => 'smtp',
    'mail_from_email' => 'noreply@example.com',
    'mail_from_name' => 'Your Company Name',
    
    // SMTP Configuration
    'smtp_host' => 'mail.example.com',
    'smtp_port' => 587,
    'smtp_username' => 'whmcs@example.com',
    'smtp_password' => 'smtp_password_here',
    'smtp_security' => 'tls',  // tls or ssl
    'smtp_debug' => false,
];
```

### Step 6: Configure WHMCS Admin Email
1. Login to WHMCS Admin
2. Go to Configuration > System Settings > General
3. Set email address for admin notifications
4. Configure email templates

### Step 7: Test Email Configuration
```php
<?php
// Create test script: test-email.php
require_once __DIR__ . '/init.php';

use WHMCS\Mail\Log;

try {
    $mail = new WHMCS\Mail\Mailer();
    $mail->setSubject("Test Email from WHMCS");
    $mail->setBody("This is a test email to verify email configuration.");
    $mail->addRecipient("admin@example.com", "Admin");
    
    $result = $mail->send();
    
    if ($result === true) {
        echo "Email sent successfully!";
    } else {
        echo "Email failed: " . print_r($result, true);
    }
} catch (Exception $e) {
    echo "Error: " . $e->getMessage();
}
```

### Step 8: Email Queue Management
```bash
# Configure email queue processing
# Add to crontab
crontab -e

# Process email queue every 5 minutes
*/5 * * * * php -q /var/www/whmcs/crons/cron.php --do=cronjobs
```

### Step 9: Monitoring Email
```bash
# Create email monitoring script
cat > /usr/local/bin/whmcs-email-monitor.sh << 'EOF'
#!/bin/bash

MAILQ=$(mailq | grep -c "total requests")
ALERT_EMAIL="admin@example.com"

if [ $MAILQ -gt 100 ]; then
    echo "Warning: $MAILQ emails in queue" | \
        mail -s "WHMCS Email Queue Alert" $ALERT_EMAIL
fi

# Check mail log
grep -i "status=bounced\|status=deferred" /var/log/mail.log | \
    tail -20 >> /var/log/email_bounces.log
EOF

chmod +x /usr/local/bin/whmcs-email-monitor.sh
```

## Email Deliverability Checklist
- [ ] SPF record configured
- [ ] DKIM signature enabled
- [ ] DMARC policy set
- [ ] Reverse DNS (PTR) configured
- [ ] Proper TLS encryption
- [ ] Email authentication working
- [ ] Testing with mail-tester.com

## Troubleshooting
```bash
# Check mail queue
mailq
postqueue -p

# Flush queue
postqueue -f

# Check logs
tail -f /var/log/mail.log

# Test SMTP manually
telnet mail.example.com 25
EHLO test.com
AUTH LOGIN
```

## Tags
- email
- smtp
- mail-server
- deliverability
- configuration
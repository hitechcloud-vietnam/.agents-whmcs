# WHMCS SMTP Debug Workflow

## Purpose
Debug and troubleshoot SMTP email sending issues.

## Debug Steps

### Step 1: Enable Debug Logging
1. Navigate to: Configuration > System > Settings > Mail
2. Enable "Debug Mode" (if available)
3. Set log verbosity to "Verbose"

### Step 2: Test SMTP Connection
1. Use SMTP test tools
2. Check with: https://mxtoolbox.com/diagnostic.aspx
3. Verify all DNS records (SPF, DKIM, DMARC)

### Step 3: Review Server Logs
Check mail server logs:
- /var/log/maillog (Linux)
- Event Viewer (Windows)

### Step 4: Common SMTP Errors

#### Error: Authentication Failed
```
Solution:
- Verify username/password
- Check for special characters in password
- Use app-specific password if 2FA enabled
- Ensure SMTP auth is enabled on server
```

#### Error: Connection Timeout
```
Solution:
- Check firewall rules
- Verify port (587/465/25)
- Ensure TLS/SSL settings correct
- Test with telnet: telnet smtp.server.com 587
```

#### Error: TLS Handshake Failed
```
Solution:
- Check SSL certificate
- Verify TLS version compatibility
- Update CA certificates on server
- Try different encryption method
```

### Step 5: WHMCS-Specific Checks
1. Verify SMTP settings in WHMCS
2. Check configuration.php for mail overrides
3. Clear WHMCS cache
4. Test with different email address

### Step 6: Use Diagnostic Tools
- SMTP test: https://app.smtpcloud.com/tools
- MX lookup: https://mxtoolbox.com
- Blacklist check: https://mxtoolbox.com/blacklists.aspx
- SSL check: https://ssldecoder.org

## Debug Output Analysis

### SMTP Response Codes
- 220: Server ready
- 250: Request completed
- 334: Authentication prompt
- 235: Authentication successful
- 550: Mailbox unavailable
- 554: Transaction failed

### Log Reading
```
[2024-01-15 10:00:00] Connection: Connecting to smtp.example.com:587
[2024-01-15 10:00:01] Connection: TLS started
[2024-01-15 10:00:02] Auth: Authenticating with username
[2024-01-15 10:00:03] Auth: Authentication successful
[2024-01-15 10:00:04] Mail: Sending email to recipient@example.com
[2024-01-15 10:00:05] Success: Email sent successfully
```

## Related Workflows
- whmcs-email-smtp
- whmcs-email-sending
- whmcs-email-log
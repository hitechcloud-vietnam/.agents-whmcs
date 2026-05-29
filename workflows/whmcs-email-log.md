# WHMCS Email Logging Workflow

## Purpose
Monitor and review email sending history in WHMCS.

## Access Email Logs

### Step 1: Find Email Logs
1. Navigate to: Utilities > Logs > Activity Log
2. Filter by "Email" type
3. Set date range

### Step 2: Log Entry Details
Each log entry contains:
- Date/Time sent
- Recipient email
- Email subject
- Sender account
- Status (Sent/Failed)
- Related entity (Invoice, Order, etc.)

### Step 3: Analyze Logs
- Check for failed emails
- Identify delivery patterns
- Review bounce events
- Monitor sending volume

## Configuration

### Enable Detailed Logging
1. Navigate to: Configuration > System > Settings > Mail
2. Enable "Log emails"
3. Set retention period

### Log Retention
- Default: 30 days
- Configure in settings
- Export logs for long-term storage

## Log Entry States

### Sent Statuses
- **Sent**: Successfully delivered
- **In Queue**: Waiting to send
- **Pending**: Scheduled

### Failed Statuses
- **Failed**: Delivery failed
- **Bounced**: Rejected by recipient server
- **Invalid**: Invalid email address

## Troubleshooting via Logs

### Identifying Issues
1. Find failed email entries
2. Check error codes
3. Review recipient history
4. Identify patterns

### Common Error Codes
- 550: Mailbox not found
- 552: Message too large
- 554: Spam detected
- 421: Server busy, retry later

## Exporting Logs

### Step 1: Access Logs
1. Go to: Utilities > Logs > Activity Log
2. Filter as needed
3. Click "Export"

### Step 2: Choose Format
- CSV
- Excel
- PDF

## Integration with External Services

### Enable API Logging
1. Use third-party email service API
2. Enable webhook logging
3. Sync with email tracking service

## Related Workflows
- whmcs-email-tracking
- whmcs-email-queue
- whmcs-email-bounce
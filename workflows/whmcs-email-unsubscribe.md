# WHMCS Email Unsubscribe Handling Workflow

## Purpose
Manage email unsubscribes and ensure compliance with regulations.

## Compliance Requirements

### CAN-SPAM (USA)
- Clear unsubscribe mechanism
- Process within 10 business days
- Include physical address

### GDPR (EU)
- Explicit consent required
- Right to be forgotten
- Data minimization

### CASL (Canada)
- Express consent required
- Identify as advertising
- Provide contact information

## Configuration

### Step 1: Enable Unsubscribe Links
1. Navigate to: Configuration > System > Settings > Mail
2. Enable "Enable Unsubscribe Headers"
3. Set unsubscribe preference per email type

### Step 2: Configure Unsubscribe Behavior
```
Options:
- One-click unsubscribe: Yes/No
- Keep client preference: Yes/No
- Add to suppression list: Yes/No
- Require confirmation: Yes/No
```

### Step 3: Unsubscribe Page Setup
1. WHMCS provides default unsubscribe page
2. Customize page at: /unsubscribe.php
3. Set redirect after unsubscribe

## Automatic Processing

### Step 1: Cron Job
Ensure cron includes email processing:
```
/usr/bin/php -q /path/to/whmcs/admin/cron.php
```

### Step 2: Processing Flow
1. User clicks unsubscribe
2. System validates email
3. Preferences updated
4. Confirmation sent
5. Email blocked from future sends

## Unsubscribe Workflow

### User Experience
1. User clicks link in email
2. Lands on unsubscribe confirmation page
3. Selects email preferences
4. Confirms unsubscribe
5. Receives confirmation

### Admin View
1. Navigate to: Clients > Email Preferences
2. View all unsubscribe requests
3. Search by client/date
4. Export list

## Managing Unsubscribes

### Review Unsubscribe Log
1. Navigate to: Utilities > Logs > Unsubscribe Log
2. Analyze reasons for unsubscribe
3. Track trends
4. Improve email content

### Prevent Unsubscribes
1. Reduce email frequency
2. Improve content quality
3. Segment audiences
4. Personalize content

## Excluded Email Types
Some emails cannot be unsubscribed from:
- Security notifications
- Billing/invoice emails
- Legal notices
- Account verification

## Related Workflows
- whmcs-email-template-create
- whmcs-email-tracking
- whmcs-marketing-email
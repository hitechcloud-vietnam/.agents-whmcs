# WHMCS Support Ticket System Setup Workflow

## Purpose
Configure comprehensive support ticket system

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Support Settings

Navigate to: Setup > Support > Support Departments

## Step 2: Create Support Departments

### Department 1: General Support
```
Name: General Support
Email: support@yourdomain.com
Administrator: [select admin]
Display Order: 1
Client Visibility: Yes
```

### Department 2: Technical Support
```
Name: Technical Support
Email: tech@yourdomain.com
Administrator: [select senior admin]
Display Order: 2
Client Visibility: Yes
```

### Department 3: Billing
```
Name: Billing Department
Email: billing@yourdomain.com
Administrator: [select admin]
Display Order: 3
Client Visibility: Yes
```

## Step 3: Configure Department Settings

For each department, configure:

### General Tab
```
Department Name: [name]
Host Department: No
Client Visibility: Yes
Awaiting Reply Tab: Yes
Escalation Path: Yes
```

### Tickets Tab
```
Ticket Import: Enabled
Auto-Lock: Yes (after 7 days inactive)
Auto-Close: Yes (after 30 days)
Feedback Survey: Yes
Escalation Threshold: 4 hours
```

### Features Tab
```
Allow Sub-Tickets: Yes
Monitor Emails: Yes (IMAP)
Client Tickets Only: No
Priority Escalation: Enabled
```

## Step 4: Set Up Ticket Priorities

Navigate to: Setup > Support > Ticket Priorities

| Priority | Color | Auto-Response Time |
|----------|-------|-------------------|
| Critical | Red | 1 hour |
| High | Orange | 4 hours |
| Medium | Yellow | 12 hours |
| Low | Blue | 24 hours |

## Step 5: Configure Ticket Statuses

Navigate to: Setup > Support > Ticket Statuses

Default statuses:
- Open (Orange)
- Answered (Green)
- Customer-Reply (Blue)
- On Hold (Purple)
- Closed (Gray)

### Add Custom Status
```
Name: In Progress
Color: Cyan
State: Open
Sort Order: 3
```

## Step 6: Set Up Escalation Rules

Navigate to: Setup > Support > Escalation Rules

### Rule 1: No Response
```
Trigger: No response after X hours
Hours: 4
Action: Escalate to Senior
Priority: Increase to High
```

### Rule 2: SLA Breach
```
Trigger: SLA threshold exceeded
Action: Alert admin, escalate ticket
Notify: Email admin
```

### Rule 3: Auto-Lock
```
Trigger: No activity after days
Days: 7
Action: Lock ticket
```

## Step 7: Configure Email Integration

### Set Up IMAP Monitoring

Navigate to: Setup > Support > Support Departments > [Department] > Email Import

```
Enable IMAP Import: Yes
Mail Server: mail.yourdomain.com
Port: 993
Username: support@yourdomain.com
Password: [password]
Encryption: SSL
Folder: INBOX
Delete Imported: No
Create Ticket For: All emails / New Senders Only
```

### Configure Auto-Response
```
Send Auto-Response: Yes
Auto-Response Template: [select]
Delay (minutes): 0
```

## Step 8: Set Up Ticket Filtering

Navigate to: Setup > Support > Ticket Import Rules

### Filter Rule Example
```
Name: Spam Filter
Condition: Subject contains [spam keywords]
Action: Delete / Skip / Mark as Spam
```

## Step 9: Configure Canned Responses

Navigate to: Setup > Support > Canned Responses

### Create Response
```
Title: Password Reset Instructions
Categories: Technical Support
Response:
Dear {$client_name},

To reset your client area password:

1. Visit {$whmcs_url}/password-reminder.php
2. Enter your email address
3. Follow the link in the email

If you need further assistance, please reply to this ticket.

Best regards,
{$ticket_admin_name}
```

## Step 10: Set Up Knowledge Base Integration

Navigate to: Setup > Support > Knowledgebase Settings

```
Enable KB Suggestions: Yes
Show Related Articles: Yes
Require Login for KB: No
```

## Step 11: Configure Support Hours

Navigate to: Setup > Support > Support Hours

### Working Hours
```
Monday-Friday: 9:00 AM - 6:00 PM
Saturday: 10:00 AM - 2:00 PM
Sunday: Closed
Timezone: America/New_York
```

### Out of Hours
```
Show Expected Response Time: Yes
Auto-Response Message: [custom template]
```

## Step 12: Set Up SLA Policies

Navigate to: Setup > Support > SLA Policies

### Premium SLA
```
Name: Premium Support
First Response: 1 hour
Resolution Time: 4 hours
Business Hours: 24/7
Assigned To: [premium clients only]
```

### Standard SLA
```
Name: Standard Support
First Response: 4 hours
Resolution Time: 24 hours
Business Hours: Business Hours Only
```

## Step 13: Configure Ticket Notifications

Navigate to: Setup > Support > Ticket Notifications

```
New Ticket: Notify admin, department
Reply Added: Notify client, admin
Priority Changed: Notify admin
Status Changed: Notify client
Escalation: Notify admin
Auto-Close: Notify client
```

## Step 14: Set Up Feedback Survey

Navigate to: Setup > Support > Feedback Settings

```
Send Survey on Close: Yes
Survey Delay: 1 hour
Survey Questions:
- How would you rate our support? (1-5)
- Was your issue resolved? (Yes/No)
- Additional comments (text)
```

## Step 15: Test Ticket System

### Test Ticket Flow
1. Submit ticket via client area
2. Verify email received
3. Reply from admin area
4. Verify client notification
5. Close ticket
6. Verify feedback survey

## Ticket System Checklist

- [ ] Departments created
- [ ] Priorities configured
- [ ] Statuses set
- [ ] Escalation rules created
- [ ] Email integration working
- [ ] Canned responses created
- [ ] Knowledge base linked
- [ ] Support hours configured
- [ ] SLA policies set
- [ ] Notifications configured
- [ ] Feedback survey enabled
- [ ] Ticket flow tested

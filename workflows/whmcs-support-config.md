# WHMCS Support Department Setup Workflow

## Purpose
Configure support ticket system and departments

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Support Settings

Navigate to: Setup > Support > Support Departments

## Step 2: Create New Department

1. Click "Create New Department"
2. Configure:
   ```
   Department Name: Technical Support
   Department Email: support@yourdomain.com
   Administrator: [select admin]
   ```
3. Save

## Step 3: Configure Department Settings

### General Tab
```
Department Name: Technical Support
Display Order: 1
Client Visibility: Yes
Awaiting Reply Tab: Yes
Escalation Path: Yes
```

### Tickets Tab
```
Ticket Import: Enabled
Auto-Lock: Yes (after 7 days)
Auto-Close: Yes (after 30 days)
Feedback Survey: Yes
```

### Features Tab
```
Allow Sub-Tickets: Yes
Monitor Emails: Yes
Client Tickets Only: No
```

## Step 4: Set Up Escalation Rules

Navigate to: Setup > Support > Escalation Rules

1. Click "Add Escalation Rule"
2. Configure:
   ```
   Department: Technical Support
   Trigger: No Response for X hours
   Escalate To: Senior Support
   Priority: High
   ```
3. Save

## Step 5: Configure Ticket Priorities

Navigate to: Setup > Support > Ticket Priorities

Default priorities:
- High (Response: 4 hours)
- Medium (Response: 12 hours)
- Low (Response: 24 hours)

## Step 6: Set Up Ticket Statuses

Navigate to: Setup > Support > Ticket Statuses

Default statuses:
- Open
- Awaiting Reply
- In Progress
- Answered
- Closed

## Step 7: Configure Automated Responses

Navigate to: Setup > Email Templates > Support Ticket Templates

### Ticket Opened Template
```html
Dear {$client_name},

Your support ticket has been opened.

Ticket ID: {$ticket_id}
Subject: {$ticket_subject}
Priority: {$ticket_priority}

We will respond within 24 hours.

Best regards,
{$company_name} Support Team
```

## Step 8: Set Up Knowledge Base Integration

Navigate to: Setup > Support > Knowledgebase Settings

```
Allow KB Articles in Tickets: Yes
Suggest Related Articles: Yes
```

## Step 9: Configure Support Hours

Navigate to: Setup > Support > Support Hours

1. Create support hours:
   ```
   Monday-Friday: 9:00 AM - 6:00 PM
   Saturday: 10:00 AM - 2:00 PM
   Sunday: Closed
   ```
2. Configure out-of-hours message

## Step 10: Set Up Canned Responses

Navigate to: Setup > Support > Canned Responses

1. Click "Add Canned Response"
2. Configure:
   ```
   Title: Password Reset Instructions
   Response: Dear {$client_name},
   
   To reset your password, please follow these steps:
   1. Go to...
   ```
3. Save

## Step 11: Configure Ticket Routing

Navigate to: Setup > Support > Support Departments

1. Edit department
2. Set routing rules:
   ```
   Match Domain: @client-domain.com
   Route To: Enterprise Support
   ```

## Step 12: Set Up Service Level Agreements

Navigate to: Setup > Support > SLA Policies

1. Click "Create SLA Policy"
2. Configure:
   ```
   Name: Premium Support
   First Response: 1 hour
   Resolution Time: 4 hours
   Working Hours: 24/7
   ```
3. Assign to client groups

## Support Department Checklist

- [ ] Departments created
- [ ] Escalation rules configured
- [ ] Priorities set
- [ ] Statuses configured
- [ ] Auto-responses set
- [ ] Knowledge base integration
- [ ] Support hours configured
- [ ] Canned responses created
- [ ] Ticket routing configured
- [ ] SLA policies created

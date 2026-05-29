# WHMCS New Year Setup Workflow

## Purpose
Complete new year setup and configuration

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Review Annual Reports

Navigate to: Reports > Revenue > Annual Revenue

Generate year-end report:
- Total revenue
- New clients
- Churn rate
- Top products

## Step 2: Update Tax Settings

Navigate to: Setup > Payments > Tax Rules

Review and update:
- Tax rates
- Tax exemptions
- Regulatory compliance

## Step 3: Review Pricing

Navigate to: Setup > Products/Services > Products/Services

Review:
- Current pricing
- Cost changes
- Market rates
- Price adjustments needed

## Step 4: Update Terms and Conditions

Navigate to: Setup > General Settings > Legal Pages

Review and update:
- Terms of Service
- Privacy Policy
- Refund Policy
- Service Level Agreement

## Step 5: Review Contract End Dates

Navigate to: Clients > Services

Check:
- Annual contracts expiring
- Domain renewals
- SSL certificate expirations

## Step 6: Update Company Information

Navigate to: Setup > General Settings > General

Update:
- Company year
- Copyright notice
- Contact information

## Step 7: Review Goals and Targets

Set targets for new year:
- Revenue targets
- Client acquisition goals
- Churn targets
- Support metrics

## Step 8: Backup and Archive

```bash
# Create year-end backup
mkdir -p /backup/archives/2024
mysqldump -u root -p whmcs_db | gzip > /backup/archives/2024/whmcs_db_20241231.sql.gz

# Archive reports
mkdir -p /var/www/whmcs/reports/2024
```

## Step 9: Update Email Templates

Navigate to: Setup > Email > Email Templates

Review:
- Footer dates
- Current year references
- Regulatory language

## Step 10: Configure New Year Settings

Navigate to: Setup > General Settings > Automation

Set:
- New year automation schedule
- Invoice numbering reset
- Report schedules

## New Year Checklist

- [ ] Annual reports generated
- [ ] Tax settings reviewed
- [ ] Pricing reviewed
- [ ] Terms updated
- [ ] Contract dates reviewed
- [ ] Company info updated
- [ ] Goals set
- [ ] Backup created
- [ ] Email templates updated
- [ ] Automation configured

# WHMCS Invoice Report Workflow

## Purpose
Generate and analyze invoice reports in WHMCS.

## Invoice Report Access

### Navigation
1. Navigate to: Reports > Invoices
2. Select report type
3. Set date range

### Report Types
```
Available:
- Invoice Summary
- Invoice Details
- Overdue Invoices
- Invoice by Client
- Invoice by Product
```

## Invoice Summary

### Overview Metrics
```
Displays:
- Total Invoices
- Total Amount
- Paid Amount
- Outstanding Amount
- Overdue Amount
```

### Status Breakdown
```
By Status:
- Paid
- Unpaid
- Overdue
- Cancelled
- Draft
```

## Invoice Details

### Detailed Report
```
Shows:
- Invoice Number
- Date Created
- Due Date
- Amount
- Status
- Client Name
- Payment Method
```

### Filters
```
Apply:
- Date range
- Status
- Amount range
- Client group
```

## Overdue Invoice Report

### Overdue Analysis
```
Metrics:
- Number of overdue invoices
- Total overdue amount
- Average days overdue
- Oldest overdue invoice
```

### Aging Report
```
Breakdown:
- 1-30 days overdue
- 31-60 days overdue
- 61-90 days overdue
- 90+ days overdue
```

## Invoice by Client

### Client Analysis
```
Shows:
- Client name
- Number of invoices
- Total amount
- Outstanding amount
- Payment history
```

### Top Clients
```
Rank by:
- Invoice count
- Total amount
- Outstanding amount
- Payment timeliness
```

## Invoice by Product

### Product Breakdown
```
Shows:
- Product/Service
- Number of invoices
- Amount invoiced
- Payment status
```

## Exporting Invoice Reports

### Export Options
```
Formats:
- CSV
- Excel
- PDF
```

### Data Included
```
Export:
- All invoice details
- Line items
- Payment history
- Custom fields
```

## Scheduled Reports

### Automation
```
Set up:
- Daily outstanding invoice report
- Weekly overdue summary
- Monthly invoice analysis
```

## Best Practices

### Regular Monitoring
```
- Daily: Overdue check
- Weekly: Aging analysis
- Monthly: Comprehensive review
```

### Key Metrics
```
Track:
- Collection rate
- Average days to payment
- Overdue percentage
- Bad debt rate
```

## Related Workflows
- whmcs-report-revenue
- whmcs-report-payment
- whmcs-report-sales
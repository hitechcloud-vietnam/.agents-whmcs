# WHMCS Custom Report Workflow

## Purpose
Create and manage custom reports with specific metrics.

## Custom Report Builder

### Access
1. Navigate to: Reports > Custom
2. Click "Create Custom Report"
3. Select data source

### Data Sources
```
Available:
- Clients
- Orders
- Invoices
- Services
- Domains
- Support Tickets
- Affiliates
```

## Building Reports

### Step 1: Select Data
1. Choose primary table
2. Add related tables
3. Define relationships

### Step 2: Add Fields
```
Select Fields:
- Client name
- Order date
- Order total
- Invoice status
- Payment method
```

### Step 3: Add Filters
```
Filters:
- Date range
- Status
- Amount range
- Client group
- Custom conditions
```

### Step 4: Group and Aggregate
```
Aggregation:
- Sum
- Count
- Average
- Min/Max
- Group by field
```

### Step 5: Sort and Format
```
Sort by:
- Any field
- Ascending/Descending

Format:
- Number formatting
- Currency
- Date format
```

## Report Examples

### Monthly Sales by Product
```
Fields: Product, Units Sold, Revenue
Group: Product
Filter: Date range = This month
Sort: Revenue descending
```

### Client Lifetime Value
```
Fields: Client, Total Orders, Total Spent, First Order Date
Filter: Status = Active
Sort: Total Spent descending
```

### Ticket Response Time by Agent
```
Fields: Agent, Tickets Handled, Avg Response Time, Avg Resolution
Group: Agent
Filter: Date range = This month
```

## Saved Reports

### Save Custom Reports
```
Options:
- Save for personal use
- Share with team
- Schedule generation
```

### Templates
```
Create:
- Weekly metrics
- Monthly summary
- Quarterly review
```

## Scheduling

### Automated Reports
```
Set up:
- Daily generation
- Weekly email
- Monthly archive
```

### Delivery
```
Options:
- Email to admin
- Email to team
- Save to file
```

## Export Options

### Export Formats
```
Available:
- CSV
- Excel
- PDF
- Print
```

### Options
```
Choose:
- Include headers
- Format numbers
- Date format
```

## Charts and Visualization

### Chart Types
```
Options:
- Bar chart
- Line chart
- Pie chart
- Table
- Metric cards
```

### Configuration
```
Set:
- X-axis
- Y-axis
- Colors
- Labels
```

## Best Practices

### Report Design
```
Guidelines:
- Start with question
- Keep simple
- Focus on action
- Update regularly
```

### Organization
```
Create:
- Folders for categories
- Naming convention
- Regular cleanup
```

## Related Workflows
- whmcs-report-export
- whmcs-report-schedule
- whmcs-report-dashboard
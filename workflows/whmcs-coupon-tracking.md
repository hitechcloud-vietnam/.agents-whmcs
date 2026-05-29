# WHMCS Coupon Tracking Workflow

## Purpose
 track and analyze coupon code usage.

## Tracking Setup

### Enable Tracking
1. Navigate to: Configuration > Promotions
2. Enable "Track Usage"
3. Set data retention

### What to Track
```
Track:
- Total uses
- Uses by client
- Uses by date
- Order value
- Discount amount
```

## Viewing Usage

### Coupon Reports
```
Access:
- Reports > Coupon Usage
- Select date range
- Filter by coupon
```

### Information Displayed
```
Shows:
- Coupon code
- Total uses
- Unique clients
- Revenue generated
- Discount given
```

## Usage Analysis

### By Coupon
```
Example:
- Code: SUMMER2024
- Uses: 150
- Revenue: $7,500
- Avg discount: 20%
```

### By Client
```
Example:
- Client: John Doe
- Uses: 2
- Total saved: $100
- Orders: 2
```

### By Time
```
Metrics:
- Uses per day
- Peak usage times
- Popular days
```

## Client Behavior

### First-Time Use
```
Track:
- New customer usage
- Conversion from coupon
- Repeat purchases after
```

### Repeat Use
```
Track:
- Same coupon multiple times
- Different coupons
- Loyalty patterns
```

## Financial Tracking

### Revenue Impact
```
Track:
- Revenue with coupon
- Revenue without coupon
- Revenue per discount dollar
- Customer lifetime value
```

### ROI Calculation
```
Formula:
ROI = (Revenue - Discount Cost) / Discount Cost

Example:
- Revenue: $10,000
- Discount given: $2,000
- Net: $8,000
- Cost: $2,000
- ROI: 300%
```

## Exporting Data

### Export Options
```
Formats:
- CSV
- Excel
- PDF report
```

### Export Content
```
Include:
- All usage data
- Client details
- Order information
- Time stamps
```

## Analytics Reports

### Performance Dashboard
```
Displays:
- Top performing coupons
- Worst performing coupons
- Usage trends
- Revenue by coupon
```

### Custom Reports
```
Create:
- Select metrics
- Set date range
- Filter by segment
- Schedule reports
```

## Attribution

### Track Source
```
Monitor:
- Where code was distributed
- Which channel performed best
- Customer acquisition source
```

### UTM Tracking
```
Add to coupon links:
example.com?coupon=SUMMER&source=email
```

## Best Practices

### Guidelines
```
- Track everything
- Regular analysis
- Compare performance
- Optimize based on data
```

## Related Workflows
- whmcs-coupon-create
- whmcs-coupon-limit
- whmcs-promotion-analytics
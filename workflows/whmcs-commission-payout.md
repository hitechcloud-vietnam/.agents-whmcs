# WHMCS Commission Payout Workflow

## Purpose
Process affiliate commission payouts

## Prerequisites
- WHMCS installed
- Affiliate module enabled
- Admin access

## Step 1: Review Affiliate Program Settings

Navigate to: Setup > Affiliates > Affiliate Settings

Verify:
- Commission rate
- Payout threshold
- Payout methods
- Commission rules

## Step 2: Generate Commission Report

Navigate to: Reports > Affiliates

Review:
- Pending commissions
- Approved commissions
- Paid commissions
- Affiliate performance

## Step 3: Calculate Commissions

### Pending Commissions
```sql
SELECT 
    a.id,
    a.name,
    a.email,
    SUM(aff.commission) as pending_commission,
    COUNT(aff.id) as referrals
FROM tblaffiliates a
JOIN tblaffiliatesaccounts aff ON a.id = aff.affiliateid
JOIN tblorders o ON aff.orderid = o.id
WHERE o.status = 'Active'
GROUP BY a.id, a.name, a.email
HAVING SUM(aff.commission) >= 50;
```

## Step 4: Review Commission Eligibility

Verify:
- Orders completed
- No refunds
- Payment cleared
- Commission not revoked

## Step 5: Approve Commissions

Navigate to: Configuration > System Settings > Affiliates > Pending Commissions

1. Review each pending commission
2. Verify eligibility
3. Approve or reject

## Step 6: Process Payouts

### PayPal Payouts
1. Export affiliate emails
2. Create PayPal mass payment
3. Process payments

### Bank Transfers
1. Get affiliate bank details
2. Process individual transfers
3. Document transactions

### Account Credit
Navigate to: Clients > [Affiliate] > Credits

1. Add credit to account
2. Notify affiliate

## Step 7: Record Payouts

Navigate to: Configuration > System Settings > Affiliates > Payout History

Record:
- Affiliate ID
- Amount paid
- Payment method
- Transaction ID
- Date

## Step 8: Send Payout Notifications

Navigate to: Configuration > System Settings > Affiliates > Notifications

Send email to each affiliate:
```
Subject: Commission Payout Processed

Dear [Name],

Your affiliate commission of $[amount] has been processed.

Method: [method]
Reference: [reference]

Thank you for your partnership!

Best regards,
[Company]
```

## Step 9: Generate Payout Report

Create report:
```
Period: [dates]
Total Affiliates Paid: XX
Total Payout Amount: $XXX
Average Payout: $XX
Largest Payout: $XX
```

## Commission Payout Checklist

- [ ] Affiliate settings reviewed
- [ ] Commission report generated
- [ ] Commissions calculated
- [ ] Eligibility verified
- [ ] Commissions approved
- [ ] Payouts processed
- [ ] Payouts recorded
- [ ] Notifications sent
- [ ] Report generated

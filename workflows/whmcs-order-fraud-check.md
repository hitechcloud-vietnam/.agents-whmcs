# WHMCS Order Fraud Check Workflow

## Purpose
Step-by-step guide for performing fraud checks on orders in WHMCS.

## Prerequisites
- Order received
- Fraud detection configured
- Fraud rules defined
- Admin access

## Workflow Steps

### Step 1: Fraud Check Trigger
- Order submitted
- Automatic fraud check
- Risk scoring activated
- Document check initiated

### Step 2: Order Data Collection
- Gather order details
- Collect customer info
- Extract payment data
- Review product selection

### Step 3: Fraud Rule Evaluation
Run fraud checks:
- Address verification
- Card verification (CVV, AVS)
- Email domain analysis
- IP geolocation
- Device fingerprinting
- Velocity checks

### Step 4: Risk Scoring
- Calculate risk score
- Assess risk factors
- Identify red flags
- Determine risk level

### Step 5: Blacklist Check
- Check against blacklist
- Verify not blocked
- Check fraud database
- Review history

### Step 6: Velocity Analysis
- Check order frequency
- Verify unusual patterns
- Assess rapid changes
- Evaluate spike alerts

### Step 7: Geolocation Check
- Verify IP location
- Compare billing/shipping
- Check shipping distance
- Validate location anomalies

### Step 8: Risk Decision
- Pass: auto-approve
- Review: flag for manual
- Block: reject order
- Hold: require verification

### Step 9: Action Execution
- Approve order (if passed)
- Flag for review (if uncertain)
- Reject order (if failed)
- Request verification

### Step 10: Documentation
- Log fraud check results
- Record risk score
- Document decision
- Update fraud database

## Fraud Indicators
- Mismatched address
- High-risk email domain
- Multiple failed payments
- Unusual order pattern
- IP mismatch
- Suspicious behavior

## Verification Checklist
- [ ] Checks executed
- [ ] Risk assessed
- [ ] Decision made
- [ ] Action taken
- [ ] Documented

## Related Workflows
- whmcs-order-review
- whmcs-order-approval
- whmcs-order-rejection
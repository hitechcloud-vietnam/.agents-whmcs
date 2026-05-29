# WHMCS Service Cancellation Workflow

## Purpose
Step-by-step guide for processing service cancellations in WHMCS.

## Prerequisites
- Active service to cancel
- Cancellation request received
- Client verification
- Final billing processed

## Workflow Steps

### Step 1: Cancellation Request
- Receive cancellation request
- Verify client identity
- Document cancellation reason
- Check contract/terms

### Step 2: Eligibility Check
- Verify service exists
- Check for pending payments
- Review cancellation policy
- Determine refund eligibility

### Step 3: Review Final Invoice
- Calculate final charges
- Generate final invoice if needed
- Apply any credits
- Determine refund amount

### Step 4: Server Preparation
- Schedule service termination
- Backup client data (if required)
- Prepare final data export
- Document data retention policy

### Step 5: Service Status Change
- Set service to Cancelled
- Record cancellation date
- Update billing status
- Disable access

### Step 6: Server Termination
- Remove account from server
- Archive data (per policy)
- Release resources
- Update DNS if applicable

### Step 7: Refund Processing
- Calculate refund amount
- Process refund to original payment
- Or apply to account credit
- Document refund reason

### Step 8: Confirmation Email
- Send cancellation confirmation
- Include final invoice
- Detail refund information
- Provide data access timeline

### Step 9: Data Handling
- Retain data per policy
- Schedule data deletion
- Provide export if requested
- Document retention log

### Step 10: Internal Documentation
- Log cancellation details
- Record reason code
- Document refund/expiration
- Update statistics

## Cancellation Reasons
- Customer requested
- Payment failure
- Abuse/TOS violation
- Non-renewal
- Migration elsewhere
- Dissatisfied with service

## Data Retention Policy
- Configuration: 30 days
- Billing records: 7 years
- Personal data: per GDPR

## Verification Checklist
- [ ] Cancellation authorized
- [ ] Final invoice generated
- [ ] Payment/refund processed
- [ ] Server account terminated
- [ ] Data backed up
- [ ] Confirmation sent
- [ ] Documentation complete

## Related Workflows
- whmcs-service-reactivation
- whmcs-order-cancellation
- whmcs-client-closure
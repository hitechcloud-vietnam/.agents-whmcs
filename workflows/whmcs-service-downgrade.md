# WHMCS Service Downgrade Workflow

## Purpose
Step-by-step guide for processing service downgrades in WHMCS.

## Prerequisites
- Existing active service
- Downgrade path defined in product
- Current cycle completion required
- Credit policy established

## Workflow Steps

### Step 1: Downgrade Request Received
- Client submits downgrade request
- Review current service details
- Identify target downgrade product
- Explain effective date (end of cycle)

### Step 2: Policy Review
- Check downgrade policy settings
- Determine if credit applies
- Verify billing cycle timing
- Document terms to client

### Step 3: Request Validation
- Confirm client owns service
- Verify service is active
- Check for outstanding invoices
- Review contract terms

### Step 4: Downgrade Scheduling
- Set downgrade for next billing date
- Record pending downgrade
- Add internal note
- Create follow-up task

### Step 5: Current Cycle Completion
- Allow current cycle to complete
- Maintain current service level
- No immediate changes
- Continue billing at current rate

### Step 6: Downgrade Execution (Next Cycle)
- Generate prorate credit if applicable
- Update product assignment
- Reduce resource allocation
- Remove premium features

### Step 7: Server Update
- Connect to server
- Reduce resource limits
- Disable removed features
- Update account configuration

### Step 8: Client Notification
- Send downgrade confirmation
- Detail new service level
- Explain billing changes
- List removed features

### Step 9: Documentation
- Log downgrade request
- Record effective date
- Document changes made
- Update client notes

### Step 10: Verification
- Confirm service level reduced
- Verify billing updated
- Ensure removed features disabled
- Check credit applied if any

## Credit Policy Options
- No credit (immediate downgrade)
- Credit to account balance
- Credit on next invoice
- Prorate refund (rare)

## Verification Checklist
- [ ] Downgrade scheduled correctly
- [ ] Client informed of timeline
- [ ] Service downgraded at cycle end
- [ ] Server updated
- [ ] Billing adjusted
- [ ] Client notified
- [ ] Documentation complete

## Related Workflows
- whmcs-service-upgrade
- whmcs-service-cancellation
- whmcs-service-change-cycle
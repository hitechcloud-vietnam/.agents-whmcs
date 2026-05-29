# WHMCS Service Renewal Workflow

## Purpose
Step-by-step guide for processing service renewals in WHMCS.

## Prerequisites
- Active service with upcoming due date
- Payment method on file
- Auto renewal enabled (optional)

## Workflow Steps

### Step 1: Renewal Trigger
- Automatic: scheduled task runs
- Manual: client initiates renewal
- Admin: staff processes renewal
- Determine renewal type

### Step 2: Due Date Check
- Calculate days until due
- Verify service status
- Check for pending changes
- Review billing history

### Step 3: Invoice Generation
- Generate renewal invoice
- Calculate pricing (current rate)
- Apply any renewal discounts
- Include addons if applicable

### Step 4: Payment Processing
- Attempt charge automatically
- If failed, send notification
- Retry failed payments
- Process manual payments

### Step 5: Payment Confirmation
- Mark invoice as paid
- Record payment transaction
- Update account balance
- Log payment details

### Step 6: Term Extension
- Calculate new due date
- Extend service term
- Update next due date
- Set billing cycle

### Step 7: Server Update
- Verify account active
- Extend service period on server
- Update expiration settings
- Confirm resource allocation

### Step 8: Feature Verification
- Confirm all features active
- Verify monitoring enabled
- Check resource limits
- Validate service status

### Step 9: Client Notification
- Send renewal confirmation
- Include new expiration date
- Detail next billing date
- Provide receipt

### Step 10: Documentation
- Log renewal transaction
- Record payment details
- Update service history
- Document any changes

## Renewal Reminder Schedule
- 14 days before: first reminder
- 7 days before: second reminder
- 1 day before: final reminder
- Due date: suspend warning

## Grace Period Handling
- Default grace period: 7 days
- Suspend after grace period
- Cancellation after extended period

## Verification Checklist
- [ ] Invoice generated
- [ ] Payment processed
- [ ] Term extended
- [ ] Server updated
- [ ] Confirmation sent
- [ ] Documentation complete

## Related Workflows
- whmcs-new-service-provisioning
- whmcs-service-suspension
- whmcs-service-cancellation
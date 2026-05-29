# WHMCS Order Status Update Workflow

## Purpose
Step-by-step guide for updating order statuses in WHMCS.

## Prerequisites
- Order exists
- Status change reason
- Authorization verified
- Update method identified

## Workflow Steps

### Step 1: Status Update Request
- Identify order
- Determine target status
- Document reason
- Verify authorization

### Step 2: Status Review
Current status options:
- Pending
- Pending Review
- Active
- Cancelled
- Fraud
- Suspended

Target status options:
- Active
- Cancelled
- Suspended
- Pending

### Step 3: Eligibility Check
- Verify status change allowed
- Check dependencies
- Review prerequisites
- Assess impact

### Step 4: Related Updates
- Identify services affected
- Plan service status changes
- Check billing impact
- Prepare invoice updates

### Step 5: Approval Process
- Route for approval
- Verify authorization
- Document approval
- Set conditions

### Step 6: Status Update Execution
- Change order status
- Update service status
- Modify billing
- Update records

### Step 7: Automation Triggers
- Execute status-based hooks
- Send notifications
- Trigger workflows
- Update integrations

### Step 8: Client Notification
- Send status update
- Explain new status
- Provide next steps
- Include instructions

### Step 9: Confirmation
- Verify update complete
- Check all records
- Confirm services updated
- Validate changes

### Step 10: Documentation
- Log status change
- Record reason
- Document authorization
- Update audit trail

## Status Change Rules
- Pending -> Active: payment required
- Active -> Cancelled: cancellation policy
- Active -> Suspended: payment issue
- Any -> Fraud: fraud detection

## Verification Checklist
- [ ] Update authorized
- [ ] Status changed
- [ ] Services updated
- [ ] Notifications sent
- [ ] Documented

## Related Workflows
- whmcs-order-tracking
- whmcs-order-cancellation
- whmcs-service-suspension
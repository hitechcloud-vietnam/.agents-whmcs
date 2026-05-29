# WHMCS Service Transfer Workflow

## Purpose
Step-by-step guide for transferring services between clients in WHMCS.

## Prerequisites
- Active service to transfer
- Both clients verified
- Transfer authorization from current owner
- Payment for transfer fee (if any)

## Workflow Steps

### Step 1: Transfer Request Received
- Current owner requests transfer
- Identify services to transfer
- Document transfer reason
- Gather recipient information

### Step 2: Recipient Verification
- Verify recipient client exists
- If not, create new client account
- Confirm recipient identity
- Collect required documentation

### Step 3: Authorization Processing
- Collect transfer authorization
- Get signature or approval
- Document terms acceptance
- Record authorization timestamp

### Step 4: Financial Review
- Check outstanding balance on service
- Calculate transfer fee
- Verify payment method
- Clear any pending invoices

### Step 5: Service Review
- Review service details
- Check contract terms for transfer
- Verify no transfer restrictions
- Document service configuration

### Step 6: Transfer Approval
- Approve transfer in admin
- Update client association
- Transfer service ownership
- Record transfer date

### Step 7: Server Update
- Change account ownership on server
- Update billing information
- Reset access credentials
- Configure new owner access

### Step 8: Data Handling
- Review data transfer requirements
- Handle personal information
- Update contact details
- Preserve service data

### Step 9: Notification to Both Parties
- Notify current owner: transfer complete
- Notify recipient: service transferred
- Include new access details
- Provide support information

### Step 10: Documentation
- Log full transfer details
- Record authorization used
- Document financial transactions
- Update both client records

## Transfer Authorization Methods
- Email approval
- Ticket confirmation
- Form signature
- Phone verification

## Transfer Restrictions
- Contract lock-in periods
- Pending cancellations
- Active disputes
- Fraud flags

## Verification Checklist
- [ ] Authorization received
- [ ] Recipient verified
- [ ] Payments cleared
- [ ] Ownership transferred
- [ ] Server updated
- [ ] Both parties notified
- [ ] Documentation complete

## Related Workflows
- whmcs-service-ownership
- whmcs-service-transfer-approval
- whmcs-client-merging
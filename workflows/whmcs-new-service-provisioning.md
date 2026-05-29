# WHMCS New Service Provisioning Workflow

## Purpose
Complete step-by-step guide for provisioning new services in WHMCS from order to active status.

## Prerequisites
- WHMCS admin access
- Product/Service configured
- Payment verified
- Server module configured

## Workflow Steps

### Step 1: Order Received
- New order appears in WHMCS admin
- Check order status: Pending
- Verify client information complete
- Check for fraud indicators

### Step 2: Payment Verification
- Confirm payment status (Paid/Pending)
- If pending, wait for payment confirmation
- Apply any applicable promo codes
- Verify billing amount correct

### Step 3: Fraud Check
- Run automated fraud checks
- Review fraud score
- If flagged, manually review order
- Document fraud decision

### Step 4: Order Approval
- Approve order in admin area
- Set order status to Active
- Trigger provisioning automation

### Step 5: Product Configuration
- Load product configuration
- Set custom values if applicable
- Configure resource limits
- Set service expiration date

### Step 6: Server Provisioning
- Connect to server module
- Create account on server
- Configure service settings
- Set initial passwords
- Configure monitoring

### Step 7: Service Activation
- Update service status to Active
- Set next due date
- Configure billing cycle
- Enable service features

### Step 8: Welcome Communication
- Send welcome email template
- Include login credentials
- Provide setup instructions
- Share support contact info

### Step 9: Documentation
- Log provisioning details
- Record server credentials used
- Document configuration choices
- Update client notes

### Step 10: Verification
- Test service accessibility
- Verify resource allocation
- Confirm monitoring active
- Ensure billing configured

## Error Handling
- If provisioning fails: retry up to 3 times, then escalate to NOC
- If payment fails: notify client, pause provisioning
- If server unavailable: mark order pending, alert sysadmin

## Verification Checklist
- [ ] Order approved
- [ ] Payment confirmed
- [ ] Service active on server
- [ ] Welcome email sent
- [ ] Client notified
- [ ] Documentation complete

## Related Workflows
- whmcs-service-renewal
- whmcs-service-cancellation
- whmcs-order-fulfillment
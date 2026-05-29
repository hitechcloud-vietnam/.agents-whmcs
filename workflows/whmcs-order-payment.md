# WHMCS Order Payment Workflow

## Purpose
Step-by-step guide for processing order payments in WHMCS.

## Prerequisites
- Order created
- Payment method selected
- Payment gateway configured
- Invoice generated

## Workflow Steps

### Step 1: Payment Initiation
- Identify pending payment
- Verify order details
- Check payment amount
- Prepare payment request

### Step 2: Payment Gateway Selection
- Select appropriate gateway
- Configure payment parameters
- Set security options
- Prepare payment form

### Step 3: Customer Payment Entry
- Customer enters payment details
- Validate card/account info
- Apply promo codes
- Verify billing address

### Step 4: Payment Processing
- Submit to payment gateway
- Handle 3D Secure
- Process authorization
- Handle declines

### Step 5: Transaction Verification
- Verify transaction response
- Check authorization code
- Validate amount
- Confirm transaction ID

### Step 6: Payment Recording
- Record payment in WHMCS
- Update invoice status
- Log transaction details
- Update order status

### Step 7: Order Status Update
- Change order to Paid
- Trigger provisioning
- Update client balance
- Clear pending flags

### Step 8: Receipt Generation
- Generate payment receipt
- Send receipt to customer
- Update accounting records
- Archive transaction

### Step 9: Error Handling
- Handle payment failures
- Process retries
- Manage declined cards
- Notify customer

### Step 10: Reconciliation
- Match payments to orders
- Update financial records
- Reconcile with gateway
- Handle disputes

## Payment Methods
- Credit/Debit cards
- PayPal
- Bank transfer
- Crypto currency
- Alternative payments

## Verification Checklist
- [ ] Payment processed
- [ ] Transaction verified
- [ ] Order updated
- [ ] Receipt sent
- [ ] Reconciled

## Related Workflows
- whmcs-new-order-flow
- whmcs-order-fulfillment
- whmcs-order-refund
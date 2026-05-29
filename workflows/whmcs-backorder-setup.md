# WHMCS Backorder Setup Workflow

## Purpose
Configure and manage backorder functionality for products that are temporarily out of stock but available for future delivery.

## Prerequisites
- WHMCS installation
- Stock management enabled
- Products that can be backordered

## Step-by-Step Process

### Step 1: Access Backorder Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services`
3. Select product to configure backorder

### Step 2: Enable Backorder Functionality
1. Configure per product:
   - Enable backorders
   - Set backorder limit
   - Configure backorder behavior
2. Set global backorder settings

### Step 3: Configure Backorder Availability
1. Set availability rules:
   - Available when out of stock
   - Available with quantity limit
   - Available with specific products only
   - Date-based availability
2. Set maximum backorder quantity

### Step 4: Configure Backorder Pricing
1. Set pricing for backorders:
   - Same price as regular
   - Premium for backorder
   - Deposit required
   - Full payment upfront
2. Configure deposit amount

### Step 5: Set Up Backorder Display
1. Configure customer-facing display:
   - Show "Available on Backorder"
   - Display estimated availability
   - Show backorder queue position
   - Display "Join Waitlist" option
2. Set expected availability dates

### Step 6: Configure Backorder Processing
1. Set processing rules:
   - Process in order received
   - Process in batches
   - Process when stock arrives
2. Configure order management:
   - Hold backorder orders
   - Partially fulfill orders
   - Auto-cancel after X days

### Step 7: Set Up Backorder Notifications
1. Configure alerts:
   - Customer notification on backorder
   - Estimated delivery update
   - Stock arrival notification
   - Order processing notification
2. Set notification triggers

### Step 8: Configure Stock Arrival Processing
1. Set fulfillment rules:
   - Auto-process when stock arrives
   - Manual processing required
   - Prioritize by order date
   - Prioritize by deposit paid
2. Configure partial fulfillment

### Step 9: Set Up Waitlist Management
1. Configure waitlist:
   - Enable waitlist for backorders
   - Show position in queue
   - Email when available
   - Allow remove from waitlist
2. Set waitlist notification rules

### Step 10: Manage Backorder Reporting
1. Set up reporting:
   - Backorder volume tracking
   - Fulfillment time tracking
   - Cancellation rate
   - Waitlist conversion rate
2. Generate backorder reports

## Verification Checklist
- [ ] Backorder option displays correctly
- [ ] Orders place successfully
- [ ] Customer notifications send
- [ ] Stock arrival triggers processing
- [ ] Waitlist functions work

## Related Workflows
- whmcs-stock-management
- whmcs-low-stock-alert
- whmcs-preorder-setup
- whmcs-checkout-flow

## Backorder Best Practices
- Set clear delivery expectations
- Communicate delays proactively
- Process orders in fair sequence
- Track and optimize fulfillment time
- Consider deposit to reduce cancellations
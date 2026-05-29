# WHMCS Stock Management Workflow

## Purpose
Implement stock/inventory management to track product quantities, control availability, and manage product visibility based on stock levels.

## Prerequisites
- WHMCS installation
- Physical or limited products
- Stock tracking requirements identified

## Step-by-Step Process

### Step 1: Access Stock Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services`
3. Select product to configure stock

### Step 2: Enable Stock Control
1. Turn on stock management per product:
   - Enable stock tracking
   - Set initial stock quantity
   - Set low stock threshold
2. Configure stock deduction triggers

### Step 3: Set Stock Levels
1. Configure stock quantities:
   - Set current stock level
   - Set minimum stock level
   - Set reorder point
   - Set maximum stock level
2. Configure stock status display

### Step 4: Configure Stock Behavior
1. Set stock action rules:
   - Hide out-of-stock products
   - Show "Out of Stock" badge
   - Allow backorders
   - Show "Pre-order" option
   - Display stock quantity
2. Set automatic status changes

### Step 5: Configure Stock Notifications
1. Set up alerts:
   - Low stock notification
   - Out of stock notification
   - Reorder notification
   - Stock replenished notification
2. Set notification recipients

### Step 6: Set Up Automatic Updates
1. Configure automation:
   - Auto-deduct on order
   - Auto-restore on cancellation
   - Auto-restore on refund
   - Sync with external inventory
2. Set sync frequency

### Step 7: Configure Stock Display
1. Set visibility rules:
   - Show stock count
   - Show "Limited Availability"
   - Show "Selling Fast" badge
   - Display restock date
2. Set threshold triggers for messages

### Step 8: Set Up Bulk Stock Management
1. Configure bulk operations:
   - Bulk stock update
   - CSV import/export
   - Scheduled stock updates
2. Set up stock reporting

### Step 9: Configure Stock History
1. Set up tracking:
   - Track stock changes
   - Record stock adjustments
   - Track stock take results
   - Record manual adjustments
2. Set adjustment reasons

### Step 10: Integrate with Suppliers
1. Configure supplier integration:
   - Auto-reorder triggers
   - Supplier notification
   - Lead time tracking
   - Backorder management
2. Set up external inventory sync

## Verification Checklist
- [ ] Stock tracks correctly on orders
- [ ] Low stock alerts trigger
- [ ] Out of stock products behave correctly
- [ ] Stock restores on cancellation
- [ ] Reports accurate

## Related Workflows
- whmcs-low-stock-alert
- whmcs-backorder-setup
- whmcs-preorder-setup
- whmcs-product-setup

## Stock Management Best Practices
- Keep accurate stock counts
- Set appropriate low-stock thresholds
- Automate restocking processes
- Monitor stock movement regularly
- Sync with physical inventory
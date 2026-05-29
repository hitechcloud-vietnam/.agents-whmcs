# WHMCS Recently Viewed Products Workflow

## Purpose
Implement a recently viewed products feature to help customers return to products they've shown interest in.

## Prerequisites
- WHMCS installation
- Products configured
- Analytics/tracking desired

## Step-by-Step Process

### Step 1: Access Recently Viewed Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Order Form`
3. Locate recently viewed/recently added settings

### Step 2: Enable Recently Viewed Feature
1. Turn on tracking:
   - Enable view tracking
   - Set tracking scope (all products, specific categories)
   - Configure storage duration
2. Set maximum products to track

### Step 3: Configure Display Location
1. Set display positions:
   - Homepage section
   - Category pages
   - Product detail pages
   - Shopping cart page
   - Checkout confirmation page
2. Set visibility rules

### Step 4: Design Display Format
1. Configure display style:
   - Carousel/slider view
   - Grid view
   - List view
   - Number of products shown
   - Thumbnail size
2. Set responsive behavior

### Step 5: Configure User Experience
1. Set interaction options:
   - Quick add to cart
   - Quick view details
   - Remove from list
   - Clear all history
   - Clear individual items
2. Set hover/click behaviors

### Step 6: Set Up Privacy Controls
1. Configure privacy options:
   - Show for logged-in users only
   - Allow guest tracking
   - Clear history option
   - Opt-out capability
2. Set cookie/consent handling

### Step 7: Configure Cross-Sell Integration
1. Set up recommendations:
   - "Customers who viewed this also viewed"
   - Related product suggestions
   - "You may also like" section
2. Link recently viewed to recommendations

### Step 8: Implement Price Tracking
1. Track price changes:
   - Show original price vs. current
   - Highlight price drops
   - Show "was" pricing
2. Set price change notifications

### Step 9: Set Up Analytics
1. Configure tracking:
   - View count tracking
   - Click tracking from recently viewed
   - Conversion tracking
   - Product popularity metrics
2. Generate reports

### Step 10: Optimize Display
1. Review performance:
   - Click-through rate
   - Conversion from recently viewed
   - User engagement
2. Test different positions
3. Adjust display based on data

## Verification Checklist
- [ ] Products tracked when viewed
- [ ] Recently viewed section displays
- [ ] Mobile experience works
- [ ] Analytics tracking active
- [ ] Privacy settings function

## Related Workflows
- whmcs-wishlist
- whmcs-product-reviews
- whmcs-cart-customization
- whmcs-upsell-checkout

## Best Practices
- Show most recently viewed first
- Include product image and price
- Make add to cart easy
- Set reasonable retention period
- Test mobile experience
# WHMCS Order Form Configuration Workflow

## Purpose
Customize and optimize WHMCS order forms

## Prerequisites
- WHMCS installed
- Admin access
- Basic design knowledge

## Step 1: Navigate to Order Form Settings

Navigate to: Setup > Payments > Order Form Settings

## Step 2: Select Order Form Template

Choose from available templates:
- Six (modern, responsive)
- Standard / Legacy
- Cart
- Minimal
- Custom

```
Current Template: Six
```

## Step 3: Configure General Settings

Navigate to: Setup > Payments > Order Form Settings > General

```
Allow Quantity Selection: No
Show Product Reviews: Yes
Enable AutoSetup: Yes
Default Checkout Country: United States
Auto-Create Account: Yes
```

## Step 4: Set Up Product Groups

Navigate to: Setup > Products/Services > Products/Services

1. Edit product group
2. Configure order form:
   ```
   Group Headline: [custom text]
   Group Description: [custom text]
   Featured Product: [select]
   ```

## Step 5: Configure Product Display

Navigate to: Setup > Payments > Order Form Settings > Product Display

```
Show Product Description: Yes
Show Feature Highlights: Yes
Show Billing Cycle Toggle: Yes
Show Quantity Field: No
Show Add to Cart Button: Yes
```

## Step 6: Set Up Domain Selection

Navigate to: Setup > Payments > Order Form Settings > Domain Selection

```
Show Domain Selection: Yes
Default to Free Domain: No
Domain Text: "Register a new domain"
Subdomain Text: "Use a subdomain"
Transfer Text: "Transfer your domain"
```

## Step 7: Configure Cart Settings

Navigate to: Setup > Payments > Order Form Settings > Cart

```
Show Cart Sidebar: Yes
Cart Summary Position: Right
Allow Item Removal: Yes
Allow Quantity Edit: No
Show Continue Shopping: Yes
```

## Step 8: Set Up Checkout Options

Navigate to: Setup > Payments > Order Form Settings > Checkout

```
Show Order Summary: Yes
Auto-Generate Password: Yes
Require Domain for Hosting: No
Hide Pricing Until Domain: No
```

## Step 9: Configure Order Status Colors

Navigate to: Setup > Payments > Order Form Settings > Status Colors

```
Pending: Orange
Active: Green
Suspended: Red
Cancelled: Gray
Terminated: Black
```

## Step 10: Set Up Promotional Codes

Navigate to: Setup > Payments > Order Form Settings > Promotions

```
Allow Promo Codes: Yes
Show Promo Field: Yes
Auto-Apply Promotions: No
```

## Step 11: Customize Order Form CSS

Navigate to: Setup > Client Area Design > Customize Theme

Add custom CSS:
```css
.order-form {
    font-family: 'Inter', sans-serif;
}

.product-box {
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.btn-order {
    background-color: #0073aa;
    border-radius: 4px;
}
```

## Step 12: Configure Product Recommendations

Navigate to: Setup > Payments > Order Form Settings > Recommendations

```
Show Upsells: Yes
Show Cross-sells: Yes
Show Frequently Bought Together: Yes
Recommendation Count: 3
```

## Order Form Checklist

- [ ] Template selected
- [ ] General settings configured
- [ ] Product groups set up
- [ ] Product display customized
- [ ] Domain selection configured
- [ ] Cart settings set
- [ ] Checkout options configured
- [ ] Promotional codes enabled
- [ ] CSS customized
- [ ] Recommendations configured

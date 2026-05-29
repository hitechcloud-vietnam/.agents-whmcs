# WHMCS Product Reviews Workflow

## Purpose
Implement a product review and review management system to build trust and gather customer feedback.

## Prerequisites
- WHMCS installation
- Products configured
- Customer orders for reviews

## Step-by-Step Process

### Step 1: Access Review Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Customization > Product Reviews`
3. Review available settings

### Step 2: Enable Review System
1. Turn on product reviews:
   - Enable for all products
   - Enable for specific categories
   - Set minimum purchase requirements
2. Configure review permissions

### Step 3: Configure Review Requirements
1. Set submission requirements:
   - Require verified purchase
   - Require account login
   - Allow guest reviews
   - Set minimum order completion
2. Configure review limits per customer

### Step 4: Set Up Review Form
1. Configure review fields:
   - Title/summary
   - Review text/description
   - Rating (star rating)
   - Pros/cons sections
   - Images/attachments
   - Nickname/display name
2. Set field requirements and validation

### Step 5: Configure Review Moderation
1. Set moderation workflow:
   - Auto-approve all reviews
   - Manual approval required
   - Auto-approve with flagging
   - Spam filter integration
2. Configure notification on new reviews

### Step 6: Set Review Display Options
1. Configure display settings:
   - Sort order (newest, highest rated, most helpful)
   - Pagination style
   - Show helpful votes
   - Display date
   - Show verified purchase badge
2. Set review summary display

### Step 7: Enable Review Voting
1. Set up helpful votes:
   - Enable "Was this helpful" voting
   - Sort by helpful votes
   - Show vote count
2. Configure voting rules

### Step 8: Configure Review Responses
1. Set up owner responses:
   - Enable merchant replies
   - Set response guidelines
   - Configure response email notifications
2. Set display of responses

### Step 9: Set Up Review Notifications
1. Configure email notifications:
   - New review notification to admin
   - Review response notification
   - Helpful vote notifications
2. Set up follow-up emails:
   - Post-purchase review request
   - Review reminder after X days

### Step 10: Manage and Moderate Reviews
1. Review management:
   - Approve/reject pending reviews
   - Edit reviews
   - Delete reviews
   - Mark as featured
   - Respond to reviews
2. Generate review reports

## Verification Checklist
- [ ] Review form displays on products
- [ ] Reviews save and moderate correctly
- [ ] Ratings calculate properly
- [ ] Display shows reviews correctly
- [ ] Email notifications send

## Related Workflows
- whmcs-product-ratings
- whmcs-product-reviews
- whmcs-product-setup
- whmcs-checkout-flow

## Review Best Practices
- Respond to all reviews
- Encourage detailed reviews
- Highlight top reviews
- Show verified purchase badges
- Make helpful votes visible
# WHMCS Card Design Workflow

## Purpose
Guide developers through creating and customizing card components in WHMCS.

## Prerequisites
- WHMCS installation
- CSS/SCSS knowledge
- Grid/flexbox understanding
- Bootstrap card components

## Steps

### Phase 1: Card Structure

1. Bootstrap card structure
   ```html
   <div class="card">
       <div class="card-header">Header</div>
       <div class="card-body">
           <h5 class="card-title">Card Title</h5>
           <p class="card-text">Card content</p>
           <a href="#" class="btn btn-primary">Action</a>
       </div>
       <div class="card-footer">Footer</div>
   </div>
   ```

2. Card template locations
   ```
   Cards in WHMCS:
   - Product cards (cart)
   - Service cards (client area)
   - Dashboard cards
   - Pricing cards
   - Widget cards
   ```

### Phase 2: Basic Card Styling

1. Base card styles
   ```css
   .card {
       background: #fff;
       border: 1px solid #e9ecef;
       border-radius: 10px;
       box-shadow: 0 2px 10px rgba(0,0,0,0.05);
       transition: transform 0.2s, box-shadow 0.2s;
   }
   
   .card:hover {
       transform: translateY(-3px);
       box-shadow: 0 8px 25px rgba(0,0,0,0.1);
   }
   
   .card-header {
       padding: 16px 20px;
       background: #fff;
       border-bottom: 1px solid #e9ecef;
       font-weight: 600;
   }
   
   .card-body {
       padding: 20px;
   }
   
   .card-footer {
       padding: 16px 20px;
       background: #f8f9fa;
       border-top: 1px solid #e9ecef;
   }
   ```

### Phase 3: Product Cards

1. Product card design
   ```css
   .product-card {
       background: #fff;
       border-radius: 12px;
       overflow: hidden;
       box-shadow: 0 4px 15px rgba(0,0,0,0.08);
   }
   
   .product-card .card-image {
       height: 180px;
       background: linear-gradient(135deg, #667eea, #764ba2);
       display: flex;
       align-items: center;
       justify-content: center;
   }
   
   .product-card .card-image img {
       max-height: 100%;
       width: auto;
   }
   
   .product-card .card-body {
       padding: 25px;
   }
   
   .product-card .product-name {
       font-size: 18px;
       font-weight: 600;
       margin-bottom: 10px;
   }
   
   .product-card .product-description {
       color: #6c757d;
       font-size: 14px;
       margin-bottom: 20px;
   }
   
   .product-card .product-price {
       display: flex;
       align-items: baseline;
       margin-bottom: 15px;
   }
   
   .product-card .price-amount {
       font-size: 28px;
       font-weight: 700;
       color: #212529;
   }
   
   .product-card .price-period {
       font-size: 14px;
       color: #6c757d;
       margin-left: 5px;
   }
   ```

### Phase 4: Pricing Cards

1. Pricing card styles
   ```css
   .pricing-card {
       background: #fff;
       border-radius: 16px;
       padding: 30px;
       text-align: center;
       border: 2px solid #e9ecef;
       transition: all 0.3s;
   }
   
   .pricing-card:hover {
       border-color: #667eea;
       box-shadow: 0 10px 30px rgba(102, 126, 234, 0.15);
   }
   
   .pricing-card.featured {
       border-color: #667eea;
       background: linear-gradient(135deg, rgba(102,126,234,0.05), rgba(118,75,162,0.05));
       transform: scale(1.05);
   }
   
   .pricing-card .plan-name {
       font-size: 20px;
       font-weight: 600;
       margin-bottom: 15px;
       color: #212529;
   }
   
   .pricing-card .plan-price {
       font-size: 48px;
       font-weight: 700;
       color: #667eea;
       line-height: 1;
   }
   
   .pricing-card .plan-price span {
       font-size: 16px;
       font-weight: 400;
       color: #6c757d;
   }
   
   .pricing-card .plan-features {
       list-style: none;
       padding: 0;
       margin: 25px 0;
       text-align: left;
   }
   
   .pricing-card .plan-features li {
       padding: 8px 0;
       border-bottom: 1px solid #f0f0f0;
   }
   
   .pricing-card .plan-features li:last-child {
       border-bottom: none;
   }
   ```

### Phase 5: Service Cards

1. Service card design
   ```css
   .service-card {
       background: #fff;
       border-radius: 10px;
       padding: 20px;
       border-left: 4px solid #667eea;
       box-shadow: 0 2px 8px rgba(0,0,0,0.05);
   }
   
   .service-card .service-header {
       display: flex;
       justify-content: space-between;
       align-items: flex-start;
       margin-bottom: 15px;
   }
   
   .service-card .service-name {
       font-weight: 600;
       font-size: 16px;
       color: #212529;
   }
   
   .service-card .service-domain {
       font-size: 13px;
       color: #6c757d;
       margin-top: 4px;
   }
   
   .service-card .service-status {
       padding: 4px 12px;
       border-radius: 20px;
       font-size: 12px;
       font-weight: 600;
       text-transform: uppercase;
   }
   
   .service-card .service-details {
       display: grid;
       grid-template-columns: repeat(2, 1fr);
       gap: 15px;
   }
   
   .service-card .detail-item {
       font-size: 13px;
   }
   
   .service-card .detail-label {
       color: #6c757d;
       margin-bottom: 4px;
   }
   
   .service-card .detail-value {
       font-weight: 500;
       color: #212529;
   }
   ```

### Phase 6: Dashboard Cards

1. Dashboard widget card
   ```css
   .dashboard-card {
       background: #fff;
       border-radius: 12px;
       padding: 20px;
       box-shadow: 0 2px 10px rgba(0,0,0,0.04);
   }
   
   .dashboard-card .card-header {
       display: flex;
       justify-content: space-between;
       align-items: center;
       padding: 0 0 15px;
       margin-bottom: 15px;
       border-bottom: 1px solid #f0f0f0;
   }
   
   .dashboard-card .card-title {
       font-size: 14px;
       font-weight: 600;
       color: #495057;
       margin: 0;
   }
   
   .dashboard-card .card-icon {
       width: 40px;
       height: 40px;
       border-radius: 10px;
       display: flex;
       align-items: center;
       justify-content: center;
       font-size: 20px;
   }
   
   .dashboard-card .card-value {
       font-size: 32px;
       font-weight: 700;
       color: #212529;
       line-height: 1;
       margin-bottom: 5px;
   }
   
   .dashboard-card .card-change {
       font-size: 13px;
       color: #28a745;
   }
   
   .dashboard-card .card-change.negative {
       color: #dc3545;
   }
   ```

### Phase 7: Card Grid Layout

1. Card grid system
   ```css
   .card-grid {
       display: grid;
       grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
       gap: 24px;
   }
   
   @media (max-width: 767px) {
       .card-grid {
           grid-template-columns: 1fr;
       }
   }
   ```

### Phase 8: Interactive Cards

1. Clickable cards
   ```css
   .card-clickable {
       cursor: pointer;
   }
   
   .card-clickable:hover {
       transform: translateY(-5px);
       box-shadow: 0 10px 30px rgba(0,0,0,0.12);
   }
   
   .card-clickable:active {
       transform: translateY(-2px);
   }
   ```

2. Selectable cards
   ```css
   .card-selectable {
       position: relative;
   }
   
   .card-selectable::after {
       content: "";
       position: absolute;
       top: 15px;
       right: 15px;
       width: 24px;
       height: 24px;
       border: 2px solid #dee2e6;
       border-radius: 50%;
       background: #fff;
       transition: all 0.2s;
   }
   
   .card-selectable.selected::after {
       background: #667eea;
       border-color: #667eea;
   }
   
   .card-selectable.selected::before {
       content: "";
       position: absolute;
       top: 21px;
       right: 21px;
       width: 12px;
       height: 6px;
       border-left: 2px solid #fff;
       border-bottom: 2px solid #fff;
       transform: rotate(-45deg);
       z-index: 1;
   }
   ```

## Related Workflows
- whmcs-css-customization
- whmcs-table-styling
- whmcs-widget-styling
- whmcs-button-styling
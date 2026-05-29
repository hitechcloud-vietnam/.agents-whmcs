# WHMCS Footer Customization Workflow

## Purpose
Guide developers through customizing the WHMCS footer area.

## Prerequisites
- WHMCS installation
- HTML/CSS skills
- Smarty template knowledge
- FTP or file manager access

## Steps

### Phase 1: Footer Structure Analysis

1. Locate footer templates
   ```
   /whmcs/templates/six/layout/
   ├── footer.tpl
   └── footer-v2.tpl
   ```

2. Footer components
   ```
   Footer structure:
   ├── Main footer
   │   ├── About section
   │   ├── Quick links
   │   ├── Services
   │   ├── Support
   │   └── Contact info
   ├── Newsletter signup
   ├── Social media links
   ├── Payment icons
   └── Copyright bar
   ```

3. Footer hook points
   ```
   Footer hooks:
   - ClientAreaFooter
   - ClientAreaPageFooter
   ```

### Phase 2: Basic Footer Modifications

1. Standard footer structure
   ```smarty
   <footer class="site-footer">
       <div class="footer-main">
           <div class="container">
               <div class="row">
                   <div class="col-md-4">
                       <h5>About Us</h5>
                       <p>{$companyname} - Your trusted hosting partner since {$companyestablished|default:'2020'}</p>
                   </div>
                   <div class="col-md-2">
                       <h5>Services</h5>
                       <ul class="footer-links">
                           <li><a href="{$WEB_ROOT}/cart.php">Web Hosting</a></li>
                           <li><a href="{$WEB_ROOT}/cart.php">VPS</a></li>
                           <li><a href="{$WEB_ROOT}/cart.php">Dedicated</a></li>
                       </ul>
                   </div>
                   <div class="col-md-2">
                       <h5>Support</h5>
                       <ul class="footer-links">
                           <li><a href="{$WEB_ROOT}/supporttickets.php">Tickets</a></li>
                           <li><a href="{$WEB_ROOT}/knowledgebase.php">Knowledge Base</a></li>
                           <li><a href="{$WEB_ROOT}/serverstatus.php">Status</a></li>
                       </ul>
                   </div>
                   <div class="col-md-4">
                       <h5>Contact</h5>
                       <p><i class="fa fa-phone"></i> {$companyphone}</p>
                       <p><i class="fa fa-envelope"></i> {$companyemail}</p>
                   </div>
               </div>
           </div>
       </div>
       
       <div class="footer-bottom">
           <div class="container">
               <p>&copy; {$date_year} {$companyname}. All rights reserved.</p>
           </div>
       </div>
   </footer>
   ```

### Phase 3: Multi-Column Footer

1. CSS layout
   ```css
   .site-footer {
       background: #1a1a2e;
       color: #fff;
       padding: 60px 0 30px;
   }
   
   .footer-main {
       padding-bottom: 40px;
       border-bottom: 1px solid rgba(255,255,255,0.1);
   }
   
   .footer-column h5 {
       color: #fff;
       font-size: 16px;
       font-weight: 600;
       margin-bottom: 20px;
   }
   
   .footer-links {
       list-style: none;
       padding: 0;
       margin: 0;
   }
   
   .footer-links li {
       margin-bottom: 10px;
   }
   
   .footer-links a {
       color: rgba(255,255,255,0.7);
       transition: color 0.3s;
   }
   
   .footer-links a:hover {
       color: #fff;
   }
   ```

2. Responsive footer columns
   ```css
   .footer-main .row > div {
       margin-bottom: 30px;
   }
   
   @media (min-width: 768px) {
       .footer-main .row {
           display: flex;
           flex-wrap: wrap;
       }
       
       .footer-main .row > div:nth-child(1) { flex: 0 0 33.333%; }
       .footer-main .row > div:nth-child(2) { flex: 0 0 16.667%; }
       .footer-main .row > div:nth-child(3) { flex: 0 0 16.667%; }
       .footer-main .row > div:nth-child(4) { flex: 0 0 33.333%; }
   }
   ```

### Phase 4: Newsletter Section

1. Newsletter form
   ```smarty
   <div class="footer-newsletter">
       <h5>Subscribe to Newsletter</h5>
       <form action="subscribe.php" method="post" class="newsletter-form">
           <div class="input-group">
               <input type="email" 
                      name="email" 
                      class="form-control" 
                      placeholder="Enter your email"
                      required>
               <button type="submit" class="btn btn-primary">
                   Subscribe
               </button>
           </div>
       </form>
   </div>
   ```

2. Newsletter CSS
   ```css
   .footer-newsletter {
       background: rgba(255,255,255,0.05);
       padding: 20px;
       border-radius: 8px;
       margin-top: 20px;
   }
   
   .newsletter-form .form-control {
       background: rgba(255,255,255,0.1);
       border: none;
       color: #fff;
   }
   
   .newsletter-form .form-control::placeholder {
       color: rgba(255,255,255,0.5);
   }
   
   .newsletter-form .btn {
       background: #667eea;
       border: none;
   }
   ```

### Phase 5: Social Media Links

1. Social icons
   ```smarty
   <div class="footer-social">
       <a href="{$facebook_url|default:'#'}" target="_blank" rel="noopener" aria-label="Facebook">
           <i class="fab fa-facebook-f"></i>
       </a>
       <a href="{$twitter_url|default:'#'}" target="_blank" rel="noopener" aria-label="Twitter">
           <i class="fab fa-twitter"></i>
       </a>
       <a href="{$linkedin_url|default:'#'}" target="_blank" rel="noopener" aria-label="LinkedIn">
           <i class="fab fa-linkedin-in"></i>
       </a>
       <a href="{$instagram_url|default:'#'}" target="_blank" rel="noopener" aria-label="Instagram">
           <i class="fab fa-instagram"></i>
       </a>
   </div>
   ```

2. Social icons CSS
   ```css
   .footer-social {
       display: flex;
       gap: 15px;
       margin-top: 15px;
   }
   
   .footer-social a {
       display: flex;
       align-items: center;
       justify-content: center;
       width: 40px;
       height: 40px;
       background: rgba(255,255,255,0.1);
       border-radius: 50%;
       color: #fff;
       transition: background 0.3s, transform 0.3s;
   }
   
   .footer-social a:hover {
       background: #667eea;
       transform: translateY(-3px);
   }
   
   .footer-social i {
       font-size: 18px;
   }
   ```

### Phase 6: Payment Icons

1. Payment methods
   ```smarty
   <div class="footer-payments">
       <span class="payment-label">We Accept:</span>
       <img src="{$BASE_URL_CUSTOM_TEMPLATE}img/payment/visa.svg" alt="Visa">
       <img src="{$BASE_URL_CUSTOM_TEMPLATE}img/payment/mastercard.svg" alt="Mastercard">
       <img src="{$BASE_URL_CUSTOM_TEMPLATE}img/payment/paypal.svg" alt="PayPal">
       <img src="{$BASE_URL_CUSTOM_TEMPLATE}img/payment/amex.svg" alt="American Express">
   </div>
   ```

2. Payment CSS
   ```css
   .footer-payments {
       display: flex;
       align-items: center;
       flex-wrap: wrap;
       gap: 10px;
       margin-top: 20px;
   }
   
   .payment-label {
       font-size: 12px;
       text-transform: uppercase;
       color: rgba(255,255,255,0.5);
       margin-right: 10px;
   }
   
   .footer-payments img {
       height: 28px;
       width: auto;
       opacity: 0.8;
       filter: brightness(0) invert(1);
   }
   
   .footer-payments img:hover {
       opacity: 1;
   }
   ```

### Phase 7: Copyright Bar

1. Copyright section
   ```smarty
   <div class="footer-bottom">
       <div class="container">
           <div class="row align-items-center">
               <div class="col-md-6">
                   <p class="copyright">
                       &copy; {$date_year} {$companyname}. 
                       All rights reserved.
                   </p>
               </div>
               <div class="col-md-6 text-md-right">
                   <div class="footer-legal">
                       <a href="{$WEB_ROOT}/privacy.php">Privacy Policy</a>
                       <a href="{$WEB_ROOT}/terms.php">Terms of Service</a>
                       <a href="{$WEB_ROOT}/refundpolicy.php">Refund Policy</a>
                   </div>
               </div>
           </div>
       </div>
   </div>
   ```

2. Copyright CSS
   ```css
   .footer-bottom {
       background: rgba(0,0,0,0.2);
       padding: 20px 0;
   }
   
   .copyright {
       font-size: 13px;
       color: rgba(255,255,255,0.6);
       margin: 0;
   }
   
   .footer-legal a {
       color: rgba(255,255,255,0.6);
       font-size: 13px;
       margin-left: 20px;
   }
   
   .footer-legal a:hover {
       color: #fff;
   }
   ```

### Phase 8: Footer Hooks

1. Hook-based footer additions
   ```php
   // hooks/footer_customization.php
   <?php
   add_hook('ClientAreaFooter', 1, function($vars) {
       return '<div class="custom-footer-content"></div>';
   });
   ```

2. Add cookie consent
   ```php
   add_hook('ClientAreaFooter', 1, function($vars) {
       echo '<script>
           window.addEventListener("load", function() {
               // Cookie consent banner code
           });
       </script>';
   });
   ```

### Phase 9: Minimal Footer

1. Compact footer design
   ```css
   /* Minimal footer for client area */
   .footer-minimal {
       background: #f8f9fa;
       padding: 20px 0;
       border-top: 1px solid #dee2e6;
   }
   
   .footer-minimal .container {
       display: flex;
       justify-content: space-between;
       align-items: center;
       flex-wrap: wrap;
       gap: 15px;
   }
   
   .footer-minimal .copyright {
       color: #6c757d;
       font-size: 13px;
       margin: 0;
   }
   
   .footer-minimal .footer-links a {
       color: #6c757d;
       font-size: 13px;
       margin-left: 15px;
   }
   ```

## Footer Enhancement Tips
- Use flexbox for alignment
- Make columns responsive
- Add hover effects to links
- Include accessibility labels for icons
- Test in multiple browsers

## Related Workflows
- whmcs-theme-customization
- whmcs-header-customization
- whmcs-css-customization
- whmcs-layout-adjustments
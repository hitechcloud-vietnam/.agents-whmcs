# WHMCS Modal Customization Workflow

## Purpose
Guide developers through customizing modal dialogs in WHMCS.

## Prerequisites
- WHMCS installation
- HTML/CSS/JS knowledge
- Bootstrap modal understanding
- jQuery familiarity

## Steps

### Phase 1: Modal Structure

1. Bootstrap modal structure
   ```html
   <div class="modal fade" id="exampleModal" tabindex="-1" role="dialog">
       <div class="modal-dialog" role="document">
           <div class="modal-content">
               <div class="modal-header">
                   <h5 class="modal-title">Modal Title</h5>
                   <button type="button" class="close" data-dismiss="modal">
                       <span>&times;</span>
                   </button>
               </div>
               <div class="modal-body">
                   Modal content here
               </div>
               <div class="modal-footer">
                   <button type="button" class="btn btn-secondary" data-dismiss="modal">Close</button>
                   <button type="button" class="btn btn-primary">Save</button>
               </div>
           </div>
       </div>
   </div>
   ```

### Phase 2: Basic Modal Styling

1. Modal CSS
   ```css
   .modal-content {
       border: none;
       border-radius: 12px;
       box-shadow: 0 10px 40px rgba(0,0,0,0.2);
   }
   
   .modal-header {
       padding: 20px 25px;
       border-bottom: 1px solid #e9ecef;
       background: #f8f9fa;
       border-radius: 12px 12px 0 0;
   }
   
   .modal-title {
       font-weight: 600;
       margin: 0;
   }
   
   .modal-header .close {
       font-size: 24px;
       padding: 0;
       margin: 0;
       opacity: 0.5;
   }
   
   .modal-header .close:hover {
       opacity: 1;
   }
   
   .modal-body {
       padding: 25px;
   }
   
   .modal-footer {
       padding: 20px 25px;
       border-top: 1px solid #e9ecef;
       background: #f8f9fa;
   }
   ```

### Phase 3: Size Variants

1. Modal sizes
   ```css
   /* Small modal */
   .modal-sm .modal-dialog {
       max-width: 400px;
   }
   
   /* Large modal */
   .modal-lg .modal-dialog {
       max-width: 800px;
   }
   
   /* Extra large modal */
   .modal-xl .modal-dialog {
       max-width: 1140px;
   }
   
   /* Full screen modal */
   @media (max-width: 767px) {
       .modal-dialog {
           max-width: 100%;
           margin: 0;
           height: 100vh;
       }
       
       .modal-content {
           height: 100%;
           border-radius: 0;
       }
   }
   ```

### Phase 4: Theme Variants

1. Dark modal
   ```css
   .modal-dark .modal-content {
       background: #212529;
       color: #fff;
   }
   
   .modal-dark .modal-header {
       background: #1a1a2e;
       border-bottom-color: rgba(255,255,255,0.1);
   }
   
   .modal-dark .modal-footer {
       background: #1a1a2e;
       border-top-color: rgba(255,255,255,0.1);
   }
   
   .modal-dark .close {
       color: #fff;
   }
   ```

2. Primary color modal
   ```css
   .modal-primary .modal-header {
       background: linear-gradient(135deg, #667eea, #764ba2);
       color: #fff;
       border: none;
   }
   
   .modal-primary .modal-header .close {
       color: #fff;
   }
   ```

### Phase 5: Custom Animations

1. Slide animation
   ```css
   .modal.fade .modal-dialog {
       transform: translateY(-30px);
       transition: transform 0.3s ease-out;
   }
   
   .modal.show .modal-dialog {
       transform: translateY(0);
   }
   ```

2. Scale animation
   ```css
   .modal.fade .modal-dialog {
       transform: scale(0.9);
       transition: transform 0.3s ease;
   }
   
   .modal.show .modal-dialog {
       transform: scale(1);
   }
   ```

3. Side slide animation
   ```css
   .modal.sidebar-modal .modal-dialog {
       position: fixed;
       right: 0;
       top: 0;
       height: 100%;
       margin: 0;
       transform: translateX(100%);
       transition: transform 0.3s ease;
   }
   
   .modal.sidebar-modal.show .modal-dialog {
       transform: translateX(0);
   }
   
   .modal.sidebar-modal .modal-content {
       height: 100%;
       border-radius: 0;
       max-width: 400px;
   }
   ```

### Phase 6: Form Modals

1. Form modal styling
   ```css
   .modal-form .modal-body {
       padding: 30px;
   }
   
   .modal-form .form-group {
       margin-bottom: 20px;
   }
   
   .modal-form .form-control {
       height: 45px;
       border-radius: 8px;
   }
   
   .modal-form .btn {
       min-height: 45px;
       border-radius: 8px;
   }
   ```

2. Contact modal example
   ```smarty
   <div class="modal fade" id="contactModal" tabindex="-1">
       <div class="modal-dialog modal-form">
           <div class="modal-content">
               <div class="modal-header">
                   <h5 class="modal-title">Contact Us</h5>
                   <button type="button" class="close" data-dismiss="modal">
                       <span>&times;</span>
                   </button>
               </div>
               <div class="modal-body">
                   <form id="contactForm">
                       <div class="form-group">
                           <label>Name</label>
                           <input type="text" class="form-control" required>
                       </div>
                       <div class="form-group">
                           <label>Email</label>
                           <input type="email" class="form-control" required>
                       </div>
                       <div class="form-group">
                           <label>Message</label>
                           <textarea class="form-control" rows="4" required></textarea>
                       </div>
                       <button type="submit" class="btn btn-primary btn-block">
                           Send Message
                       </button>
                   </form>
               </div>
           </div>
       </div>
   </div>
   ```

### Phase 7: Confirmation Modals

1. Alert modal styling
   ```css
   .modal-alert .modal-header {
       padding: 15px 25px;
   }
   
   .modal-alert .modal-body {
       padding: 25px;
       text-align: center;
   }
   
   .modal-alert .modal-icon {
       font-size: 48px;
       margin-bottom: 15px;
   }
   
   .modal-alert.success .modal-icon {
       color: #28a745;
   }
   
   .modal-alert.warning .modal-icon {
       color: #ffc107;
   }
   
   .modal-alert.danger .modal-icon {
       color: #dc3545;
   }
   ```

2. Confirmation modal template
   ```smarty
   <div class="modal fade modal-alert" id="confirmModal">
       <div class="modal-dialog">
           <div class="modal-content">
               <div class="modal-body warning">
                   <div class="modal-icon">
                       <i class="fa fa-exclamation-triangle"></i>
                   </div>
                   <h5 class="modal-title">Confirm Action</h5>
                   <p class="modal-message">
                       Are you sure you want to proceed?
                   </p>
                   <div class="modal-buttons">
                       <button type="button" class="btn btn-secondary" data-dismiss="modal">
                           Cancel
                       </button>
                       <button type="button" class="btn btn-danger" id="confirmBtn">
                           Confirm
                       </button>
                   </div>
               </div>
           </div>
       </div>
   </div>
   ```

### Phase 8: WHMCS-Specific Modals

1. Product modal
   ```smarty
   <div class="modal product-modal fade" id="productModal">
       <div class="modal-dialog modal-lg">
           <div class="modal-content">
               <div class="modal-header">
                   <h5 class="modal-title">Product Details</h5>
                   <button type="button" class="close" data-dismiss="modal">
                       <span>&times;</span>
                   </button>
               </div>
               <div class="modal-body">
                   <div class="product-modal-content">
                       <img src="" alt="Product" class="product-image">
                       <div class="product-info">
                           <h3 class="product-name"></h3>
                           <p class="product-description"></p>
                           <div class="product-price"></div>
                       </div>
                   </div>
               </div>
               <div class="modal-footer">
                   <button type="button" class="btn btn-secondary" data-dismiss="modal">
                       Close
                   </button>
                   <a href="#" class="btn btn-primary">Order Now</a>
               </div>
           </div>
       </div>
   </div>
   ```

2. Cart modal
   ```javascript
   // Open product modal with AJAX
   $(document).on('click', '.product-quick-view', function(e) {
       e.preventDefault();
       var productId = $(this).data('product-id');
       
       $.ajax({
           url: 'ajax.php',
           type: 'POST',
           data: {
               action: 'getProduct',
               id: productId
           },
           success: function(response) {
               $('#productModal .product-name').text(response.name);
               $('#productModal .product-description').text(response.description);
               $('#productModal .product-price').text(response.price);
               $('#productModal').modal('show');
           }
       });
   });
   ```

## Related Workflows
- whmcs-css-customization
- whmcs-form-styling
- whmcs-button-styling
- whmcs-template-modification
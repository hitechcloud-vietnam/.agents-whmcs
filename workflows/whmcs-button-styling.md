# WHMCS Button Styling Workflow

## Purpose
Guide developers through customizing buttons in WHMCS.

## Prerequisites
- WHMCS installation
- CSS/SCSS knowledge
- Bootstrap button components understanding

## Steps

### Phase 1: Button Structure

1. Button types in WHMCS
   ```
   Button Types:
   ├── Primary buttons
   ├── Secondary buttons
   ├── Outline buttons
   ├── Ghost buttons
   ├── Icon buttons
   └── Button groups
   ```

2. Bootstrap button classes
   ```
   Button Classes:
   - .btn
   - .btn-primary, .btn-secondary, .btn-success, .btn-danger, .btn-warning, .btn-info
   - .btn-outline-primary, etc.
   - .btn-sm, .btn-lg
   - .btn-block
   ```

### Phase 2: Basic Button Styling

1. Button base styles
   ```css
   .btn {
       display: inline-flex;
       align-items: center;
       justify-content: center;
       padding: 10px 24px;
       font-size: 14px;
       font-weight: 600;
       line-height: 1.5;
       border-radius: 6px;
       border: 1px solid transparent;
       cursor: pointer;
       transition: all 0.2s ease;
       text-decoration: none;
       white-space: nowrap;
   }
   
   .btn:focus {
       outline: none;
       box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.25);
   }
   
   .btn:disabled {
       opacity: 0.6;
       cursor: not-allowed;
   }
   ```

### Phase 3: Primary Buttons

1. Gradient primary button
   ```css
   .btn-primary {
       background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
       color: #fff;
       border: none;
       box-shadow: 0 2px 8px rgba(102, 126, 234, 0.3);
   }
   
   .btn-primary:hover {
       background: linear-gradient(135deg, #5a6fd6 0%, #6a4393 100%);
       box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
       transform: translateY(-2px);
   }
   
   .btn-primary:active {
       transform: translateY(0);
       box-shadow: 0 2px 6px rgba(102, 126, 234, 0.3);
   }
   ```

2. Solid primary button
   ```css
   .btn-primary-solid {
       background-color: #007bff;
       color: #fff;
       border: none;
   }
   
   .btn-primary-solid:hover {
       background-color: #0056b3;
       box-shadow: 0 4px 12px rgba(0, 123, 255, 0.3);
   }
   ```

### Phase 4: Secondary Buttons

1. Secondary button styles
   ```css
   .btn-secondary {
       background-color: #6c757d;
       color: #fff;
       border: none;
   }
   
   .btn-secondary:hover {
       background-color: #545b62;
   }
   ```

### Phase 5: Outline Buttons

1. Outline primary
   ```css
   .btn-outline-primary {
       background: transparent;
       color: #667eea;
       border: 2px solid #667eea;
       padding: 8px 22px;
   }
   
   .btn-outline-primary:hover {
       background: #667eea;
       color: #fff;
   }
   
   .btn-outline-primary:active {
       background: #5a6fd6;
   }
   ```

2. Outline secondary
   ```css
   .btn-outline-secondary {
       background: transparent;
       color: #6c757d;
       border: 2px solid #ced4da;
       padding: 8px 22px;
   }
   
   .btn-outline-secondary:hover {
       background: #6c757d;
       border-color: #6c757d;
       color: #fff;
   }
   ```

### Phase 6: Ghost Buttons

1. Ghost button styles
   ```css
   .btn-ghost {
       background: transparent;
       color: #495057;
       border: none;
       padding: 8px 16px;
   }
   
   .btn-ghost:hover {
       background: #f8f9fa;
       color: #212529;
   }
   
   .btn-ghost-primary {
       color: #667eea;
   }
   
   .btn-ghost-primary:hover {
       background: rgba(102, 126, 234, 0.1);
       color: #667eea;
   }
   ```

### Phase 7: Icon Buttons

1. Icon button styles
   ```css
   .btn-icon {
       width: 40px;
       height: 40px;
       padding: 0;
       border-radius: 50%;
       display: inline-flex;
       align-items: center;
       justify-content: center;
   }
   
   .btn-icon i {
       font-size: 18px;
   }
   
   .btn-icon.btn-lg {
       width: 50px;
       height: 50px;
   }
   
   .btn-icon.btn-sm {
       width: 32px;
       height: 32px;
   }
   ```

2. Icon with text button
   ```css
   .btn-icon-text {
       display: inline-flex;
       align-items: center;
       gap: 10px;
   }
   
   .btn-icon-text i {
       font-size: 18px;
   }
   ```

### Phase 8: Button Sizes

1. Size variants
   ```css
   .btn-sm {
       padding: 6px 16px;
       font-size: 13px;
       border-radius: 4px;
   }
   
   .btn-lg {
       padding: 14px 32px;
       font-size: 16px;
       border-radius: 8px;
   }
   
   .btn-xl {
       padding: 18px 40px;
       font-size: 18px;
       border-radius: 10px;
   }
   ```

2. Full-width button
   ```css
   .btn-block {
       display: block;
       width: 100%;
   }
   ```

### Phase 9: Button States

1. Hover effects
   ```css
   .btn-transform:hover {
       transform: translateY(-2px);
   }
   
   .btn-grow:hover {
       transform: scale(1.05);
   }
   
   .btn-shadow:hover {
       box-shadow: 0 6px 20px rgba(0, 0, 0, 0.15);
   }
   ```

2. Loading state
   ```css
   .btn-loading {
       position: relative;
       color: transparent !important;
       pointer-events: none;
   }
   
   .btn-loading::after {
       content: "";
       position: absolute;
       width: 18px;
       height: 18px;
       top: 50%;
       left: 50%;
       margin-top: -9px;
       margin-left: -9px;
       border: 2px solid rgba(255, 255, 255, 0.3);
       border-top-color: #fff;
       border-radius: 50%;
       animation: btn-spinner 0.8s linear infinite;
   }
   
   @keyframes btn-spinner {
       to { transform: rotate(360deg); }
   }
   ```

### Phase 10: Button Groups

1. Button group styles
   ```css
   .btn-group .btn {
       border-radius: 0;
   }
   
   .btn-group .btn:first-child {
       border-radius: 6px 0 0 6px;
   }
   
   .btn-group .btn:last-child {
       border-radius: 0 6px 6px 0;
   }
   ```

2. Toolbar style
   ```css
   .btn-toolbar {
       display: flex;
       flex-wrap: wrap;
       gap: 10px;
       align-items: center;
   }
   ```

## Related Workflows
- whmcs-css-customization
- whmcs-form-styling
- whmcs-color-scheme
- whmcs-button-styling
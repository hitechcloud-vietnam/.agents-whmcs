# WHMCS Form Styling Workflow

## Purpose
Guide developers through customizing form elements in WHMCS.

## Prerequisites
- WHMCS installation
- CSS/SCSS knowledge
- Bootstrap form components understanding
- HTML form structure knowledge

## Steps

### Phase 1: Form Structure

1. Form template locations
   ```
   WHMCS Form Templates:
   - /whmcs/templates/six/forms/
   - Inline within page templates
   - Order form templates in /orderforms/
   ```

2. Common form elements
   ```
   WHMCS Forms:
   ├── Text inputs
   ├── Textareas
   ├── Selects/Dropdowns
   ├── Checkboxes
   ├── Radio buttons
   ├── File uploads
   ├── Date pickers
   └── Password fields
   ```

3. Bootstrap form structure
   ```html
   <form>
       <div class="form-group">
           <label for="inputField">Label</label>
           <input type="text" class="form-control" id="inputField">
           <small class="form-text text-muted">Help text</small>
       </div>
       
       <div class="form-group">
           <div class="custom-control custom-checkbox">
               <input type="checkbox" class="custom-control-input" id="customCheck">
               <label class="custom-control-label" for="customCheck">Checkbox</label>
           </div>
       </div>
       
       <button type="submit" class="btn btn-primary">Submit</button>
   </form>
   ```

### Phase 2: Input Field Styling

1. Text input styling
   ```css
   .form-control {
       height: 45px;
       padding: 10px 16px;
       font-size: 15px;
       border: 1px solid #ced4da;
       border-radius: 6px;
       transition: border-color 0.2s, box-shadow 0.2s;
   }
   
   .form-control:focus {
       border-color: #667eea;
       box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.15);
       outline: none;
   }
   
   .form-control::placeholder {
       color: #adb5bd;
   }
   
   .form-control:disabled {
       background-color: #e9ecef;
       opacity: 0.7;
   }
   ```

2. Input with icon
   ```css
   .input-icon-wrapper {
       position: relative;
   }
   
   .input-icon-wrapper .form-control {
       padding-left: 45px;
   }
   
   .input-icon-wrapper .input-icon {
       position: absolute;
       left: 15px;
       top: 50%;
       transform: translateY(-50%);
       color: #adb5bd;
   }
   
   .input-icon-wrapper .form-control:focus + .input-icon,
   .input-icon-wrapper:focus-within .input-icon {
       color: #667eea;
   }
   ```

3. Floating label inputs
   ```css
   .form-floating {
       position: relative;
   }
   
   .form-floating .form-control {
       padding: 20px 16px 8px;
       height: auto;
   }
   
   .form-floating label {
       position: absolute;
       top: 0;
       left: 0;
       padding: 16px;
       pointer-events: none;
       color: #6c757d;
       transition: all 0.2s;
   }
   
   .form-floating .form-control:focus ~ label,
   .form-floating .form-control:not(:placeholder-shown) ~ label {
       top: 4px;
       font-size: 12px;
       color: #667eea;
   }
   ```

### Phase 3: Textarea Styling

1. Textarea styles
   ```css
   textarea.form-control {
       min-height: 120px;
       height: auto;
       resize: vertical;
   }
   
   .form-textarea-lg {
       min-height: 200px;
   }
   
   .form-textarea-sm {
       min-height: 80px;
   }
   ```

2. Rich text editor styling
   ```css
   .tox-tinymce {
       border-radius: 6px;
       border: 1px solid #ced4da;
   }
   
   .tox-tinymce:focus {
       border-color: #667eea;
       box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.15);
   }
   ```

### Phase 4: Select/Dropdown Styling

1. Custom select styling
   ```css
   .custom-select {
       height: 45px;
       padding: 10px 40px 10px 16px;
       border: 1px solid #ced4da;
       border-radius: 6px;
       background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%236c757d' d='M6 9L1 4h10z'/%3E%3C/svg%3E");
       background-repeat: no-repeat;
       background-position: right 15px center;
       appearance: none;
   }
   
   .custom-select:focus {
       border-color: #667eea;
       box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.15);
   }
   ```

2. Multi-select styling
   ```css
   select[multiple] {
       height: auto;
       padding: 10px;
   }
   
   select[multiple] option {
       padding: 8px 12px;
       border-radius: 4px;
   }
   
   select[multiple] option:checked {
       background: #667eea;
       color: #fff;
   }
   ```

### Phase 5: Checkbox and Radio Styling

1. Custom checkbox
   ```css
   .custom-checkbox {
       padding-left: 30px;
       min-height: 24px;
       display: flex;
       align-items: center;
   }
   
   .custom-checkbox .custom-control-input {
       width: 20px;
       height: 20px;
       margin-left: -30px;
       cursor: pointer;
   }
   
   .custom-checkbox .custom-control-label {
       cursor: pointer;
       position: static;
   }
   
   .custom-checkbox .custom-control-label::before {
       left: 0;
       top: 50%;
       transform: translateY(-50%);
       width: 20px;
       height: 20px;
       border: 2px solid #ced4da;
       border-radius: 4px;
       background-color: #fff;
       transition: all 0.2s;
   }
   
   .custom-checkbox .custom-control-input:checked ~ .custom-control-label::before {
       background: #667eea;
       border-color: #667eea;
   }
   
   .custom-checkbox .custom-control-input:checked ~ .custom-control-label::after {
       background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%23fff' d='M9.8 2.7L4.5 8 2.2 5.7c-.4-.4-1-.4-1.4 0-.4.4-.4 1 0 1.4l3.5 3.5c.2.2.5.3.7.3.3 0 .5-.1.7-.3l5.5-5.5c.4-.4.4-1 0-1.4-.4-.4-1-.4-1.4 0z'/%3E%3C/svg%3E");
   }
   ```

2. Custom radio buttons
   ```css
   .custom-radio .custom-control-label::before {
       width: 20px;
       height: 20px;
       border-radius: 50%;
       border: 2px solid #ced4da;
   }
   
   .custom-radio .custom-control-input:checked ~ .custom-control-label::before {
       background: #667eea;
       border-color: #667eea;
   }
   
   .custom-radio .custom-control-input:checked ~ .custom-control-label::after {
       background-image: none;
       top: 50%;
       left: 5px;
       width: 10px;
       height: 10px;
       border-radius: 50%;
       background: #fff;
       transform: translateY(-50%);
   }
   ```

3. Toggle switch
   ```css
   .custom-switch {
       padding-left: 50px;
   }
   
   .custom-switch .custom-control-label::before {
       left: -50px;
       width: 40px;
       height: 22px;
       border-radius: 11px;
       background-color: #ced4da;
   }
   
   .custom-switch .custom-control-label::after {
       top: 2px;
       left: -48px;
       width: 18px;
       height: 18px;
       border-radius: 50%;
       background-color: #fff;
       transition: transform 0.2s;
   }
   
   .custom-switch .custom-control-input:checked ~ .custom-control-label::before {
       background-color: #667eea;
   }
   
   .custom-switch .custom-control-input:checked ~ .custom-control-label::after {
       transform: translateX(18px);
   }
   ```

### Phase 6: Form Validation Styling

1. Valid state
   ```css
   .form-control.is-valid {
       border-color: #28a745;
       background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%2328a745' d='M9.8 2.7L4.5 8 2.2 5.7c-.4-.4-1-.4-1.4 0-.4.4-.4 1 0 1.4l3.5 3.5c.2.2.5.3.7.3.3 0 .5-.1.7-.3l5.5-5.5c.4-.4.4-1 0-1.4-.4-.4-1-.4-1.4 0z'/%3E%3C/svg%3E");
       background-repeat: no-repeat;
       background-position: right 10px center;
       padding-right: 40px;
   }
   
   .form-control.is-valid:focus {
       box-shadow: 0 0 0 3px rgba(40, 167, 69, 0.15);
   }
   ```

2. Invalid state
   ```css
   .form-control.is-invalid {
       border-color: #dc3545;
       background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%23dc3545' d='M9.8 2.7L4.5 8 2.2 5.7c-.4-.4-1-.4-1.4 0-.4.4-.4 1 0 1.4l3.5 3.5c.2.2.5.3.7.3.3 0 .5-.1.7-.3l5.5-5.5c.4-.4.4-1 0-1.4-.4-.4-1-.4-1.4 0z'/%3E%3C/svg%3E");
       background-repeat: no-repeat;
       background-position: right 10px center;
       padding-right: 40px;
   }
   
   .form-control.is-invalid:focus {
       box-shadow: 0 0 0 3px rgba(220, 53, 69, 0.15);
   }
   
   .invalid-feedback {
       color: #dc3545;
       font-size: 13px;
       margin-top: 5px;
   }
   ```

### Phase 7: Form Layout

1. Inline forms
   ```css
   .form-inline {
       display: flex;
       flex-wrap: wrap;
       align-items: center;
       gap: 15px;
   }
   
   .form-inline .form-group {
       display: flex;
       align-items: center;
       gap: 10px;
   }
   
   .form-inline label {
       margin: 0;
       white-space: nowrap;
   }
   ```

2. Horizontal forms
   ```css
   .form-horizontal .form-group {
       display: flex;
       align-items: flex-start;
       margin-bottom: 20px;
   }
   
   .form-horizontal .col-form-label {
       padding-top: calc(.375rem + 1px);
       padding-bottom: calc(.375rem + 1px);
       margin-bottom: 0;
   }
   
   @media (min-width: 576px) {
       .form-horizontal label {
           text-align: right;
       }
   }
   ```

### Phase 8: Form Actions

1. Button styling
   ```css
   .form-actions {
       display: flex;
       gap: 15px;
       padding-top: 20px;
       border-top: 1px solid #e9ecef;
       margin-top: 25px;
   }
   
   .form-actions .btn {
       min-width: 120px;
       height: 45px;
       border-radius: 6px;
       font-weight: 600;
   }
   
   .form-actions .btn-primary {
       background: linear-gradient(135deg, #667eea, #764ba2);
       border: none;
   }
   
   .form-actions .btn-primary:hover {
       box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
       transform: translateY(-1px);
   }
   ```

## Related Workflows
- whmcs-css-customization
- whmcs-button-styling
- whmcs-modal-customization
- whmcs-template-modification
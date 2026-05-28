# WHMCS Order Form Builder Module

A powerful customizable order form builder for WHMCS with drag-and-drop product placement, multi-step checkout, and flexible configuration.

## Features

- Custom order form creation
- Multi-step checkout flow
- Product positioning control
- Custom pricing per form
- Featured products highlighting
- Quantity selectors
- Product comparison
- Form analytics
- Abandoned cart tracking
- Template support

## Installation

1. Copy the module to your WHMCS installation:
   ```
   /path/to/whmcs/modules/servers/orderformbuilder/
   ```

2. Activate through WHMCS Admin:
   - Go to **Setup > Addon Modules**
   - Find "Order Form Builder"
   - Click **Activate**
   - Configure settings

3. Module Settings:
   - **Default Form**: Default form identifier
   - **Enable Multi-Step**: Enable multi-step checkout
   - **Enable Product Comparison**: Allow product comparison
   - **Enable Quantity Selector**: Show quantity input
   - **Default Currency**: Default currency selection

## Usage

### Creating Order Forms

```php
// Create a new order form
$result = orderformbuilder_CreateForm(array(
    'name' => 'Custom Hosting Form',
    'description' => 'Special hosting order form',
    'config' => array(
        'enable_multistep' => true,
        'show_categories' => true,
    ),
    'products' => array(1, 2, 3), // Product IDs
    'categories' => array('hosting', 'addons'),
    'template' => 'modern',
));

$formKey = $result['form_key'];
```

### Managing Forms

```php
// Get all forms
$forms = orderformbuilder_GetForms(true);

// Get specific form
$form = orderformbuilder_GetForm($formKey);

// Get default form
$form = orderformbuilder_GetDefaultForm();

// Set as default
orderformbuilder_SetDefaultForm($formKey);

// Update form
orderformbuilder_UpdateForm($formKey, array(
    'name' => 'Updated Form Name',
    'config' => array('enable_multistep' => false),
));

// Delete form
orderformbuilder_DeleteForm($formKey);
```

### Managing Products

```php
// Add product to form
orderformbuilder_AddProduct($formKey, $productId, array(
    'position' => 1,
    'is_featured' => true,
    'pricing' => array('monthly' => 9.99, 'annually' => 99.99),
    'features' => array('Feature 1', 'Feature 2'),
    'min_quantity' => 1,
    'max_quantity' => 10,
));

// Update product config
orderformbuilder_UpdateProductConfig($formKey, $productId, array(
    'position' => 2,
    'is_featured' => false,
    'custom_features' => array('New Feature'),
));

// Remove product
orderformbuilder_RemoveProduct($formKey, $productId);

// Get product configs
$configs = orderformbuilder_GetProductConfigs($formId);
```

### Working with Submissions

```php
// Create new submission (cart session)
$result = orderformbuilder_CreateSubmission($formKey, session_id(), $userId);
$submissionId = $result['submission_id'];

// Update cart data
$cartData = array(
    'products' => array(
        array('product_id' => 1, 'quantity' => 1, 'config' => array()),
    ),
    'total' => 19.99,
);

orderformbuilder_UpdateSubmission($submissionId, $cartData);

// Complete submission (after payment)
orderformbuilder_CompleteSubmission($submissionId);

// Get submission
$submission = orderformbuilder_GetSubmission($submissionId);
```

### Statistics

```php
// Get form statistics
$stats = orderformbuilder_GetFormStats($formKey);
// Returns: total, completed, in_progress, abandoned, conversion_rate

// Get all forms stats
$allStats = orderformbuilder_GetFormStats();
```

### Rendering

```php
// Render form
$html = orderformbuilder_RenderForm($formKey, array(
    'current_step' => 1,
));

echo $html;
```

## Default Steps

1. **Select Products** - Choose products and services
2. **Configure** - Configure selected items
3. **Checkout** - Review and checkout
4. **Complete** - Order confirmation

## Database Tables

- `mod_orderformbuilder_forms` - Order form definitions
- `mod_orderformbuilder_submissions` - Cart/order submissions
- `mod_orderformbuilder_configs` - Product configurations per form

## API Functions

| Function | Description |
|----------|-------------|
| `orderformbuilder_CreateForm()` | Create new order form |
| `orderformbuilder_GetForms()` | Get all forms |
| `orderformbuilder_GetForm()` | Get form by key |
| `orderformbuilder_GetDefaultForm()` | Get default form |
| `orderformbuilder_SetDefaultForm()` | Set default form |
| `orderformbuilder_UpdateForm()` | Update form |
| `orderformbuilder_DeleteForm()` | Delete form |
| `orderformbuilder_AddProduct()` | Add product to form |
| `orderformbuilder_RemoveProduct()` | Remove product |
| `orderformbuilder_UpdateProductConfig()` | Update product config |
| `orderformbuilder_GetProductConfigs()` | Get product configs |
| `orderformbuilder_CreateSubmission()` | Create submission |
| `orderformbuilder_UpdateSubmission()` | Update submission |
| `orderformbuilder_CompleteSubmission()` | Complete submission |
| `orderformbuilder_GetSubmission()` | Get submission |
| `orderformbuilder_GetFormStats()` | Get form statistics |
| `orderformbuilder_RenderForm()` | Render form HTML |

## Version History

- **1.0.0** - Initial release
  - Custom order form creation
  - Multi-step checkout
  - Product management
  - Form submissions tracking
  - Statistics and analytics

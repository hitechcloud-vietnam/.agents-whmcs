# WHMCS Product Configurator Module

A step-by-step product configuration wizard for WHMCS with live pricing, option validation, and flexible templates.

## Features

- Step-by-step configuration wizard
- Live price calculation
- Option validation
- Multiple selection types
- Price modifiers (fixed, percentage, multiplier)
- Session persistence
- Template-based configuration
- Progress tracking

## Installation

1. Copy the module to your WHMCS installation:
   ```
   /path/to/whmcs/modules/servers/productconfigurator/
   ```

2. Activate through WHMCS Admin:
   - Go to **Setup > Addon Modules**
   - Find "Product Configurator"
   - Click **Activate**
   - Configure settings

## Usage

### Creating Templates

```php
// Create configuration template
$result = productconfigurator_CreateTemplate($productId, array(
    'name' => 'Server Configuration',
    'steps' => array(
        array('key' => 'hardware', 'name' => 'Hardware', 'description' => 'Select hardware specs'),
        array('key' => 'software', 'name' => 'Software', 'description' => 'Choose OS and software'),
        array('key' => 'addons', 'name' => 'Add-ons', 'description' => 'Optional features'),
    ),
    'options' => array(
        'hardware' => array(
            array('key' => 'cpu_basic', 'name' => 'Basic CPU', 'price' => 0),
            array('key' => 'cpu_pro', 'name' => 'Pro CPU', 'price' => 20),
            array('key' => 'cpu_enterprise', 'name' => 'Enterprise CPU', 'price' => 50),
        ),
        'software' => array(
            array('key' => 'os_centos', 'name' => 'CentOS', 'price' => 0),
            array('key' => 'os_ubuntu', 'name' => 'Ubuntu', 'price' => 0),
            array('key' => 'os_windows', 'name' => 'Windows Server', 'price' => 30),
        ),
        'addons' => array(
            array('key' => 'backup', 'name' => 'Automated Backups', 'price' => 10),
            array('key' => 'monitoring', 'name' => '24/7 Monitoring', 'price' => 15),
            array('key' => 'ssl', 'name' => 'SSL Certificate', 'price' => 25),
        ),
    ),
    'pricing_rules' => array(
        array('step' => 'hardware', 'required' => true),
        array('step' => 'software', 'required' => true),
    ),
));
```

### Managing Templates

```php
// Get template for product
$template = productconfigurator_GetTemplateForProduct($productId);

// Get template by key
$template = productconfigurator_GetTemplate($templateKey);

// Get all templates
$templates = productconfigurator_GetTemplates();

// Update template
productconfigurator_UpdateTemplate($templateKey, array(
    'name' => 'Updated Name',
    'steps' => $newSteps,
));

// Delete template
productconfigurator_DeleteTemplate($templateKey);
```

### Managing Sessions

```php
// Start configuration session
$result = productconfigurator_StartSession($templateKey, $productId, $userId);
$sessionId = $result['session_id'];

// Get session
$session = productconfigurator_GetSession($sessionId);

// Update selection
$result = productconfigurator_UpdateSelection($sessionId, 'hardware', array('cpu_pro'));

// Navigate steps
productconfigurator_NextStep($sessionId);
productconfigurator_PreviousStep($sessionId);

// Complete configuration
$result = productconfigurator_CompleteSession($sessionId);
// Returns: selections, pricing
```

### Price Modifiers

```php
// Add price modifier
productconfigurator_AddPriceModifier($templateKey, 'cpu_pro', 'fixed', 20.00);
productconfigurator_AddPriceModifier($templateKey, 'premium_support', 'percentage', 15);
productconfigurator_AddPriceModifier($templateKey, 'volume_discount', 'multiplier', 0.9);

// Get modifier (returns: modifier_type, modifier_value)
$modifier = productconfigurator_GetPriceModifier($templateId, 'cpu_pro', 1);
```

### Rendering

```php
// Render wizard
$html = productconfigurator_RenderWizard($sessionId);
echo $html;
```

## Template Structure

```php
$template = array(
    'template_key' => 'tmpl-123-abc',
    'product_id' => 123,
    'name' => 'Server Configurator',
    'steps' => array(
        array(
            'key' => 'step_key',
            'name' => 'Step Name',
            'description' => 'Step description',
        ),
    ),
    'options' => array(
        'step_key' => array(
            array(
                'key' => 'option_key',
                'name' => 'Option Name',
                'description' => 'Option description',
                'price' => 10.00,
                'multi_select' => false,
            ),
        ),
    ),
    'pricing_rules' => array(
        array('step' => 'step_key', 'required' => true),
    ),
);
```

## Database Tables

- `mod_productconfigurator_templates` - Configuration templates
- `mod_productconfigurator_sessions` - Configuration sessions
- `mod_productconfigurator_pricing` - Price modifiers

## API Functions

| Function | Description |
|----------|-------------|
| `productconfigurator_CreateTemplate()` | Create template |
| `productconfigurator_GetTemplate()` | Get template by key |
| `productconfigurator_GetTemplateForProduct()` | Get template for product |
| `productconfigurator_GetTemplates()` | Get all templates |
| `productconfigurator_UpdateTemplate()` | Update template |
| `productconfigurator_DeleteTemplate()` | Delete template |
| `productconfigurator_StartSession()` | Start config session |
| `productconfigurator_GetSession()` | Get session |
| `productconfigurator_UpdateSelection()` | Update selections |
| `productconfigurator_NextStep()` | Go to next step |
| `productconfigurator_PreviousStep()` | Go to previous step |
| `productconfigurator_CompleteSession()` | Complete configuration |
| `productconfigurator_AddPriceModifier()` | Add price modifier |
| `productconfigurator_GetPriceModifier()` | Get price modifier |
| `productconfigurator_RenderWizard()` | Render wizard HTML |

## Version History

- **1.0.0** - Initial release
  - Step-by-step configuration wizard
  - Live pricing calculation
  - Option validation
  - Price modifiers
  - Session management

# WHMCS Order Form Builder Module

```php
<?php
/**
 * WHMCS Order Form Builder Module
 * 
 * Creates customizable order forms with drag-and-drop product placement,
 * custom fields, and flexible pricing options.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access prohibited");
}

function orderformbuilder_MetaData()
{
    return array(
        'DisplayName' => 'Order Form Builder',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    );
}

function orderformbuilder_ConfigArray()
{
    return array(
        'FriendlyName' => array(
            'Type' => 'System',
            'Value' => 'Order Form Builder',
        ),
        'DefaultForm' => array(
            'Type' => 'text',
            'Size' => '50',
            'Default' => 'standard',
            'Description' => 'Default order form identifier',
        ),
        'EnableMultiStep' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable multi-step checkout flow',
        ),
        'EnableProductComparison' => array(
            'Type' => 'yesno',
            'Default' => 'no',
            'Description' => 'Allow product comparison',
        ),
        'EnableQuantitySelector' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Show quantity selector',
        ),
        'DefaultCurrency' => array(
            'Type' => 'dropdown',
            'Options' => array(
                '' => 'Auto Detect',
                'USD' => 'USD',
                'EUR' => 'EUR',
                'GBP' => 'GBP',
            ),
            'Default' => '',
            'Description' => 'Default currency for new orders',
        ),
    );
}

function orderformbuilder_activate()
{
    try {
        if (!function_exists('createTable')) {
            require_once dirname(__FILE__) . '/../../includes/modulefunctions.php';
        }
        
        // Order forms table
        $formsTable = 'mod_orderformbuilder_forms';
        $formsSchema = "
            CREATE TABLE `{$formsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `form_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `config` JSON NOT NULL,
                `products` JSON NOT NULL,
                `categories` JSON NULL,
                `steps` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `is_default` TINYINT(1) DEFAULT 0,
                `created_by` INT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_form_key` (`form_key`),
                INDEX `idx_is_active` (`is_active`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($formsTable, $formsSchema);
        
        // Form submissions table
        $submissionsTable = 'mod_orderformbuilder_submissions';
        $submissionsSchema = "
            CREATE TABLE `{$submissionsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `form_id` INT NOT NULL,
                `session_id` VARCHAR(255) NULL,
                `user_id` INT NULL,
                `cart_data` JSON NOT NULL,
                `custom_fields` JSON NULL,
                `step_reached` INT DEFAULT 1,
                `status` ENUM('in_progress', 'completed', 'abandoned') DEFAULT 'in_progress',
                `ip_address` VARCHAR(45) NULL,
                `user_agent` VARCHAR(500) NULL,
                `started_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `completed_at` DATETIME NULL,
                INDEX `idx_form_id` (`form_id`),
                INDEX `idx_user_id` (`user_id`),
                INDEX `idx_status` (`status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($submissionsTable, $submissionsSchema);
        
        // Product configurations table
        $configsTable = 'mod_orderformbuilder_configs';
        $configsSchema = "
            CREATE TABLE `{$configsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `form_id` INT NOT NULL,
                `product_id` INT NOT NULL,
                `position` INT DEFAULT 0,
                `is_featured` TINYINT(1) DEFAULT 0,
                `is_hidden` TINYINT(1) DEFAULT 0,
                `custom_pricing` JSON NULL,
                `custom_features` JSON NULL,
                `min_quantity` INT DEFAULT 1,
                `max_quantity` INT DEFAULT 99,
                `config_options` JSON NULL,
                INDEX `idx_form_product` (`form_id`, `product_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($configsTable, $configsSchema);
        
        return array(
            'status' => 'success',
            'description' => 'Order Form Builder module activated.',
        );
    } catch (\Exception $e) {
        return array(
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        );
    }
}

function orderformbuilder_deactivate()
{
    return array('status' => 'success', 'description' => 'Module deactivated.');
}

function orderformbuilder_upgrade($vars)
{
    $version = $vars['version'];
    
    if ($version < '1.1.0') {
        try {
            full_query("ALTER TABLE `mod_orderformbuilder_forms` 
                        ADD COLUMN `template` VARCHAR(50) DEFAULT 'default' AFTER `config`");
        } catch (\Exception $e) {
            logActivity("Order Form Builder upgrade error: " . $e->getMessage());
        }
    }
}

/**
 * Create a new order form
 */
function orderformbuilder_CreateForm($formData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $formKey = preg_replace('/[^a-zA-Z0-9_-]/', '', strtolower($formData['name']));
        $formKey .= '-' . substr(md5(uniqid()), 0, 6);
        
        $data = array(
            'form_key' => $formKey,
            'name' => $formData['name'],
            'description' => $formData['description'] ?? '',
            'config' => json_encode($formData['config'] ?? array()),
            'products' => json_encode($formData['products'] ?? array()),
            'categories' => json_encode($formData['categories'] ?? array()),
            'steps' => json_encode($formData['steps'] ?? orderformbuilder_GetDefaultSteps()),
            'template' => $formData['template'] ?? 'default',
            'created_by' => $formData['created_by'] ?? 0,
        );
        
        Capsule::table('mod_orderformbuilder_forms')->insert($data);
        
        return array('success' => true, 'form_key' => $formKey);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get default checkout steps
 */
function orderformbuilder_GetDefaultSteps()
{
    return array(
        array(
            'id' => 'products',
            'name' => 'Select Products',
            'description' => 'Choose your products and services',
            'required' => true,
        ),
        array(
            'id' => 'configure',
            'name' => 'Configure',
            'description' => 'Configure your selections',
            'required' => true,
        ),
        array(
            'id' => 'checkout',
            'name' => 'Checkout',
            'description' => 'Review and checkout',
            'required' => true,
        ),
        array(
            'id' => 'complete',
            'name' => 'Complete',
            'description' => 'Order complete',
            'required' => false,
        ),
    );
}

/**
 * Get all order forms
 */
function orderformbuilder_GetForms($activeOnly = true)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_orderformbuilder_forms');
    
    if ($activeOnly) {
        $query->where('is_active', 1);
    }
    
    $forms = $query->orderBy('name', 'asc')->get();
    
    foreach ($forms as &$form) {
        $form->config = json_decode($form->config, true);
        $form->products = json_decode($form->products, true);
        $form->categories = json_decode($form->categories, true);
        $form->steps = json_decode($form->steps, true);
    }
    
    return $forms;
}

/**
 * Get form by key
 */
function orderformbuilder_GetForm($formKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $form = Capsule::table('mod_orderformbuilder_forms')
        ->where('form_key', $formKey)
        ->first();
    
    if ($form) {
        $form->config = json_decode($form->config, true);
        $form->products = json_decode($form->products, true);
        $form->categories = json_decode($form->categories, true);
        $form->steps = json_decode($form->steps, true);
        
        // Get product configurations
        $form->product_configs = orderformbuilder_GetProductConfigs($form->id);
    }
    
    return $form;
}

/**
 * Get default form
 */
function orderformbuilder_GetDefaultForm()
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $form = Capsule::table('mod_orderformbuilder_forms')
        ->where('is_default', 1)
        ->where('is_active', 1)
        ->first();
    
    if ($form) {
        return orderformbuilder_GetForm($form->form_key);
    }
    
    return null;
}

/**
 * Set form as default
 */
function orderformbuilder_SetDefaultForm($formKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_orderformbuilder_forms')
            ->update(array('is_default' => 0));
        
        Capsule::table('mod_orderformbuilder_forms')
            ->where('form_key', $formKey)
            ->update(array('is_default' => 1));
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Update form
 */
function orderformbuilder_UpdateForm($formKey, $formData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $updateData = array();
        
        if (isset($formData['name'])) {
            $updateData['name'] = $formData['name'];
        }
        if (isset($formData['description'])) {
            $updateData['description'] = $formData['description'];
        }
        if (isset($formData['config'])) {
            $updateData['config'] = json_encode($formData['config']);
        }
        if (isset($formData['products'])) {
            $updateData['products'] = json_encode($formData['products']);
        }
        if (isset($formData['categories'])) {
            $updateData['categories'] = json_encode($formData['categories']);
        }
        if (isset($formData['steps'])) {
            $updateData['steps'] = json_encode($formData['steps']);
        }
        if (isset($formData['template'])) {
            $updateData['template'] = $formData['template'];
        }
        if (isset($formData['is_active'])) {
            $updateData['is_active'] = $formData['is_active'];
        }
        
        $updateData['updated_at'] = date('Y-m-d H:i:s');
        
        Capsule::table('mod_orderformbuilder_forms')
            ->where('form_key', $formKey)
            ->update($updateData);
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Delete form
 */
function orderformbuilder_DeleteForm($formKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $form = Capsule::table('mod_orderformbuilder_forms')
            ->where('form_key', $formKey)
            ->first();
        
        if (!$form) {
            return array('success' => false, 'error' => 'Form not found');
        }
        
        // Delete product configs
        Capsule::table('mod_orderformbuilder_configs')
            ->where('form_id', $form->id)
            ->delete();
        
        // Delete form
        Capsule::table('mod_orderformbuilder_forms')
            ->where('form_key', $formKey)
            ->delete();
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Add product to form
 */
function orderformbuilder_AddProduct($formKey, $productId, $options = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $form = Capsule::table('mod_orderformbuilder_forms')
            ->where('form_key', $formKey)
            ->first();
        
        if (!$form) {
            return array('success' => false, 'error' => 'Form not found');
        }
        
        // Check if already added
        $existing = Capsule::table('mod_orderformbuilder_configs')
            ->where('form_id', $form->id)
            ->where('product_id', $productId)
            ->first();
        
        if ($existing) {
            return array('success' => false, 'error' => 'Product already in form');
        }
        
        // Get max position
        $maxPos = Capsule::table('mod_orderformbuilder_configs')
            ->where('form_id', $form->id)
            ->max('position') ?? 0;
        
        $data = array(
            'form_id' => $form->id,
            'product_id' => $productId,
            'position' => $options['position'] ?? ($maxPos + 1),
            'is_featured' => $options['is_featured'] ?? 0,
            'is_hidden' => $options['is_hidden'] ?? 0,
            'custom_pricing' => isset($options['pricing']) ? json_encode($options['pricing']) : null,
            'custom_features' => isset($options['features']) ? json_encode($options['features']) : null,
            'min_quantity' => $options['min_quantity'] ?? 1,
            'max_quantity' => $options['max_quantity'] ?? 99,
            'config_options' => isset($options['config_options']) ? json_encode($options['config_options']) : null,
        );
        
        Capsule::table('mod_orderformbuilder_configs')->insert($data);
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Remove product from form
 */
function orderformbuilder_RemoveProduct($formKey, $productId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $form = Capsule::table('mod_orderformbuilder_forms')
            ->where('form_key', $formKey)
            ->first();
        
        if (!$form) {
            return array('success' => false, 'error' => 'Form not found');
        }
        
        Capsule::table('mod_orderformbuilder_configs')
            ->where('form_id', $form->id)
            ->where('product_id', $productId)
            ->delete();
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Update product configuration in form
 */
function orderformbuilder_UpdateProductConfig($formKey, $productId, $config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $form = Capsule::table('mod_orderformbuilder_forms')
            ->where('form_key', $formKey)
            ->first();
        
        if (!$form) {
            return array('success' => false, 'error' => 'Form not found');
        }
        
        $updateData = array();
        
        if (isset($config['position'])) {
            $updateData['position'] = $config['position'];
        }
        if (isset($config['is_featured'])) {
            $updateData['is_featured'] = $config['is_featured'];
        }
        if (isset($config['is_hidden'])) {
            $updateData['is_hidden'] = $config['is_hidden'];
        }
        if (isset($config['pricing'])) {
            $updateData['custom_pricing'] = json_encode($config['pricing']);
        }
        if (isset($config['features'])) {
            $updateData['custom_features'] = json_encode($config['features']);
        }
        if (isset($config['min_quantity'])) {
            $updateData['min_quantity'] = $config['min_quantity'];
        }
        if (isset($config['max_quantity'])) {
            $updateData['max_quantity'] = $config['max_quantity'];
        }
        if (isset($config['config_options'])) {
            $updateData['config_options'] = json_encode($config['config_options']);
        }
        
        Capsule::table('mod_orderformbuilder_configs')
            ->where('form_id', $form->id)
            ->where('product_id', $productId)
            ->update($updateData);
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get product configurations for form
 */
function orderformbuilder_GetProductConfigs($formId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $configs = Capsule::table('mod_orderformbuilder_configs')
        ->where('form_id', $formId)
        ->where('is_hidden', 0)
        ->orderBy('position', 'asc')
        ->get();
    
    foreach ($configs as &$config) {
        $config->custom_pricing = $config->custom_pricing ? json_decode($config->custom_pricing, true) : null;
        $config->custom_features = $config->custom_features ? json_decode($config->custom_features, true) : null;
        $config->config_options = $config->config_options ? json_decode($config->config_options, true) : null;
        
        // Get product details
        $product = Capsule::table('tblproducts')
            ->where('id', $config->product_id)
            ->first();
        
        if ($product) {
            $config->product = $product;
        }
    }
    
    return $configs;
}

/**
 * Create new order submission
 */
function orderformbuilder_CreateSubmission($formKey, $sessionId = null, $userId = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $form = Capsule::table('mod_orderformbuilder_forms')
            ->where('form_key', $formKey)
            ->first();
        
        if (!$form) {
            return array('success' => false, 'error' => 'Form not found');
        }
        
        $data = array(
            'form_id' => $form->id,
            'session_id' => $sessionId ?? session_id(),
            'user_id' => $userId,
            'cart_data' => json_encode(array()),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
            'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500),
        );
        
        Capsule::table('mod_orderformbuilder_submissions')->insert($data);
        
        return array('success' => true, 'submission_id' => Capsule::connection()->getPdo()->lastInsertId());
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Update submission cart data
 */
function orderformbuilder_UpdateSubmission($submissionId, $cartData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_orderformbuilder_submissions')
            ->where('id', $submissionId)
            ->update(array(
                'cart_data' => json_encode($cartData),
                'updated_at' => date('Y-m-d H:i:s'),
            ));
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Complete submission
 */
function orderformbuilder_CompleteSubmission($submissionId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_orderformbuilder_submissions')
            ->where('id', $submissionId)
            ->update(array(
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s'),
            ));
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get submission
 */
function orderformbuilder_GetSubmission($submissionId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $submission = Capsule::table('mod_orderformbuilder_submissions')
        ->where('id', $submissionId)
        ->first();
    
    if ($submission) {
        $submission->cart_data = json_decode($submission->cart_data, true);
        $submission->custom_fields = $submission->custom_fields ? json_decode($submission->custom_fields, true) : array();
    }
    
    return $submission;
}

/**
 * Get form statistics
 */
function orderformbuilder_GetFormStats($formKey = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_orderformbuilder_submissions');
    
    if ($formKey) {
        $form = Capsule::table('mod_orderformbuilder_forms')
            ->where('form_key', $formKey)
            ->first();
        
        if ($form) {
            $query->where('form_id', $form->id);
        }
    }
    
    $stats = array(
        'total' => (clone $query)->count(),
        'completed' => (clone $query)->where('status', 'completed')->count(),
        'in_progress' => (clone $query)->where('status', 'in_progress')->count(),
        'abandoned' => (clone $query)->where('status', 'abandoned')->count(),
    );
    
    $stats['conversion_rate'] = $stats['total'] > 0 
        ? round(($stats['completed'] / $stats['total']) * 100, 2) 
        : 0;
    
    return $stats;
}

/**
 * Render form HTML
 */
function orderformbuilder_RenderForm($formKey, $options = array())
{
    $form = orderformbuilder_GetForm($formKey);
    
    if (!$form) {
        return '<p>Form not found.</p>';
    }
    
    $html = '<div class="order-form-builder" data-form-key="' . htmlspecialchars($formKey) . '">';
    
    // Render steps navigation
    if (!empty($form->steps) && ($form->config['enable_multistep'] ?? false)) {
        $html .= orderformbuilder_RenderSteps($form->steps, $options['current_step'] ?? 1);
    }
    
    // Render form content based on step
    $step = $options['current_step'] ?? 1;
    $html .= orderformbuilder_RenderStepContent($form, $step, $options);
    
    $html .= '</div>';
    
    return $html;
}

/**
 * Render steps navigation
 */
function orderformbuilder_RenderSteps($steps, $currentStep)
{
    $html = '<div class="form-steps"><ul class="steps-navigation">';
    
    foreach ($steps as $index => $step) {
        $stepNum = $index + 1;
        $isActive = $stepNum == $currentStep;
        $isComplete = $stepNum < $currentStep;
        
        $html .= '<li class="step-item' . ($isActive ? ' active' : '') . ($isComplete ? ' complete' : '') . '">';
        $html .= '<span class="step-number">' . $stepNum . '</span>';
        $html .= '<span class="step-name">' . htmlspecialchars($step['name']) . '</span>';
        $html .= '</li>';
    }
    
    $html .= '</ul></div>';
    
    return $html;
}

/**
 * Render step content
 */
function orderformbuilder_RenderStepContent($form, $step, $options)
{
    switch ($step) {
        case 1:
            return orderformbuilder_RenderProductSelection($form, $options);
        case 2:
            return orderformbuilder_RenderConfiguration($form, $options);
        case 3:
            return orderformbuilder_RenderCheckout($form, $options);
        default:
            return '<p>Unknown step.</p>';
    }
}

/**
 * Render product selection step
 */
function orderformbuilder_RenderProductSelection($form, $options)
{
    $html = '<div class="step-content step-products">';
    $html .= '<h3>Select Products</h3>';
    $html .= '<div class="products-grid">';
    
    foreach ($form->product_configs as $config) {
        if (!$config->product) continue;
        
        $product = $config->product;
        $pricing = $config->custom_pricing ?? array();
        $price = $pricing['monthly'] ?? $product->monthly ?? 'Custom';
        
        $html .= '<div class="product-card' . ($config->is_featured ? ' featured' : '') . '">';
        $html .= '<div class="product-header">';
        $html .= '<h4>' . htmlspecialchars($product->name) . '</h4>';
        
        if ($config->is_featured) {
            $html .= '<span class="featured-badge">Featured</span>';
        }
        
        $html .= '</div>';
        $html .= '<div class="product-price">';
        $html .= '<span class="price-amount">' . ($price !== 'Custom' ? '$' . number_format($price, 2) : 'Custom') . '</span>';
        $html .= '<span class="price-period">/month</span>';
        $html .= '</div>';
        
        if ($config->custom_features) {
            $html .= '<ul class="product-features">';
            foreach ($config->custom_features as $feature) {
                $html .= '<li>' . htmlspecialchars($feature) . '</li>';
            }
            $html .= '</ul>';
        }
        
        $html .= '<button type="button" class="btn-add-product" data-product-id="' . $product->id . '">';
        $html .= 'Add to Cart</button>';
        $html .= '</div>';
    }
    
    $html .= '</div></div>';
    
    return $html;
}

/**
 * Render configuration step
 */
function orderformbuilder_RenderConfiguration($form, $options)
{
    $html = '<div class="step-content step-configure">';
    $html .= '<h3>Configure Your Selection</h3>';
    $html .= '<div class="configuration-form">';
    $html .= '<p>Configure your selected products here.</p>';
    $html .= '</div></div>';
    
    return $html;
}

/**
 * Render checkout step
 */
function orderformbuilder_RenderCheckout($form, $options)
{
    $html = '<div class="step-content step-checkout">';
    $html .= '<h3>Review and Checkout</h3>';
    $html .= '<div class="checkout-summary">';
    $html .= '<p>Review your order and proceed to checkout.</p>';
    $html .= '</div></div>';
    
    return $html;
}

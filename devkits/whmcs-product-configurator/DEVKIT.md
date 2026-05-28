# WHMCS Product Configurator Module

```php
<?php
/**
 * WHMCS Product Configurator Module
 * 
 * Provides a product configuration wizard with step-by-step setup,
 * option validation, and pricing calculations.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access prohibited");
}

function productconfigurator_MetaData()
{
    return array(
        'DisplayName' => 'Product Configurator',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    );
}

function productconfigurator_ConfigArray()
{
    return array(
        'FriendlyName' => array(
            'Type' => 'System',
            'Value' => 'Product Configurator',
        ),
        'EnableWizard' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable wizard-style configuration',
        ),
        'EnableLivePricing' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Show live price updates',
        ),
        'EnableImagePreview' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Show product images',
        ),
        'DefaultTheme' => array(
            'Type' => 'dropdown',
            'Options' => array(
                'default' => 'Default',
                'modern' => 'Modern',
                'minimal' => 'Minimal',
            ),
            'Default' => 'default',
            'Description' => 'Wizard theme',
        ),
    );
}

function productconfigurator_activate()
{
    try {
        if (!function_exists('createTable')) {
            require_once dirname(__FILE__) . '/../../includes/modulefunctions.php';
        }
        
        // Configurator templates table
        $templatesTable = 'mod_productconfigurator_templates';
        $templatesSchema = "
            CREATE TABLE `{$templatesTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `product_id` INT NOT NULL,
                `template_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `steps` JSON NOT NULL,
                `options` JSON NOT NULL,
                `pricing_rules` JSON NULL,
                `validation_rules` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `sort_order` INT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_product_id` (`product_id`),
                INDEX `idx_template_key` (`template_key`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($templatesTable, $templatesSchema);
        
        // Configuration sessions table
        $sessionsTable = 'mod_productconfigurator_sessions';
        $sessionsSchema = "
            CREATE TABLE `{$sessionsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `session_id` VARCHAR(255) NOT NULL,
                `template_id` INT NOT NULL,
                `product_id` INT NOT NULL,
                `user_id` INT NULL,
                `current_step` INT DEFAULT 1,
                `selections` JSON NOT NULL,
                `pricing` JSON NULL,
                `validation_errors` JSON NULL,
                `is_complete` TINYINT(1) DEFAULT 0,
                `ip_address` VARCHAR(45) NULL,
                `started_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `completed_at` DATETIME NULL,
                `expires_at` DATETIME NULL,
                INDEX `idx_session_id` (`session_id`),
                INDEX `idx_template_id` (`template_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($sessionsTable, $sessionsSchema);
        
        // Price modifiers table
        $pricingTable = 'mod_productconfigurator_pricing';
        $pricingSchema = "
            CREATE TABLE `{$pricingTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `template_id` INT NOT NULL,
                `option_key` VARCHAR(100) NOT NULL,
                `modifier_type` ENUM('fixed', 'percentage', 'multiplier') DEFAULT 'fixed',
                `modifier_value` DECIMAL(10,2) NOT NULL,
                `min_quantity` INT DEFAULT 1,
                `max_quantity` INT DEFAULT 1,
                INDEX `idx_template_option` (`template_id`, `option_key`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($pricingTable, $pricingSchema);
        
        return array(
            'status' => 'success',
            'description' => 'Product Configurator module activated.',
        );
    } catch (\Exception $e) {
        return array(
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        );
    }
}

function productconfigurator_deactivate()
{
    return array('status' => 'success', 'description' => 'Module deactivated.');
}

/**
 * Create configuration template
 */
function productconfigurator_CreateTemplate($productId, $templateData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $templateKey = 'tmpl-' . $productId . '-' . substr(md5(uniqid()), 0, 8);
        
        $data = array(
            'product_id' => $productId,
            'template_key' => $templateKey,
            'name' => $templateData['name'],
            'steps' => json_encode($templateData['steps'] ?? array()),
            'options' => json_encode($templateData['options'] ?? array()),
            'pricing_rules' => isset($templateData['pricing_rules']) ? json_encode($templateData['pricing_rules']) : null,
            'validation_rules' => isset($templateData['validation_rules']) ? json_encode($templateData['validation_rules']) : null,
            'sort_order' => $templateData['sort_order'] ?? 0,
        );
        
        Capsule::table('mod_productconfigurator_templates')->insert($data);
        
        return array('success' => true, 'template_key' => $templateKey);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get template by key
 */
function productconfigurator_GetTemplate($templateKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $template = Capsule::table('mod_productconfigurator_templates')
        ->where('template_key', $templateKey)
        ->where('is_active', 1)
        ->first();
    
    if ($template) {
        $template->steps = json_decode($template->steps, true);
        $template->options = json_decode($template->options, true);
        $template->pricing_rules = json_decode($template->pricing_rules, true);
        $template->validation_rules = json_decode($template->validation_rules, true);
    }
    
    return $template;
}

/**
 * Get template for product
 */
function productconfigurator_GetTemplateForProduct($productId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $template = Capsule::table('mod_productconfigurator_templates')
        ->where('product_id', $productId)
        ->where('is_active', 1)
        ->orderBy('sort_order', 'asc')
        ->first();
    
    if ($template) {
        return productconfigurator_GetTemplate($template->template_key);
    }
    
    return null;
}

/**
 * Get all templates
 */
function productconfigurator_GetTemplates($productId = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_productconfigurator_templates')
        ->where('is_active', 1);
    
    if ($productId) {
        $query->where('product_id', $productId);
    }
    
    $templates = $query->orderBy('sort_order', 'asc')->get();
    
    foreach ($templates as &$template) {
        $template->steps = json_decode($template->steps, true);
        $template->options = json_decode($template->options, true);
    }
    
    return $templates;
}

/**
 * Update template
 */
function productconfigurator_UpdateTemplate($templateKey, $templateData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $updateData = array();
        
        if (isset($templateData['name'])) {
            $updateData['name'] = $templateData['name'];
        }
        if (isset($templateData['steps'])) {
            $updateData['steps'] = json_encode($templateData['steps']);
        }
        if (isset($templateData['options'])) {
            $updateData['options'] = json_encode($templateData['options']);
        }
        if (isset($templateData['pricing_rules'])) {
            $updateData['pricing_rules'] = json_encode($templateData['pricing_rules']);
        }
        if (isset($templateData['validation_rules'])) {
            $updateData['validation_rules'] = json_encode($templateData['validation_rules']);
        }
        if (isset($templateData['is_active'])) {
            $updateData['is_active'] = $templateData['is_active'];
        }
        if (isset($templateData['sort_order'])) {
            $updateData['sort_order'] = $templateData['sort_order'];
        }
        
        $updateData['updated_at'] = date('Y-m-d H:i:s');
        
        Capsule::table('mod_productconfigurator_templates')
            ->where('template_key', $templateKey)
            ->update($updateData);
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Delete template
 */
function productconfigurator_DeleteTemplate($templateKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_productconfigurator_templates')
            ->where('template_key', $templateKey)
            ->update(array('is_active' => 0));
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Start configuration session
 */
function productconfigurator_StartSession($templateKey, $productId, $userId = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $template = productconfigurator_GetTemplate($templateKey);
        
        if (!$template) {
            return array('success' => false, 'error' => 'Template not found');
        }
        
        $sessionId = 'cfg-' . session_id() . '-' . substr(md5(uniqid()), 0, 8);
        $expiresAt = date('Y-m-d H:i:s', strtotime('+2 hours'));
        
        $data = array(
            'session_id' => $sessionId,
            'template_id' => $template->id,
            'product_id' => $productId,
            'user_id' => $userId,
            'current_step' => 1,
            'selections' => json_encode(array()),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
            'expires_at' => $expiresAt,
        );
        
        Capsule::table('mod_productconfigurator_sessions')->insert($data);
        
        return array(
            'success' => true,
            'session_id' => $sessionId,
            'expires_at' => $expiresAt,
        );
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get configuration session
 */
function productconfigurator_GetSession($sessionId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $session = Capsule::table('mod_productconfigurator_sessions')
        ->where('session_id', $sessionId)
        ->first();
    
    if ($session) {
        $session->selections = json_decode($session->selections, true);
        $session->pricing = $session->pricing ? json_decode($session->pricing, true) : array();
        $session->validation_errors = $session->validation_errors ? json_decode($session->validation_errors, true) : array();
        
        // Get template
        $session->template = productconfigurator_GetTemplateById($session->template_id);
    }
    
    return $session;
}

/**
 * Get template by ID
 */
function productconfigurator_GetTemplateById($templateId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $template = Capsule::table('mod_productconfigurator_templates')
        ->where('id', $templateId)
        ->first();
    
    if ($template) {
        $template->steps = json_decode($template->steps, true);
        $template->options = json_decode($template->options, true);
        $template->pricing_rules = json_decode($template->pricing_rules, true);
    }
    
    return $template;
}

/**
 * Update configuration selection
 */
function productconfigurator_UpdateSelection($sessionId, $stepKey, $selections)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $session = productconfigurator_GetSession($sessionId);
        
        if (!$session) {
            return array('success' => false, 'error' => 'Session not found');
        }
        
        // Update selections
        $allSelections = $session->selections;
        $allSelections[$stepKey] = $selections;
        
        // Validate selections
        $errors = productconfigurator_ValidateSelections($session->template, $allSelections);
        
        // Calculate pricing
        $pricing = productconfigurator_CalculatePricing($session->template, $allSelections, $session->product_id);
        
        // Update session
        Capsule::table('mod_productconfigurator_sessions')
            ->where('session_id', $sessionId)
            ->update(array(
                'selections' => json_encode($allSelections),
                'pricing' => json_encode($pricing),
                'validation_errors' => json_encode($errors),
            ));
        
        return array(
            'success' => true,
            'selections' => $allSelections,
            'pricing' => $pricing,
            'errors' => $errors,
        );
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Validate selections
 */
function productconfigurator_ValidateSelections($template, $selections)
{
    $errors = array();
    
    if (!$template || !$template->validation_rules) {
        return $errors;
    }
    
    foreach ($template->validation_rules as $rule) {
        $step = $rule['step'];
        $required = $rule['required'] ?? false;
        
        if ($required && empty($selections[$step])) {
            $errors[$step] = 'This selection is required';
        }
    }
    
    return $errors;
}

/**
 * Calculate pricing
 */
function productconfigurator_CalculatePricing($template, $selections, $productId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    // Get base product price
    $product = Capsule::table('tblproducts')
        ->where('id', $productId)
        ->first();
    
    $basePrice = $product ? ($product->monthly ?? $product->setup ?? 0) : 0;
    
    $pricing = array(
        'base_price' => $basePrice,
        'addons' => 0,
        'modifiers' => 0,
        'total' => $basePrice,
    );
    
    // Apply modifiers from selections
    foreach ($selections as $stepKey => $stepSelections) {
        if (is_array($stepSelections)) {
            foreach ($stepSelections as $optionKey => $value) {
                $modifier = productconfigurator_GetPriceModifier($template->id, $optionKey, $value);
                
                if ($modifier) {
                    $modifierAmount = productconfigurator_ApplyModifier($modifier, $basePrice);
                    $pricing['modifiers'] += $modifierAmount;
                }
            }
        }
    }
    
    $pricing['total'] = $basePrice + $pricing['addons'] + $pricing['modifiers'];
    
    return $pricing;
}

/**
 * Get price modifier
 */
function productconfigurator_GetPriceModifier($templateId, $optionKey, $value)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    return Capsule::table('mod_productconfigurator_pricing')
        ->where('template_id', $templateId)
        ->where('option_key', $optionKey)
        ->first();
}

/**
 * Apply price modifier
 */
function productconfigurator_ApplyModifier($modifier, $basePrice)
{
    switch ($modifier->modifier_type) {
        case 'fixed':
            return (float) $modifier->modifier_value;
        case 'percentage':
            return $basePrice * ($modifier->modifier_value / 100);
        case 'multiplier':
            return $basePrice * ($modifier->modifier_value - 1);
        default:
            return 0;
    }
}

/**
 * Add price modifier
 */
function productconfigurator_AddPriceModifier($templateKey, $optionKey, $modifierType, $modifierValue, $minQty = 1, $maxQty = 1)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $template = Capsule::table('mod_productconfigurator_templates')
            ->where('template_key', $templateKey)
            ->first();
        
        if (!$template) {
            return array('success' => false, 'error' => 'Template not found');
        }
        
        Capsule::table('mod_productconfigurator_pricing')->insert(array(
            'template_id' => $template->id,
            'option_key' => $optionKey,
            'modifier_type' => $modifierType,
            'modifier_value' => $modifierValue,
            'min_quantity' => $minQty,
            'max_quantity' => $maxQty,
        ));
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Move to next step
 */
function productconfigurator_NextStep($sessionId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $session = productconfigurator_GetSession($sessionId);
        
        if (!$session) {
            return array('success' => false, 'error' => 'Session not found');
        }
        
        // Check if current step is valid
        if (!empty($session->validation_errors)) {
            return array(
                'success' => false,
                'error' => 'Please complete the current step before proceeding',
            );
        }
        
        $steps = $session->template->steps ?? array();
        $currentStep = $session->current_step;
        
        if ($currentStep >= count($steps)) {
            return array('success' => false, 'error' => 'Already at final step');
        }
        
        Capsule::table('mod_productconfigurator_sessions')
            ->where('session_id', $sessionId)
            ->update(array('current_step' => $currentStep + 1));
        
        return array('success' => true, 'current_step' => $currentStep + 1);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Move to previous step
 */
function productconfigurator_PreviousStep($sessionId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $session = productconfigurator_GetSession($sessionId);
        
        if (!$session) {
            return array('success' => false, 'error' => 'Session not found');
        }
        
        $currentStep = $session->current_step;
        
        if ($currentStep <= 1) {
            return array('success' => false, 'error' => 'Already at first step');
        }
        
        Capsule::table('mod_productconfigurator_sessions')
            ->where('session_id', $sessionId)
            ->update(array('current_step' => $currentStep - 1));
        
        return array('success' => true, 'current_step' => $currentStep - 1);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Complete configuration
 */
function productconfigurator_CompleteSession($sessionId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $session = productconfigurator_GetSession($sessionId);
        
        if (!$session) {
            return array('success' => false, 'error' => 'Session not found');
        }
        
        if (!empty($session->validation_errors)) {
            return array('success' => false, 'error' => 'Please complete all required selections');
        }
        
        Capsule::table('mod_productconfigurator_sessions')
            ->where('session_id', $sessionId)
            ->update(array(
                'is_complete' => 1,
                'completed_at' => date('Y-m-d H:i:s'),
            ));
        
        return array(
            'success' => true,
            'selections' => $session->selections,
            'pricing' => $session->pricing,
        );
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Render wizard HTML
 */
function productconfigurator_RenderWizard($sessionId)
{
    $session = productconfigurator_GetSession($sessionId);
    
    if (!$session) {
        return '<p>Configuration session not found.</p>';
    }
    
    $template = $session->template;
    $steps = $template->steps ?? array();
    $currentStepIndex = $session->current_step - 1;
    $currentStep = $steps[$currentStepIndex] ?? null;
    
    $html = '<div class="product-configurator" data-session="' . htmlspecialchars($sessionId) . '">';
    
    // Progress bar
    $html .= '<div class="config-progress">';
    $html .= '<div class="progress-bar" style="width: ' . (($session->current_step / count($steps)) * 100) . '%"></div>';
    $html .= '</div>';
    
    // Steps navigation
    $html .= '<div class="steps-nav"><ul>';
    foreach ($steps as $index => $step) {
        $stepNum = $index + 1;
        $isActive = $stepNum == $session->current_step;
        $isComplete = $stepNum < $session->current_step;
        
        $html .= '<li class="' . ($isActive ? 'active' : '') . ($isComplete ? 'complete' : '') . '">';
        $html .= '<span class="step-num">' . $stepNum . '</span>';
        $html .= '<span class="step-name">' . htmlspecialchars($step['name']) . '</span>';
        $html .= '</li>';
    }
    $html .= '</ul></div>';
    
    // Current step content
    if ($currentStep) {
        $html .= '<div class="step-content">';
        $html .= '<h3>' . htmlspecialchars($currentStep['name']) . '</h3>';
        $html .= '<p>' . htmlspecialchars($currentStep['description'] ?? '') . '</p>';
        $html .= productconfigurator_RenderStepOptions($template, $currentStep, $session);
        $html .= '</div>';
    }
    
    // Navigation buttons
    $html .= '<div class="step-navigation">';
    if ($session->current_step > 1) {
        $html .= '<button type="button" class="btn-prev" data-action="prev">Previous</button>';
    }
    if ($session->current_step < count($steps)) {
        $html .= '<button type="button" class="btn-next" data-action="next">Next</button>';
    } else {
        $html .= '<button type="button" class="btn-complete" data-action="complete">Complete</button>';
    }
    $html .= '</div>';
    
    // Pricing summary
    if (!empty($session->pricing)) {
        $html .= '<div class="pricing-summary">';
        $html .= '<h4>Pricing Summary</h4>';
        $html .= '<div class="price-line"><span>Base Price:</span><span>$' . number_format($session->pricing['base_price'], 2) . '</span></div>';
        if ($session->pricing['modifiers'] != 0) {
            $html .= '<div class="price-line"><span>Options:</span><span>$' . number_format($session->pricing['modifiers'], 2) . '</span></div>';
        }
        $html .= '<div class="price-line total"><span>Total:</span><span>$' . number_format($session->pricing['total'], 2) . '</span></div>';
        $html .= '</div>';
    }
    
    $html .= '</div>';
    
    return $html;
}

/**
 * Render step options
 */
function productconfigurator_RenderStepOptions($template, $step, $session)
{
    $stepKey = $step['key'] ?? '';
    $options = $template->options[$stepKey] ?? array();
    $selections = $session->selections[$stepKey] ?? array();
    
    $html = '<div class="step-options">';
    
    foreach ($options as $option) {
        $optionKey = $option['key'];
        $isSelected = in_array($optionKey, (array)$selections) || $selections[$optionKey] ?? false;
        
        $html .= '<div class="option-item' . ($isSelected ? ' selected' : '') . '">';
        $html .= '<input type="' . ($option['multi_select'] ?? false ? 'checkbox' : 'radio') . '"';
        $html .= ' name="options[' . htmlspecialchars($stepKey) . ']"';
        $html .= ' value="' . htmlspecialchars($optionKey) . '"';
        $html .= ' id="opt-' . htmlspecialchars($optionKey) . '"';
        $html .= ($isSelected ? ' checked' : '') . '>';
        $html .= '<label for="opt-' . htmlspecialchars($optionKey) . '">';
        $html .= '<span class="option-name">' . htmlspecialchars($option['name']) . '</span>';
        if (isset($option['price'])) {
            $html .= '<span class="option-price">+$' . number_format($option['price'], 2) . '</span>';
        }
        $html .= '</label>';
        if (isset($option['description'])) {
            $html .= '<p class="option-desc">' . htmlspecialchars($option['description']) . '</p>';
        }
        $html .= '</div>';
    }
    
    $html .= '</div>';
    
    return $html;
}

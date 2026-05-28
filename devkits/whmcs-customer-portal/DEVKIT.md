# WHMCS Custom Customer Portal Module

```php
<?php
/**
 * WHMCS Custom Customer Portal Module
 * 
 * Provides a custom client portal dashboard with enhanced features,
 * customizable widgets, and improved user experience.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access prohibited");
}

/**
 * Module meta data
 */
function customerportal_MetaData()
{
    return array(
        'DisplayName' => 'Custom Customer Portal',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    );
}

/**
 * Module configuration
 */
function customerportal_ConfigArray()
{
    return array(
        'FriendlyName' => array(
            'Type' => 'System',
            'Value' => 'Custom Customer Portal',
        ),
        'EnableDashboard' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable custom dashboard view',
        ),
        'DashboardTheme' => array(
            'Type' => 'dropdown',
            'Options' => array(
                'default' => 'Default',
                'dark' => 'Dark Mode',
                'light' => 'Light Mode',
                'custom' => 'Custom Theme',
            ),
            'Default' => 'default',
            'Description' => 'Portal theme',
        ),
        'EnableWidgets' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable customizable widgets',
        ),
        'EnableQuickActions' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Show quick action buttons',
        ),
        'EnableAnnouncements' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Show announcements widget',
        ),
        'RecentActivityLimit' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '10',
            'Description' => 'Number of recent activities to show',
        ),
        'EnableTickets' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable ticket widget',
        ),
        'EnableInvoices' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable invoices widget',
        ),
        'CustomCSS' => array(
            'Type' => 'textarea',
            'Rows' => '5',
            'Default' => '',
            'Description' => 'Custom CSS for the portal',
        ),
    );
}

/**
 * Activate module
 */
function customerportal_activate()
{
    try {
        if (!function_exists('createTable')) {
            require_once dirname(__FILE__) . '/../../includes/modulefunctions.php';
        }
        
        // Widget preferences table
        $widgetPrefsTable = 'mod_customerportal_widget_prefs';
        $widgetPrefsSchema = "
            CREATE TABLE `{$widgetPrefsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `widget_id` VARCHAR(100) NOT NULL,
                `position` INT DEFAULT 0,
                `is_visible` TINYINT(1) DEFAULT 1,
                `config` TEXT NULL,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_user_widget` (`user_id`, `widget_id`),
                INDEX `idx_user_id` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($widgetPrefsTable, $widgetPrefsSchema);
        
        // Portal settings table
        $settingsTable = 'mod_customerportal_settings';
        $settingsSchema = "
            CREATE TABLE `{$settingsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL UNIQUE,
                `theme` VARCHAR(50) DEFAULT 'default',
                `language` VARCHAR(20) DEFAULT 'english',
                `timezone` VARCHAR(100) DEFAULT 'UTC',
                `date_format` VARCHAR(50) DEFAULT 'Y-m-d',
                `notifications` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_user_id` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($settingsTable, $settingsSchema);
        
        // Quick links table
        $linksTable = 'mod_customerportal_quicklinks';
        $linksSchema = "
            CREATE TABLE `{$linksTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `title` VARCHAR(255) NOT NULL,
                `url` VARCHAR(500) NOT NULL,
                `icon` VARCHAR(100) DEFAULT 'fa-link',
                `category` VARCHAR(100) DEFAULT 'general',
                `sort_order` INT DEFAULT 0,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user_id` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($linksTable, $linksSchema);
        
        return array(
            'status' => 'success',
            'description' => 'Customer Portal module activated successfully.',
        );
    } catch (\Exception $e) {
        return array(
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        );
    }
}

/**
 * Deactivate module
 */
function customerportal_deactivate()
{
    return array(
        'status' => 'success',
        'description' => 'Module deactivated. User data preserved.',
    );
}

/**
 * Get available widgets
 * 
 * @return array Widget definitions
 */
function customerportal_GetWidgets()
{
    return array(
        'overview' => array(
            'id' => 'overview',
            'name' => 'Account Overview',
            'description' => 'Displays account summary and key information',
            'icon' => 'fa-home',
            'size' => 'large',
            'default_visible' => true,
        ),
        'services' => array(
            'id' => 'services',
            'name' => 'My Services',
            'description' => 'Shows active services and products',
            'icon' => 'fa-server',
            'size' => 'medium',
            'default_visible' => true,
        ),
        'invoices' => array(
            'id' => 'invoices',
            'name' => 'Recent Invoices',
            'description' => 'Displays recent invoices and payments',
            'icon' => 'fa-file-invoice-dollar',
            'size' => 'medium',
            'default_visible' => true,
        ),
        'tickets' => array(
            'id' => 'tickets',
            'name' => 'Support Tickets',
            'description' => 'Shows open support tickets',
            'icon' => 'fa-life-ring',
            'size' => 'medium',
            'default_visible' => true,
        ),
        'domains' => array(
            'id' => 'domains',
            'name' => 'My Domains',
            'description' => 'Lists registered domains',
            'icon' => 'fa-globe',
            'size' => 'medium',
            'default_visible' => false,
        ),
        'billing' => array(
            'id' => 'billing',
            'name' => 'Billing Summary',
            'description' => 'Shows billing overview and payment methods',
            'icon' => 'fa-credit-card',
            'size' => 'small',
            'default_visible' => true,
        ),
        'usage' => array(
            'id' => 'usage',
            'name' => 'Resource Usage',
            'description' => 'Displays bandwidth and storage usage',
            'icon' => 'fa-chart-bar',
            'size' => 'medium',
            'default_visible' => false,
        ),
        'announcements' => array(
            'id' => 'announcements',
            'name' => 'Announcements',
            'description' => 'Latest news and updates',
            'icon' => 'fa-bullhorn',
            'size' => 'small',
            'default_visible' => true,
        ),
        'activity' => array(
            'id' => 'activity',
            'name' => 'Recent Activity',
            'description' => 'Log of recent account activities',
            'icon' => 'fa-history',
            'size' => 'medium',
            'default_visible' => true,
        ),
        'quickactions' => array(
            'id' => 'quickactions',
            'name' => 'Quick Actions',
            'description' => 'Common actions and shortcuts',
            'icon' => 'fa-bolt',
            'size' => 'small',
            'default_visible' => true,
        ),
        'affiliate' => array(
            'id' => 'affiliate',
            'name' => 'Affiliate Program',
            'description' => 'Affiliate stats and referral link',
            'icon' => 'fa-hand-holding-usd',
            'size' => 'small',
            'default_visible' => false,
        ),
        'security' => array(
            'id' => 'security',
            'name' => 'Security Status',
            'description' => 'Account security overview',
            'icon' => 'fa-shield-alt',
            'size' => 'small',
            'default_visible' => false,
        ),
    );
}

/**
 * Get dashboard data for a user
 * 
 * @param int $userId User ID
 * @return array Dashboard data
 */
function customerportal_GetDashboardData($userId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $data = array(
        'user' => customerportal_GetUserInfo($userId),
        'widgets' => customerportal_GetUserWidgets($userId),
        'settings' => customerportal_GetUserSettings($userId),
        'quick_actions' => customerportal_GetQuickActions($userId),
    );
    
    return $data;
}

/**
 * Get user information
 * 
 * @param int $userId User ID
 * @return array User data
 */
function customerportal_GetUserInfo($userId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $user = Capsule::table('tblclients')
        ->where('id', $userId)
        ->first();
    
    if (!$user) {
        return null;
    }
    
    // Get additional stats
    $services = Capsule::table('tblhosting')
        ->where('userid', $userId)
        ->whereIn('domainstatus', array('Active', 'Suspended'))
        ->count();
    
    $domains = Capsule::table('tbldomains')
        ->where('userid', $userId)
        ->count();
    
    $openTickets = Capsule::table('tbltickets')
        ->where('userid', $userId)
        ->whereNotIn('status', array('Closed', 'Resolved'))
        ->count();
    
    $pendingInvoices = Capsule::table('tblinvoices')
        ->where('userid', $userId)
        ->whereIn('status', array('Unpaid', 'Overdue'))
        ->selectRaw('SUM(total) as total')
        ->first();
    
    return array(
        'id' => $user->id,
        'name' => $user->firstname . ' ' . $user->lastname,
        'email' => $user->email,
        'client_since' => $user->datecreated,
        'stats' => array(
            'services' => $services,
            'domains' => $domains,
            'open_tickets' => $openTickets,
            'pending_amount' => (float) ($pendingInvoices->total ?? 0),
        ),
    );
}

/**
 * Get user's configured widgets
 * 
 * @param int $userId User ID
 * @return array Widget configurations
 */
function customerportal_GetUserWidgets($userId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $widgets = customerportal_GetWidgets();
    $prefs = Capsule::table('mod_customerportal_widget_prefs')
        ->where('user_id', $userId)
        ->get();
    
    // Build widget preferences map
    $prefsMap = array();
    foreach ($prefs as $pref) {
        $prefsMap[$pref->widget_id] = array(
            'position' => $pref->position,
            'is_visible' => (bool) $pref->is_visible,
            'config' => $pref->config ? json_decode($pref->config, true) : array(),
        );
    }
    
    // Merge with defaults
    $result = array();
    foreach ($widgets as $widget) {
        $widgetId = $widget['id'];
        $result[$widgetId] = array_merge($widget, array(
            'position' => $prefsMap[$widgetId]['position'] ?? 0,
            'is_visible' => $prefsMap[$widgetId]['is_visible'] ?? $widget['default_visible'],
            'config' => $prefsMap[$widgetId]['config'] ?? array(),
        ));
    }
    
    // Sort by position
    uasort($result, function($a, $b) {
        return $a['position'] - $b['position'];
    });
    
    return $result;
}

/**
 * Save user widget preferences
 * 
 * @param int $userId User ID
 * @param array $widgets Widget configurations
 * @return bool Success
 */
function customerportal_SaveWidgetPreferences($userId, $widgets)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        foreach ($widgets as $widgetId => $config) {
            Capsule::table('mod_customerportal_widget_prefs')->updateOrInsert(
                array('user_id' => $userId, 'widget_id' => $widgetId),
                array(
                    'position' => $config['position'] ?? 0,
                    'is_visible' => $config['is_visible'] ?? 1,
                    'config' => isset($config['config']) ? json_encode($config['config']) : null,
                )
            );
        }
        
        return true;
    } catch (\Exception $e) {
        return false;
    }
}

/**
 * Get user settings
 * 
 * @param int $userId User ID
 * @return array Settings
 */
function customerportal_GetUserSettings($userId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $settings = Capsule::table('mod_customerportal_settings')
        ->where('user_id', $userId)
        ->first();
    
    if (!$settings) {
        return array(
            'theme' => 'default',
            'language' => 'english',
            'timezone' => 'UTC',
            'date_format' => 'Y-m-d',
            'notifications' => array(),
        );
    }
    
    return array(
        'theme' => $settings->theme,
        'language' => $settings->language,
        'timezone' => $settings->timezone,
        'date_format' => $settings->date_format,
        'notifications' => $settings->notifications ? json_decode($settings->notifications, true) : array(),
    );
}

/**
 * Save user settings
 * 
 * @param int $userId User ID
 * @param array $settings Settings to save
 * @return bool Success
 */
function customerportal_SaveUserSettings($userId, $settings)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $data = array(
            'theme' => $settings['theme'] ?? 'default',
            'language' => $settings['language'] ?? 'english',
            'timezone' => $settings['timezone'] ?? 'UTC',
            'date_format' => $settings['date_format'] ?? 'Y-m-d',
            'notifications' => isset($settings['notifications']) ? json_encode($settings['notifications']) : null,
        );
        
        Capsule::table('mod_customerportal_settings')->updateOrInsert(
            array('user_id' => $userId),
            $data
        );
        
        return true;
    } catch (\Exception $e) {
        return false;
    }
}

/**
 * Get quick actions for user
 * 
 * @param int $userId User ID
 * @return array Quick actions
 */
function customerportal_GetQuickActions($userId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    // Get user custom links
    $customLinks = Capsule::table('mod_customerportal_quicklinks')
        ->where('user_id', $userId)
        ->where('is_active', 1)
        ->orderBy('sort_order', 'asc')
        ->get();
    
    // Default quick actions
    $defaultActions = array(
        array(
            'id' => 'new_ticket',
            'title' => 'Open Ticket',
            'url' => 'submitticket.php',
            'icon' => 'fa-plus-circle',
            'category' => 'support',
        ),
        array(
            'id' => 'view_invoices',
            'title' => 'View Invoices',
            'url' => 'invoice.php',
            'icon' => 'fa-file-invoice-dollar',
            'category' => 'billing',
        ),
        array(
            'id' => 'update_profile',
            'title' => 'Update Profile',
            'url' => 'clientarea.php?action=details',
            'icon' => 'fa-user-edit',
            'category' => 'account',
        ),
        array(
            'id' => 'order_new',
            'title' => 'Order New Service',
            'url' => 'cart.php',
            'icon' => 'fa-shopping-cart',
            'category' => 'services',
        ),
        array(
            'id' => 'change_password',
            'title' => 'Change Password',
            'url' => 'clientarea.php?action=security',
            'icon' => 'fa-key',
            'category' => 'security',
        ),
    );
    
    // Merge with custom links
    foreach ($customLinks as $link) {
        $defaultActions[] = array(
            'id' => 'custom_' . $link->id,
            'title' => $link->title,
            'url' => $link->url,
            'icon' => $link->icon,
            'category' => $link->category,
            'is_custom' => true,
        );
    }
    
    return $defaultActions;
}

/**
 * Add custom quick link for user
 * 
 * @param int $userId User ID
 * @param array $link Link data
 * @return array Result
 */
function customerportal_AddQuickLink($userId, $link)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $data = array(
            'user_id' => $userId,
            'title' => $link['title'],
            'url' => $link['url'],
            'icon' => $link['icon'] ?? 'fa-link',
            'category' => $link['category'] ?? 'general',
            'sort_order' => $link['sort_order'] ?? 0,
        );
        
        Capsule::table('mod_customerportal_quicklinks')->insert($data);
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Remove quick link
 * 
 * @param int $linkId Link ID
 * @param int $userId User ID (for security)
 * @return bool Success
 */
function customerportal_RemoveQuickLink($linkId, $userId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_customerportal_quicklinks')
            ->where('id', $linkId)
            ->where('user_id', $userId)
            ->delete();
        
        return true;
    } catch (\Exception $e) {
        return false;
    }
}

/**
 * Get widget content data
 * 
 * @param int $userId User ID
 * @param string $widgetId Widget ID
 * @param array $config Widget configuration
 * @return array Widget content
 */
function customerportal_GetWidgetContent($userId, $widgetId, $config = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    switch ($widgetId) {
        case 'overview':
            return customerportal_GetOverviewData($userId);
        
        case 'services':
            return customerportal_GetServicesData($userId, $config);
        
        case 'invoices':
            return customerportal_GetInvoicesData($userId, $config);
        
        case 'tickets':
            return customerportal_GetTicketsData($userId, $config);
        
        case 'domains':
            return customerportal_GetDomainsData($userId, $config);
        
        case 'billing':
            return customerportal_GetBillingData($userId);
        
        case 'usage':
            return customerportal_GetUsageData($userId);
        
        case 'announcements':
            return customerportal_GetAnnouncementsData($config);
        
        case 'activity':
            return customerportal_GetActivityData($userId, $config);
        
        default:
            return array('error' => 'Unknown widget');
    }
}

/**
 * Get overview data
 */
function customerportal_GetOverviewData($userId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $user = customerportal_GetUserInfo($userId);
    
    // Get upcoming renewals
    $upcomingRenewals = Capsule::table('tblhosting')
        ->where('userid', $userId)
        ->where('domainstatus', 'Active')
        ->where('nextduedate', '>=', date('Y-m-d'))
        ->where('nextduedate', '<=', date('Y-m-d', strtotime('+7 days')))
        ->count();
    
    // Get recent activity
    $recentActivity = Capsule::table('tblactivitylog')
        ->where('userid', $userId)
        ->orderBy('id', 'desc')
        ->limit(5)
        ->get();
    
    return array(
        'stats' => $user['stats'],
        'upcoming_renewals' => $upcomingRenewals,
        'account_status' => 'Active',
        'recent_activity' => $recentActivity,
    );
}

/**
 * Get services data
 */
function customerportal_GetServicesData($userId, $config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $limit = $config['limit'] ?? 10;
    
    $services = Capsule::table('tblhosting')
        ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
        ->where('tblhosting.userid', $userId)
        ->select(array(
            'tblhosting.id',
            'tblhosting.domain',
            'tblhosting.domainstatus',
            'tblhosting.nextduedate',
            'tblhosting.firstpaymentamount',
            'tblproducts.name as product_name',
        ))
        ->orderBy('tblhosting.id', 'desc')
        ->limit($limit)
        ->get();
    
    return array('services' => $services);
}

/**
 * Get invoices data
 */
function customerportal_GetInvoicesData($userId, $config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $limit = $config['limit'] ?? 5;
    
    $invoices = Capsule::table('tblinvoices')
        ->where('userid', $userId)
        ->orderBy('id', 'desc')
        ->limit($limit)
        ->get();
    
    // Get totals
    $totals = Capsule::table('tblinvoices')
        ->where('userid', $userId)
        ->selectRaw("
            SUM(CASE WHEN status = 'Unpaid' THEN total ELSE 0 END) as unpaid_total,
            SUM(CASE WHEN status = 'Paid' THEN total ELSE 0 END) as paid_total
        ")
        ->first();
    
    return array(
        'invoices' => $invoices,
        'totals' => array(
            'unpaid' => (float) $totals->unpaid_total,
            'paid' => (float) $totals->paid_total,
        ),
    );
}

/**
 * Get tickets data
 */
function customerportal_GetTicketsData($userId, $config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $limit = $config['limit'] ?? 5;
    
    $tickets = Capsule::table('tbltickets')
        ->where('userid', $userId)
        ->orderBy('id', 'desc')
        ->limit($limit)
        ->get();
    
    // Count by status
    $statusCounts = Capsule::table('tbltickets')
        ->where('userid', $userId)
        ->selectRaw("
            COUNT(CASE WHEN status = 'Open' THEN 1 END) as open_count,
            COUNT(CASE WHEN status = 'Awaiting Reply' THEN 1 END) as awaiting_count,
            COUNT(CASE WHEN status = 'Closed' THEN 1 END) as closed_count
        ")
        ->first();
    
    return array(
        'tickets' => $tickets,
        'counts' => array(
            'open' => (int) $statusCounts->open_count,
            'awaiting' => (int) $statusCounts->awaiting_count,
            'closed' => (int) $statusCounts->closed_count,
        ),
    );
}

/**
 * Get domains data
 */
function customerportal_GetDomainsData($userId, $config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $limit = $config['limit'] ?? 10;
    
    $domains = Capsule::table('tbldomains')
        ->where('userid', $userId)
        ->orderBy('id', 'desc')
        ->limit($limit)
        ->get();
    
    return array('domains' => $domains);
}

/**
 * Get billing data
 */
function customerportal_GetBillingData($userId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    // Get payment methods
    $paymentMethods = Capsule::table('tblpaymentgateways')
        ->where('setting', 'name')
        ->where('gateway', '!=', '')
        ->groupBy('gateway')
        ->get();
    
    // Get default payment method
    $defaultPayment = Capsule::table('tblclients')
        ->where('id', $userId)
        ->select('defaultgateway')
        ->first();
    
    return array(
        'default_payment_method' => $defaultPayment->defaultgateway ?? null,
        'available_methods' => $paymentMethods,
    );
}

/**
 * Get usage data
 */
function customerportal_GetUsageData($userId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    // Get services with usage info
    $services = Capsule::table('tblhosting')
        ->where('userid', $userId)
        ->where('domainstatus', 'Active')
        ->select(array('id', 'domain', 'diskusage', 'disklimit', 'bwusage', 'bwlimit'))
        ->get();
    
    return array('services' => $services);
}

/**
 * Get announcements data
 */
function customerportal_GetAnnouncementsData($config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $limit = $config['limit'] ?? 3;
    
    $announcements = Capsule::table('tblannouncements')
        ->where('published', 'yes')
        ->orderBy('date', 'desc')
        ->limit($limit)
        ->get();
    
    return array('announcements' => $announcements);
}

/**
 * Get activity data
 */
function customerportal_GetActivityData($userId, $config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $limit = $config['limit'] ?? 10;
    
    $activity = Capsule::table('tblactivitylog')
        ->where('userid', $userId)
        ->orderBy('id', 'desc')
        ->limit($limit)
        ->get();
    
    return array('activity' => $activity);
}

/**
 * Generate portal HTML output
 * 
 * @param int $userId User ID
 * @param array $config Portal configuration
 * @return string HTML output
 */
function customerportal_RenderPortal($userId, $config = array())
{
    $widgets = customerportal_GetUserWidgets($userId);
    $data = customerportal_GetDashboardData($userId);
    
    $html = '<div class="customer-portal-dashboard">';
    $html .= '<div class="portal-header">';
    $html .= '<h2>Welcome back, ' . htmlspecialchars($data['user']['name']) . '</h2>';
    $html .= '</div>';
    $html .= '<div class="portal-widgets">';
    
    foreach ($widgets as $widget) {
        if (!$widget['is_visible']) {
            continue;
        }
        
        $content = customerportal_GetWidgetContent($userId, $widget['id'], $widget['config']);
        $html .= customerportal_RenderWidget($widget, $content);
    }
    
    $html .= '</div></div>';
    
    return $html;
}

/**
 * Render single widget
 * 
 * @param array $widget Widget configuration
 * @param array $content Widget content
 * @return string HTML
 */
function customerportal_RenderWidget($widget, $content)
{
    if (isset($content['error'])) {
        return '';
    }
    
    $html = '<div class="portal-widget portal-widget-' . htmlspecialchars($widget['id']) . ' portal-widget-' . htmlspecialchars($widget['size']) . '">';
    $html .= '<div class="widget-header">';
    $html .= '<i class="fas ' . htmlspecialchars($widget['icon']) . '"></i>';
    $html .= '<h3>' . htmlspecialchars($widget['name']) . '</h3>';
    $html .= '</div>';
    $html .= '<div class="widget-content">';
    $html .= customerportal_RenderWidgetContent($widget['id'], $content);
    $html .= '</div></div>';
    
    return $html;
}

/**
 * Render widget-specific content
 */
function customerportal_RenderWidgetContent($widgetId, $content)
{
    switch ($widgetId) {
        case 'overview':
            return customerportal_RenderOverviewWidget($content);
        case 'services':
            return customerportal_RenderServicesWidget($content);
        case 'invoices':
            return customerportal_RenderInvoicesWidget($content);
        case 'tickets':
            return customerportal_RenderTicketsWidget($content);
        default:
            return '<p>Widget content</p>';
    }
}

function customerportal_RenderOverviewWidget($content)
{
    $html = '<div class="overview-stats">';
    $html .= '<div class="stat-item"><span class="stat-value">' . $content['stats']['services'] . '</span><span class="stat-label">Services</span></div>';
    $html .= '<div class="stat-item"><span class="stat-value">' . $content['stats']['domains'] . '</span><span class="stat-label">Domains</span></div>';
    $html .= '<div class="stat-item"><span class="stat-value">' . $content['stats']['open_tickets'] . '</span><span class="stat-label">Open Tickets</span></div>';
    $html .= '</div>';
    return $html;
}

function customerportal_RenderServicesWidget($content)
{
    if (empty($content['services'])) {
        return '<p>No services found.</p>';
    }
    
    $html = '<ul class="services-list">';
    foreach ($content['services'] as $service) {
        $html .= '<li>';
        $html .= '<span class="service-name">' . htmlspecialchars($service->domain ?: $service->product_name) . '</span>';
        $html .= '<span class="service-status status-' . strtolower($service->domainstatus) . '">' . htmlspecialchars($service->domainstatus) . '</span>';
        $html .= '</li>';
    }
    $html .= '</ul>';
    
    return $html;
}

function customerportal_RenderInvoicesWidget($content)
{
    if (empty($content['invoices'])) {
        return '<p>No invoices found.</p>';
    }
    
    $html = '<ul class="invoices-list">';
    foreach ($content['invoices'] as $invoice) {
        $html .= '<li>';
        $html .= '<span class="invoice-id">#' . $invoice->id . '</span>';
        $html .= '<span class="invoice-amount">$' . number_format($invoice->total, 2) . '</span>';
        $html .= '<span class="invoice-status status-' . strtolower($invoice->status) . '">' . htmlspecialchars($invoice->status) . '</span>';
        $html .= '</li>';
    }
    $html .= '</ul>';
    
    return $html;
}

function customerportal_RenderTicketsWidget($content)
{
    if (empty($content['tickets'])) {
        return '<p>No tickets found.</p>';
    }
    
    $html = '<ul class="tickets-list">';
    foreach ($content['tickets'] as $ticket) {
        $html .= '<li>';
        $html .= '<span class="ticket-subject">' . htmlspecialchars($ticket->title) . '</span>';
        $html .= '<span class="ticket-status status-' . strtolower($ticket->status) . '">' . htmlspecialchars($ticket->status) . '</span>';
        $html .= '</li>';
    }
    $html .= '</ul>';
    
    return $html;
}

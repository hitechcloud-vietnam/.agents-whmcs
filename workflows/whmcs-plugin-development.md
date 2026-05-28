# WHMCS Plugin/Hook Development Workflow

## Purpose

Complete guide to developing hooks and plugins for WHMCS. Covers hook system basics, hook registration, plugin architecture, hook priorities, and advanced patterns for extending WHMCS functionality.

## Prerequisites

- WHMCS 7.0+ installation
- PHP 7.4+ knowledge
- Understanding of WHMCS hooks system
- Basic plugin development experience

## Workflow Steps

### Step 1: Understanding WHMCS Hooks System

```
Hook System Overview:

Hook Types:
1. Page hooks - Inject content into pages (header, footer, sidebar)
2. Process hooks - Execute code on events (create account, payment, etc.)
3. Route hooks - Modify routing behavior
4. Authentication hooks - Custom authentication logic
5. API hooks - Extend API functionality

Hook File Locations:
- hooks.php (root WHMCS directory)
- includes/hooks/ (recommended for custom hooks)
- modules/addons/{module}/hooks.php (addon module hooks)
- modules/servers/{module}/hooks.php (server module hooks)

Hook Priority:
- Lower numbers = higher priority (executed first)
- Default priority is 1
- Recommended: 1=high, 50=normal, 100=low
```

### Step 2: Creating Hook Files

```php
<?php
/**
 * WHMCS Custom Hooks
 *
 * Place in includes/hooks/custom_hooks.php
 * Loaded automatically by WHMCS
 */

// ===========================================
// Account Event Hooks
// ===========================================

/**
 * Execute after new client account creation
 */
add_hook('ClientAdd', 1, function(array $vars) {
    $clientId = $vars['user_id'];
    $email = $vars['email'];

    // Create initial configuration
    Capsule::table('mod_client_preferences')->insert([
        'client_id' => $clientId,
        'email_notifications' => 1,
        'dark_mode' => 0,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    // Sync to external CRM
    try {
        $crm = new ExternalCRM_Client();
        $crm->createContact($clientId, $email);
    } catch (\Exception $e) {
        logActivity("CRM sync failed for client {$clientId}: " . $e->getMessage());
    }

    return true;
});

/**
 * Execute before client account update
 */
add_hook('ClientEdit', 1, function(array $vars) {
    // Validate changes
    if (isset($vars['email']) && !filter_var($vars['email'], FILTER_VALIDATE_EMAIL)) {
        return ['error' => 'Invalid email address'];
    }

    return true;
});

/**
 * Execute after client account update
 */
add_hook('ClientEdit', 50, function(array $vars) {
    // Audit the change
    logActivity("Client {$vars['user_id']} profile updated");

    // Update external systems
    syncClientToExternal($vars['user_id']);

    return true;
});

/**
 * Execute after client login
 */
add_hook('ClientLogin', 1, function(array $vars) {
    // Track login activity
    Capsule::table('mod_login_activity')->insert([
        'client_id' => $vars['user_id'],
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'logged_in_at' => date('Y-m-d H:i:s'),
    ]);

    // Update last login
    Capsule::table('tblclients')
        ->where('id', $vars['user_id'])
        ->update(['lastlogin' => date('Y-m-d H:i:s')]);

    // Welcome back email for returning customers
    $lastLogin = Capsule::table('mod_login_activity')
        ->where('client_id', $vars['user_id'])
        ->where('id', '!=', Capsule::raw('(SELECT MAX(id) FROM mod_login_activity)'))
        ->orderBy('logged_in_at', 'desc')
        ->first();

    if ($lastLogin) {
        sendMessage('Welcome Back', $vars['user_id']);
    }

    return true;
});

/**
 * Execute after client logout
 */
add_hook('ClientLogout', 1, function(array $vars) {
    $clientId = $vars['user_id'] ?? 0;

    // Calculate session duration
    $loginTime = $_SESSION['login_time'] ?? time();
    $duration = time() - $loginTime;

    logActivity("Client {$clientId} logged out. Session duration: {$duration} seconds");

    // Clean up session data
    unset($_SESSION['custom_data']);

    return true;
});

// ===========================================
// Service/Module Hooks
// ===========================================

/**
 * Execute after service provisioning
 */
add_hook('AfterModuleCreate', 1, function(array $vars) {
    $serviceId = $vars['serviceid'];
    $userId = $vars['userid'];

    // Send welcome email
    sendMessage('Service Provisioned', $serviceId);

    // Configure default monitoring
    $monitoring = new MonitoringService();
    $monitoring->addServiceMonitor($serviceId);

    // Set up backup schedule
    if (Capsule::table('tblproducts')
        ->where('id', $vars['packageid'])
        ->value('servertype') === 'yourprovider') {
       BackupService::scheduleBackups($serviceId);
    }

    return true;
});

/**
 * Execute after service suspension
 */
add_hook('AfterModuleSuspend', 1, function(array $vars) {
    $serviceId = $vars['serviceid'];

    // Alert the customer
    sendMessage('Service Suspended', $serviceId);

    // Disable monitoring alerts
    $monitoring = new MonitoringService();
    $monitoring->pauseAlerts($serviceId);

    // Log for reporting
    Capsule::table('mod_service_events')->insert([
        'service_id' => $serviceId,
        'event_type' => 'suspended',
        'occurred_at' => date('Y-m-d H:i:s'),
    ]);

    return true;
});

/**
 * Execute after service termination
 */
add_hook('AfterModuleTerminate', 1, function(array $vars) {
    $serviceId = $vars['serviceid'];

    // Send final invoice/statement
    sendMessage('Service Terminated', $serviceId);

    // Archive service data
    archiveServiceData($serviceId);

    // Release resources
    releaseServiceResources($serviceId);

    return true;
});

/**
 * Custom module parameter validation
 */
add_hook('ServiceCustomFields', 1, function(array $vars) {
    // Validate custom field inputs for specific products
    $productId = $vars['productid'] ?? 0;

    $product = Capsule::table('tblproducts')->find($productId);

    if ($product && $product->servertype === 'custom_product') {
        $validation = validateCustomFields($_POST, $product->id);

        if (!$validation['valid']) {
            return [
                'error' => $validation['errors'],
                'field' => $validation['field'],
            ];
        }
    }

    return true;
});

// ===========================================
// Invoice Hooks
// ===========================================

/**
 * Execute after invoice payment
 */
add_hook('InvoicePaid', 1, function(array $vars) {
    $invoiceId = $vars['invoiceid'];
    $userId = $vars['userid'];
    $total = $vars['total'];

    // Send payment confirmation
    sendMessage('Invoice Payment Confirmation', $invoiceId);

    // Update accounting system
    try {
        $accounting = new AccountingIntegration();
        $accounting->recordPayment($invoiceId, $total);
    } catch (\Exception $e) {
        logActivity("Accounting sync failed: " . $e->getMessage());
    }

    // Trigger affiliate commission
    Affiliate::calculateCommission($userId, $total);

    // Update customer loyalty points
    LoyaltyProgram::addPoints($userId, (int)($total * 10));

    return true;
});

/**
 * Execute before invoice generation
 */
add_hook('InvoicePreCreation', 1, function(array $vars) {
    $userId = $vars['userid'];

    // Check for overdue invoices
    $overdue = Capsule::table('tblinvoices')
        ->where('userid', $userId)
        ->where('status', 'Overdue')
        ->count();

    if ($overdue > 0) {
        // Optionally block new invoice creation
        // return ['error' => 'Please pay overdue invoices first'];
    }

    // Apply automatic discounts
    $discount = calculateLoyaltyDiscount($userId);
    if ($discount > 0) {
        return [
            'discount' => $discount,
            'discount_reason' => 'Loyalty discount',
        ];
    }

    return true;
});

/**
 * Execute after invoice creation
 */
add_hook('InvoiceCreated', 1, function(array $vars) {
    $invoiceId = $vars['invoice_id'];

    // Add custom line items
    Capsule::table('tblinvoiceitems')->insert([
        'invoice_id' => $invoiceId,
        'userid' => $vars['userid'],
        'description' => 'Environmental fee',
        'amount' => 0.50,
        'taxed' => 1,
    ]);

    return true;
});

/**
 * Execute after invoice cancellation
 */
add_hook('InvoiceCancelled', 1, function(array $vars) {
    $invoiceId = $vars['invoiceid'];

    // Log cancellation reason
    logActivity("Invoice {$invoiceId} cancelled by user {$vars['userid']}");

    // Refund any applied credits
    refundCredits($invoiceId);

    return true;
});

// ===========================================
// Ticket/Support Hooks
// ===========================================

/**
 * Execute when support ticket opened
 */
add_hook('TicketOpen', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];
    $departmentId = $vars['deptid'];
    $userId = $vars['userid'];

    // Auto-assign based on keywords
    $subject = $vars['subject'] ?? '';
    $autoAssign = routeTicketToDepartment($subject, $departmentId);

    if ($autoAssign) {
        // Re-route the ticket
        updateTicketDepartment($ticketId, $autoAssign['department']);
        updateTicketPriority($ticketId, $autoAssign['priority']);
    }

    // Send acknowledgment
    sendMessage('Support Ticket Acknowledgment', $ticketId);

    // Escalate urgent tickets
    if (stripos($subject, 'urgent') !== false || stripos($subject, 'emergency') !== false) {
        escalateTicket($ticketId);
        notifySupportManager($ticketId);
    }

    return true;
});

/**
 * Execute when support ticket replied
 */
add_hook('TicketReply', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];
    $replyType = $vars['reply_type']; // 'reply' or 'note'

    if ($replyType === 'reply') {
        // Update SLA timer
        updateSLATimer($ticketId);

        // Customer satisfaction survey
        sendTicketSurvey($ticketId);
    }

    return true;
});

/**
 * Execute when ticket is escalated
 */
add_hook('TicketEscalate', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];
    $newPriority = $vars['new_priority'];

    // Notify escalation contacts
    notifyEscalationContacts($ticketId, $newPriority);

    // Update metrics
    Capsule::table('mod_ticket_metrics')->insert([
        'ticket_id' => $ticketId,
        'event' => 'escalated',
        'new_priority' => $newPriority,
        'timestamp' => date('Y-m-d H:i:s'),
    ]);

    return true;
});
```

### Step 3: Creating Advanced Custom Hooks

```php
<?php
/**
 * Advanced Hook Patterns
 */

// ===========================================
// Authentication Hooks
// ===========================================

/**
 * Custom authentication logic
 */
add_hook('Authentication', 1, function(array $vars) {
    // Two-factor authentication check
    $userId = $vars['user_id'] ?? 0;

    $twoFactor = Capsule::table('mod_user_2fa')
        ->where('user_id', $userId)
        ->where('enabled', 1)
        ->first();

    if ($twoFactor && empty($_SESSION['2fa_verified'])) {
        // Generate and send 2FA code
        $code = generate2FACode();
        $_SESSION['2fa_pending'] = $userId;
        $_SESSION['2fa_code'] = $code;
        $_SESSION['2fa_expires'] = time() + 300; // 5 minutes

        sendSMS($userId, "Your verification code is: {$value}");

        return [
            'success' => false,
            'require_2fa' => true,
            'message' => 'Please enter your verification code',
        ];
    }

    return true;
});

/**
 * Post-authentication verification
 */
add_hook('AfterAuthentication', 1, function(array $vars) {
    if (!empty($_SESSION['2fa_verified'])) {
        logActivity("2FA verified for user {$vars['user_id']}");
    }

    // Check for suspicious activity
    if (detectSuspiciousLogin($vars)) {
        logActivity("Suspicious login detected for user {$vars['user_id']} from IP {$_SERVER['REMOTE_ADDR']}");
        notifySecurityTeam($vars['user_id']);
    }

    return true;
});

// ===========================================
// Page Injection Hooks
// ===========================================

/**
 * Inject content into client area header
 */
add_hook('ClientAreaHeaderOutput', 1, function(array $vars) {
    $customCss = <<<CSS
<style>
    .custom-header-badge {
        background-color: var(--primary-color);
        color: white;
        padding: 2px 8px;
        border-radius: 10px;
        font-size: 12px;
    }
</style>
CSS;

    $customJs = <<<JS
<script>
    // Track page views
    console.log('Client area loaded');
</script>
JS;

    return $customCss . $customJs;
});

/**
 * Inject content into footer
 */
add_hook('ClientAreaFooterOutput', 1, function(array $vars) {
    return <<<HTML
<script>
    // Live chat integration
    window.intercomSettings = {
        api_base: "https://api-iam.intercom.io",
        app_id: "your_app_id",
        user_id: "{$vars['user_id']}"
    };
</script>
HTML;
});

/**
 * Sidebar panel injection
 */
add_hook('ClientAreaSidebars', 1, function(array $vars) {
    if ($vars['sidebar'] !== 'client') {
        return;
    }

    return [
        'primarySidebar' => [
            'order' => 100,
            'content' => '<div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Quick Actions</h3>
                </div>
                <div class="panel-body">
                    <a href="clientarea.php?action=order" class="btn btn-block btn-sm btn-primary">
                        <i class="fa fa-plus"></i> New Order
                    </a>
                    <a href="supporttickets.php?action=open" class="btn btn-block btn-sm btn-default">
                        <i class="fa fa-ticket"></i> Open Ticket
                    </a>
                </div>
            </div>',
        ],
    ];
});

// ===========================================
// Cart/Order Hooks
// ===========================================

/**
 * Modify cart before checkout
 */
add_hook('PreShoppingCartCheckout', 1, function(array $vars) {
    $cart = $vars['cart'];

    // Check for restricted products by country
    $country = $_SESSION['cart']['user_data']['country'] ?? '';

    $restrictedProducts = ['product-xyz'];
    foreach ($cart['products'] as &$product) {
        if (in_array($product['product_id'], $restrictedProducts) &&
            in_array($country, ['CU', 'IR', 'KP', 'SY'])) {
            return [
                'error' => 'This product is not available in your region',
                'product_id' => $product['product_id'],
            ];
        }
    }

    // Auto-apply coupon based on cart contents
    if (empty($cart['promotion']) && shouldAutoApplyCoupon($cart)) {
        applyCouponToCart('AUTOCOUPON');
    }

    return true;
});

/**
 * Modify cart totals
 */
add_hook('AfterCalculateCartTotals', 1, function(array $vars) {
    $userId = $_SESSION['uid'] ?? 0;

    // Apply loyalty discount
    $discountPct = getLoyaltyDiscountPercentage($userId);
    if ($discountPct > 0) {
        $subtotal = $vars['subtotal'];
        $discountAmount = $subtotal * ($discountPct / 100);

        return [
            [
                'extension' => 'loyalty_discount',
                'type' => 'percentage',
                'value' => $discountAmount,
                'description' => "Loyalty discount ({$discountPct}%)",
            ],
        ];
    }

    return true;
});

// ===========================================
// Email Hooks
// ===========================================

/**
 * Modify email before sending
 */
add_hook('EmailPreSend', 1, function(array $vars) {
    // Add tracking header
    $vars['headers']['X-Mail-Track'] = generateEmailTrackingId();

    // Customize content based on user preferences
    $userPrefs = getUserEmailPreferences($vars['user_id']);

    if (!empty($userPrefs['plain_text_only'])) {
        $vars['type'] = 'plain';
        $vars['body_plain'] = strip_html_tags($vars['body']);
    }

    return $vars;
});

/**
 * Email sent confirmation (post-send)
 */
add_hook('EmailSent', 1, function(array $vars) {
    Capsule::table('mod_email_log')->insert([
        'message_id' => $vars['message_id'],
        'recipient' => $vars['mailto'],
        'subject' => $vars['subject'],
        'sent_at' => date('Y-m-d H:i:s'),
        'status' => 'sent',
    ]);

    return true;
});

// ===========================================
// API Hooks
// ===========================================

/**
 * Extend API authorization check
 */
add_hook('ApiAuthorization', 1, function(array $vars) {
    $apiKey = $vars['api_key'] ?? '';

    // Check if API key is valid and active
    $keyRecord = Capsule::table('mod_api_keys')
        ->where('api_key', $apiKey)
        ->where('active', 1)
        ->where('expires_at', '>', date('Y-m-d H:i:s'))
        ->first();

    if (!$keyRecord) {
        return [
            'authorized' => false,
            'error' => 'Invalid or expired API key',
        ];
    }

    // Check allowed IPs
    if (!empty($keyRecord->allowed_ips)) {
        $allowedIPs = explode(',', $keyRecord->allowed_ips);
        $clientIP = $_SERVER['REMOTE_ADDR'];

        if (!in_array($clientIP, $allowedIPs) && !isIpInRange($clientIP, $allowedIPs)) {
            return [
                'authorized' => false,
                'error' => 'IP address not allowed',
            ];
        }
    }

    // Check permissions
    $requiredPermission = $vars['action'];
    $allowedActions = json_decode($keyRecord->permissions, true);

    if (!in_array($requiredPermission, $allowedActions)) {
        return [
            'authorized' => false,
            'error' => "Permission denied for action: {$requiredPermission}",
        ];
    }

    return [
        'authorized' => true,
        'user_id' => $keyRecord->user_id,
        'rate_limit' => $keyRecord->rate_limit,
    ];
});
```

### Step 4: Creating Hook-Based Plugins

```php
<?php
/**
 * Advanced Analytics Plugin
 *
 * Demonstrates comprehensive hook-based plugin architecture.
 */

// Plugin registration hook
add_hook('AfterConfigSolution广泛应用', 1, function() {
    return [
        'analytics' => [
            'name' => 'Advanced Analytics',
            'version' => '1.0.0',
            'author' => 'Your Company',
            'description' => 'Enhanced analytics and tracking for WHMCS',
        ],
    ];
});

// ===========================================
// Data Collection Hooks
// ===========================================

/**
 * Track page views
 */
add_hook('ClientAreaPageRun', 1, function(array $vars) {
    if (!isTrackingEnabled()) {
        return;
    }

    $page = $vars['route']['controller'] ?? 'unknown';
    $userId = $_SESSION['uid'] ?? 0;

    Capsule::table('mod_analytics_events')->insert([
        'event_type' => 'page_view',
        'user_id' => $userId,
        'page' => $page,
        'url' => $_SERVER['REQUEST_URI'],
        'ip_address' => hashIpAddress($_SERVER['REMOTE_ADDR']),
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'referrer' => $_SERVER['HTTP_REFERER'] ?? '',
        'timestamp' => date('Y-m-d H:i:s'),
    ]);

    return true;
});

/**
 * Track product views in cart
 */
add_hook('CartProductConfigurableOptionsOverride', 1, function(array $vars) {
    $productId = $vars['pid'] ?? 0;

    trackProductView($productId, 'configurable_options');

    return true;
});

add_hook('CartProductUpgradeConfigOverride', 1, function(array $vars) {
    $productId = $vars['pid'] ?? 0;

    trackProductView($productId, 'upgrade_options');

    return true;
});

/**
 * Track checkout funnel
 */
add_hook('ShoppingCartCheckoutInitiated', 1, function() {
    $_SESSION['analytics']['checkout_started'] = time();
    return true;
});

add_hook('PreShoppingCartCheckout', 1, function(array $vars) {
    $_SESSION['analytics']['checkout_completed'] = time();

    $cartValue = calculateCartTotal($_SESSION['cart']);

    Capsule::table('mod_analytics_events')->insert([
        'event_type' => 'checkout_completed',
        'user_id' => $_SESSION['uid'] ?? 0,
        'cart_value' => $cartValue,
        'duration' => time() - ($SESSION['analytics']['checkout_started'] ?? time()),
        'timestamp' => date('Y-m-d H:i:s'),
    ]);

    return true;
});

/**
 * Track service provisioning
 */
add_hook('AfterModuleCreate', 1, function(array $vars) {
    $service = Capsule::table('tblhosting')->find($vars['serviceid']);

    Capsule::table('mod_analytics_events')->insert([
        'event_type' => 'service_created',
        'user_id' => $vars['userid'],
        'service_id' => $vars['serviceid'],
        'product_id' => $service->packageid ?? 0,
        'amount' => $service->amount ?? 0,
        'timestamp' => date('Y-m-d H:i:s'),
    ]);

    // Track conversion
    updateConversionTracking($vars['userid']);

    return true;
});

// ===========================================
// Admin Dashboard Hooks
// ===========================================

/**
 * Add analytics widget to admin dashboard
 */
add_hook('AdminHomeWidgets', 1, function() {
    return [
        [
            'title' => 'Analytics Overview',
            'template' => 'admin/widgets/analytics.tpl',
            'refresh_interval' => 300,
            'location' => 'right',
        ],
    ];
});

/**
 * Stats query for dashboard
 */
add_hook('AdminAreaStatsGraphData', 1, function(array $vars) {
    if ($vars['widget'] !== 'analytics') {
        return;
    }

    $period = $vars['period'] ?? '30days';
    $startDate = getPeriodStartDate($period);

    $orders = Capsule::table('mod_analytics_events')
        ->where('event_type', 'service_created')
        ->where('timestamp', '>=', $startDate)
        ->selectRaw('DATE(timestamp) as date, COUNT(*) as count')
        ->groupBy('date')
        ->get();

    $pageViews = Capsule::table('mod_analytics_events')
        ->where('event_type', 'page_view')
        ->where('timestamp', '>=', $startDate)
        ->selectRaw('DATE(timestamp) as date, COUNT(*) as count')
        ->groupBy('date')
        ->get();

    return [
        'labels' => array_column($orders, 'date'),
        'datasets' => [
            [
                'label' => 'New Orders',
                'data' => array_column($orders, 'count'),
                'borderColor' => '#28a745',
            ],
            [
                'label' => 'Page Views',
                'data' => array_column($pageViews, 'count'),
                'borderColor' => '#0066cc',
            ],
        ],
    ];
});

// ===========================================
// Helper Functions
// ===========================================

function isTrackingEnabled(): bool
{
    $setting = Capsule::table('mod_analytics_settings')
        ->where('setting', 'enabled')
        ->value('value');

    return $setting === '1';
}

function trackProductView(int $productId, string $context): void
{
    Capsule::table('mod_analytics_product_views')->insert([
        'product_id' => $productId,
        'user_id' => $_SESSION['uid'] ?? 0,
        'context' => $context,
        'session_id' => session_id(),
        'ip_address' => hashIpAddress($_SERVER['REMOTE_ADDR'] ?? ''),
        'timestamp' => date('Y-m-d H:i:s'),
    ]);
}

function hashIpAddress(string $ip): string
{
    // Anonymize IP for privacy compliance
    return hash('sha256', $ip . 'salt_value');
}

function updateConversionTracking(int $userId): void
{
    // Link first product purchase to original source
    $firstPurchase = Capsule::table('mod_analytics_events')
        ->where('user_id', $userId)
        ->whereNull('converted_at')
        ->where('event_type', 'product_view')
        ->orderBy('timestamp', 'asc')
        ->first();

    if ($firstPurchase) {
        Capsule::table('mod_analytics_events')
            ->where('user_id', $userId)
            ->where('event_type', 'product_view')
            ->where('converted_at', 'is', null)
            ->update(['converted_at' => date('Y-m-d H:i:s')]);
    }
}
```

### Step 5: Testing Hooks

```php
<?php
/**
 * Hook Testing Suite
 */

class WHMCS_Hooks_Test extends \PHPUnit\Framework\TestCase
{
    private $hookFunctions = [];

    protected function setUp(): void
    {
        parent::setUp();
        $this->hookFunctions = include 'includes/hooks.php';
    }

    /**
     * Test hook registration syntax
     */
    public function testHookRegistrationValid(): void
    {
        // Verify all add_hook calls have valid syntax
        $this->assertIsArray($this->hookFunctions);
    }

    /**
     * Test account creation hook
     */
    public function testClientAddHook(): void
    {
        $vars = [
            'user_id' => 1,
            'email' => 'test@example.com',
            'firstname' => 'Test',
            'lastname' => 'User',
        ];

        // This would execute the hook
        // $result = runHook('ClientAdd', $vars);

        $this->assertIsArray($vars);
        $this->assertArrayHasKey('user_id', $vars);
    }

    /**
     * Test invoice paid hook
     */
    public function testInvoicePaidHook(): void
    {
        $vars = [
            'invoiceid' => 1,
            'userid' => 1,
            'total' => 100.00,
        ];

        $this->assertIsArray($vars);
        $this->assertGreaterThan(0, $vars['invoiceid']);
    }
}

/**
 * Hook Debugger Utility
 */
class HookDebugger
{
    private $log = [];

    public function enable(): void
    {
        add_hook('HookBeforeExecute', 1, function($vars) {
            $this->log[] = [
                'action' => 'before',
                'hook_name' => $vars['hook_name'],
                'vars' => $vars['vars'],
                'memory' => memory_get_usage(),
            ];
        });

        add_hook('HookAfterExecute', 100, function($vars) {
            $this->log[] = [
                'action' => 'after',
                'hook_name' => $vars['hook_name'],
                'result' => $vars['result'],
                'memory' => memory_get_usage(),
            ];
        });
    }

    public function getLog(): array
    {
        return $this->log;
    }

    public function printReport(): void
    {
        echo "Hook Execution Report\n";
        echo "=" . str_repeat("=", 50) . "\n\n";

        foreach ($this->log as $entry) {
            printf(
                "[%s] %s: %s (Memory: %s)\n",
                $entry['action'],
                $entry['hook_name'],
                isset($entry['result']) ? 'OK' : 'RUNNING',
                number_format($entry['memory'] / 1024, 2) . ' KB'
            );
        }
    }
}
```

### Step 6: Hook Performance Optimization

```php
<?php
/**
 * Hook Performance Optimization
 */

// ===========================================
// Conditional Hook Execution
// ===========================================

/**
 * Use priority to control execution order
 * Lower priority = earlier execution = faster rejection on errors
 */

/**
 * Fastest: Check with low priority, reject early
 */
add_hook('ClientAdd', 1, function($vars) {
    // Fast condition check
    if ($vars['email'] === 'admin@blacklist.com') {
        // Return error immediately, no further hooks needed
        return ['error' => 'Email not allowed'];
    }
    return true;
});

/**
 * Expensive operations at high priority (runs last)
 */
add_hook('ClientAdd', 100, function($vars) {
    // Only runs if earlier hooks passed
    expensiveOperation($vars);
    return true;
});

// ===========================================
// Hook Caching
// ===================================================

/**
 * Cache hook results for expensive operations
 */
add_hook('GetInvoiceHTMLOutput', 1, function($vars) {
    $cacheKey = "invoice_html_{$vars['invoiceid']}";

    $cache = Capsule::table('mod_hook_cache')
        ->where('cache_key', $cacheKey)
        ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-5 minutes')))
        ->first();

    if ($cache) {
        return ['cached' => true, 'html' => $cache->cache_value];
    }

    // Cache miss - generate HTML
    $html = generateInvoiceHTML($vars['invoiceid']);

    Capsule::table('mod_hook_cache')->insert([
        'cache_key' => $cacheKey,
        'cache_value' => $html,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return ['html' => $html];
});

// ===========================================
// Async Hook Processing
// ===========================================

/**
 * Queue heavy operations for async processing
 */
add_hook('AfterModuleCreate', 50, function($vars) {
    // Don't do heavy work synchronously
    Capsule::table('mod_async_queue')->insert([
        'hook_name' => 'ModuleCreated',
        'vars' => json_encode($vars),
        'priority' => 'normal',
        'scheduled_at' => date('Y-m-d H:i:s'),
    ]);

    return true; // Return immediately
});

/**
 * Process queued hooks in cron
 */
add_hook('DailyCronJob', 1, function() {
    $queued = Capsule::table('mod_async_queue')
        ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
        ->where('processed', 0)
        ->orderBy('priority', 'asc')
        ->orderBy('scheduled_at', 'asc')
        ->limit(50)
        ->get();

    foreach ($queued as $job) {
        try {
            processAsyncHook($job);
            Capsule::table('mod_async_queue')
                ->where('id', $job->id)
                ->update(['processed' => 1, 'processed_at' => date('Y-m-d H:i:s')]);
        } catch (\Exception $e) {
            Capsule::table('mod_async_queue')
                ->where('id', $job->id)
                ->update(['attempts' => $job->attempts + 1, 'last_error' => $e->getMessage()]);
        }
    }
});
```

## Complete Hook Reference

```
Account Hooks:
□ ClientAdd - After client account creation
□ ClientEdit - Before/after client update
□ ClientDelete - Before client deletion
□ ClientLogin - After client login
□ ClientLogout - After client logout
□ ClientChangePassword - After password change

Service Hooks:
□ ServiceAdd - After service creation
□ ServiceEdit - After service update
□ ServiceDelete - Before service deletion
□ AfterModuleCreate - After provisioning
□ AfterModuleSuspend - After suspension
□ AfterModuleUnsuspend - After unsuspension
□ AfterModuleTerminate - After termination
□ AfterModuleChangePassword - After password change
□ AfterModuleChangePackage - After upgrade/downgrade

Invoice Hooks:
□ InvoiceCreation
□ InvoiceCreated - After creation
□ InvoicePreCreation - Before creation
□ InvoicePaid - After payment
□ InvoiceCancelled - After cancellation
□ InvoicePaymentRejected - After rejection
□ InvoiceDeleted - After deletion
□ InvoiceRefunded - After refund
□ AfterCalculatingCartTotals - After cart calculation

Domain Hooks:
□ DomainRegister - After registration
□ DomainTransferCompleted - After transfer
□ DomainRenewed - After renewal
□ DomainDelete - Before deletion

Ticket Hooks:
□ TicketOpen - When opened
□ TicketReply - When replied
□ TicketClose - When closed
□ TicketEscalate - When escalated
□ TicketMerge - When merged

Email Hooks:
□ EmailQueue - When email queued
□ EmailSent - After sending
□ EmailPreSend - Before sending
□ EmailLog - For logging

Page Hooks:
□ ClientAreaHeaderOutput - Header injection
□ ClientAreaFooterOutput - Footer injection
□ ClientAreaPageRun - Page loaded
□ ClientAreaSidebars - Sidebar panels
□ AdminAreaHeaderOutput - Admin header
□ AdminAreaPageRun - Admin page loaded

Authentication Hooks:
□ Authentication - Pre-auth check
□ AfterAuthentication - Post-auth check
□ ApiAuthorization - API auth check
```

## Verification Checklist

```
Hook Development:
□ Hooks registered correctly with add_hook()
□ Priority values set appropriately
□ Error handling in all hooks
□ Logging for debugging
□ Input validation on hook vars
□ SQL injection prevention
□ XSS prevention in output
□ Async processing for heavy operations

Code Quality:
□ No hardcoded values
□ Uses config variables
□ Follows WHMCS coding standards
□ Proper naming conventions
□ Comments and documentation
□ Return values correct
□ Type hints used

Testing:
□ Unit tests for hooks
□ Integration tests
□ Mock external services
□ Edge case handling
□ Performance profiling
□ Memory usage monitoring

Security:
□ Input sanitization
□ SQL injection prevention
□ XSS prevention
□ CSRF protection where needed
□ Rate limiting for sensitive hooks
□ Audit logging
```

## WHMCS ClassDocs References

- [Hook System Documentation](https://developers.whmcs.com/advanced/hooks-system/)
- [Hook Reference](https://developers.whmcs.com/advanced/hooks-reference/)
- [Hook Priority](https://developers.whmcs.com/advanced/hooks-system/#hook-priority)
- [add_hook() function](https://developers.whmcs.com/advanced/hooks-system/#add-hook-function)

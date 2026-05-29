# WHMCS New Client Onboarding Workflow

## Overview
This workflow automates and standardizes the client onboarding process in WHMCS, from registration to first service provisioning.

## Onboarding Stages

```
Registration → Email Verification → Welcome Email → Service Setup → First Login → Training
```

## Step 1: Configure Onboarding Settings

```php
<?php
// includes/hooks/onboarding_hook.php

use WHMCS\Database\Capsule;

// Hook: After client registration
add_hook('ClientAreaRegistration', 1, function($params) {
    $clientId = $params['userid'];

    // Create onboarding record
    Capsule::table('mod_onboarding')->insert([
        'client_id' => $clientId,
        'stage' => 'registered',
        'created_at' => date('Y-m-d H:i:s'),
        'welcome_email_sent' => false,
        'setup_complete' => false
    ]);

    // Trigger welcome email sequence
    run_hook('OnboardingWelcomeEmail', ['client_id' => $clientId]);

    return $params;
});

// Hook: After email verification
add_hook('AfterEmailVerification', 1, function($params) {
    $clientId = $params['client_id'];

    Capsule::table('mod_onboarding')
        ->where('client_id', $clientId)
        ->update([
            'email_verified' => true,
            'email_verified_at' => date('Y-m-d H:i:s')
        ]);

    // Trigger next onboarding step
    run_hook('OnboardingEmailVerified', ['client_id' => $clientId]);

    return $params;
});
```

## Step 2: Welcome Email Automation

```php
<?php
// src/Service/WelcomeEmailService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;
use WHMCS\Mail\Email;
use WHMCS\View\Formatter\Date;

class WelcomeEmailService
{
    private $emailQueue;
    private $templateService;

    public function __construct()
    {
        $this->emailQueue = new EmailQueueService();
        $this->templateService = new EmailTemplateService();
    }

    public function sendWelcomeSequence(int $clientId): void
    {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        if (!$client) {
            throw new \Exception("Client not found: $clientId");
        }

        // Day 0: Welcome email
        $this->sendEmail($clientId, 'welcome', [
            'client_name' => $client->firstname . ' ' . $client->lastname,
            'client_email' => $client->email,
            'login_url' => $this->getClientLoginUrl(),
            'support_url' => $this->getSupportUrl()
        ]);

        // Day 1: Getting started guide
        $this->scheduleEmail($clientId, 'getting_started', '+1 day');

        // Day 3: First service reminder
        $this->scheduleEmail($clientId, 'first_service', '+3 days');

        // Day 7: Tutorial invitation
        $this->scheduleEmail($clientId, 'tutorial_invite', '+7 days');

        // Update onboarding status
        $this->updateOnboardingStatus($clientId, 'welcome_sequence_started');
    }

    private function sendEmail(int $clientId, string $template, array $vars): void
    {
        $email = new Email();
        $email->sendEmail($template, [
            'id' => $clientId,
            'vars' => $vars
        ]);

        // Log email sent
        $this->logEmailSent($clientId, $template);
    }

    private function scheduleEmail(int $clientId, string $template, string $delay): void
    {
        $sendAt = date('Y-m-d H:i:s', strtotime($delay));

        Capsule::table('mod_onboarding_email_queue')->insert([
            'client_id' => $clientId,
            'template' => $template,
            'send_at' => $sendAt,
            'status' => 'scheduled',
            'created_at' => date('Y-md H:i:s')
        ]);
    }

    private function updateOnboardingStatus(int $clientId, string $stage): void
    {
        Capsule::table('mod_onboarding')
            ->where('client_id', $clientId)
            ->update([
                'stage' => $stage,
                'updated_at' => date('Y-m-d H:i:s')
            ]);
    }

    private function logEmailSent(int $clientId, string $template): void
    {
        Capsule::table('mod_onboarding_email_log')->insert([
            'client_id' => $clientId,
            'email_type' => $template,
            'sent_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function getClientLoginUrl(): string
    {
        return Capsule::config('system_url') . '/clientarea.php';
    }

    private function getSupportUrl(): string
    {
        return Capsule::config('system_url') . '/supporttickets.php';
    }
}
```

## Step 3: Client Profile Completion

```php
<?php
// src/Service/ProfileCompletionService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class ProfileCompletionService
{
    private $requiredFields = [
        'firstname',
        'lastname',
        'email',
        'phonenumber',
        'address1',
        'city',
        'state',
        'country',
        'postcode'
    ];

    private $completionThresholds = [
        'basic' => 30,
        'standard' => 60,
        'complete' => 100
    ];

    public function calculateCompletionPercentage(int $clientId): int
    {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        if (!$client) {
            return 0;
        }

        $filledFields = 0;
        $totalFields = count($this->requiredFields);

        foreach ($this->requiredFields as $field) {
            $value = $client->$field ?? null;
            if (!empty($value) && $value !== ' ') {
                $filledFields++;
            }
        }

        // Add bonus for additional fields
        $additionalFields = ['companyname', 'tax_id', 'password_id'];
        foreach ($additionalFields as $field) {
            if (!empty($client->$field)) {
                $filledFields += 0.5;
            }
        }

        return (int) round(($filledFields / $totalFields) * 100);
    }

    public function getMissingFields(int $clientId): array
    {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        $missing = [];
        foreach ($this->requiredFields as $field) {
            $value = $client->$field ?? null;
            if (empty($value) || $value === ' ') {
                $missing[] = $field;
            }
        }

        return $missing;
    }

    public function sendCompletionReminder(int $clientId): void
    {
        $percentage = $this->calculateCompletionPercentage($clientId);
        $missingFields = $this->getMissingFields($clientId);

        if ($percentage < 100 && count($missingFields) > 0) {
            $client = Capsule::table('tblclients')
                ->where('id', $clientId)
                ->first();

            // Send reminder email
            send_email('profile_completion_reminder', $client->email, [
                'client_name' => $client->firstname,
                'missing_fields' => implode(', ', $missingFields),
                'completion_percentage' => $percentage,
                'profile_edit_url' => Capsule::config('system_url') . '/clientarea.php?action=details'
            ]);
        }
    }
}
```

## Step 4: Service Provisioning on Signup

```php
<?php
// src/Service/ServiceProvisioningService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;
use WHMCS\Module\Server;
use WHMCS\Service\Status;

class ServiceProvisioningService
{
    public function provisionFirstService(array $params): array
    {
        $clientId = $params['client_id'];
        $productId = $params['product_id'];
        $billingCycle = $params['billing_cycle'] ?? 'monthly';

        // Validate client
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        if (!$client) {
            return ['success' => false, 'error' => 'Client not found'];
        }

        // Validate product
        $product = Capsule::table('tblproducts')
            ->where('id', $productId)
            ->first();

        if (!$product) {
            return ['success' => false, 'error' => 'Product not found'];
        }

        // Create order
        $orderId = $this->createOrder($clientId, $productId, $billingCycle);

        // Auto-approve order
        $this->approveOrder($orderId);

        // Provision service
        $provisionResult = $this->provisionService($orderId);

        if ($provisionResult['success']) {
            // Send provisioning notification
            $this->sendProvisioningEmail($clientId, $provisionResult);
        }

        return $provisionResult;
    }

    private function createOrder(int $clientId, int $productId, string $billingCycle): int
    {
        $orderNum = generate_unique_order_number();

        $orderId = Capsule::table('tblorders')->insertGetId([
            'userid' => $clientId,
            'ordernum' => $orderNum,
            'date' => date('Y-m-d H:i:s'),
            'status' => 'Pending',
            'paymentmethod' => 'free',
            'totalamount' => 0,
            'billingcycle' => $billingCycle
        ]);

        // Add order item
        Capsule::table('tblorderitems')->insert([
            'orderid' => $orderId,
            'type' => 'hostingaccount',
            'relid' => 0,
            'productid' => $productId,
            'domain' => '',
            'billingcycle' => $billingCycle,
            'amount' => 0,
            'status' => 'Pending'
        ]);

        return $orderId;
    }

    private function approveOrder(int $orderId): void
    {
        Capsule::table('tblorders')
            ->where('id', $orderId)
            ->update(['status' => 'Active']);

        Capsule::table('tblorderitems')
            ->where('orderid', $orderId)
            ->update(['status' => 'Active']);
    }

    private function provisionService(int $orderId): array
    {
        $orderItems = Capsule::table('tblorderitems')
            ->where('orderid', $orderId)
            ->get();

        $results = [];
        foreach ($orderItems as $item) {
            if ($item->type === 'hostingaccount') {
                $results[] = $this->provisionHosting($item, $orderId);
            }
        }

        return [
            'success' => !in_array(false, array_column($results, 'success')),
            'results' => $results
        ];
    }

    private function provisionHosting($item, int $orderId): array
    {
        $order = Capsule::table('tblorders')
            ->where('id', $orderId)
            ->first();

        $product = Capsule::table('tblproducts')
            ->where('id', $item->productid)
            ->first();

        // Create hosting account
        $hostingId = Capsule::table('tblhosting')->insertGetId([
            'userid' => $order->userid,
            'orderid' => $orderId,
            'packageid' => $item->productid,
            'serverid' => $product->servergroup ? $this->getServerForProduct($product->id) : 0,
            'regdate' => date('Y-m-d H:i:s'),
            'domainstatus' => 'Active',
            'billingcycle' => $item->billingcycle,
            'nextduedate' => date('Y-m-d'),
            'paymentmethod' => $order->paymentmethod,
            'username' => $this->generateUsername($order->userid),
            'password' => encrypt(random_string(12))
        ]);

        // Update order item with hosting ID
        Capsule::table('tblorderitems')
            ->where('id', $item->id)
            ->update(['relid' => $hostingId]);

        return [
            'success' => true,
            'hosting_id' => $hostingId,
            'service' => $hostingId
        ];
    }

    private function generateUsername(int $clientId): string
    {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        $prefix = 'usr';
        $random = substr(md5($clientId . time()), 0, 6);

        return strtolower($prefix . $random);
    }

    private function getServerForProduct(int $productId): int
    {
        $server = Capsule::table('tblservers')
            ->join('tblservergroupsrel', 'tblservers.id', '=', 'tblservergroupsrel.serverid')
            ->join('tblproducts', 'tblservergroupsrel.groupid', '=', 'tblproducts.servergroup')
            ->where('tblproducts.id', $productId)
            ->where('tblservers.active', 1)
            ->orderBy('tblservers.weight')
            ->first();

        return $server ? $server->serverid : 0;
    }

    private function sendProvisioningEmail(int $clientId, array $result): void
    {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        send_email('service_provisioned', $client->email, [
            'client_name' => $client->firstname . ' ' . $client->lastname,
            'service_details' => $result,
            'login_url' => Capsule::config('system_url') . '/clientarea.php',
            'support_url' => Capsule::config('system_url') . '/supporttickets.php'
        ]);
    }
}
```

## Step 5: Onboarding Progress Tracking

```php
<?php
// src/Service/OnboardingTrackerService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class OnboardingTrackerService
{
    private $stages = [
        'registered' => 0,
        'email_verified' => 10,
        'profile_completed' => 30,
        'first_login' => 50,
        'service_activated' => 70,
        'first_payment' => 90,
        'completed' => 100
    ];

    public function getOnboardingProgress(int $clientId): array
    {
        $record = Capsule::table('mod_onboarding')
            ->where('client_id', $clientId)
            ->first();

        if (!$record) {
            return [
                'exists' => false,
                'progress' => 0,
                'stage' => 'not_started',
                'next_steps' => $this->getInitialSteps()
            ];
        }

        $currentStage = $record->stage ?? 'registered';
        $progress = $this->stages[$currentStage] ?? 0;

        return [
            'exists' => true,
            'progress' => $progress,
            'stage' => $currentStage,
            'started_at' => $record->created_at,
            'last_activity' => $record->updated_at,
            'next_steps' => $this->getNextSteps($currentStage),
            'completed_steps' => $this->getCompletedSteps($currentStage),
            'emails_sent' => $this->getEmailsSent($clientId)
        ];
    }

    public function advanceStage(int $clientId, string $newStage): void
    {
        if (!isset($this->stages[$newStage])) {
            throw new \Exception("Invalid stage: $newStage");
        }

        Capsule::table('mod_onboarding')
            ->where('client_id', $clientId)
            ->update([
                'stage' => $newStage,
                'updated_at' => date('Y-m-d H:i:s')
            ]);

        // Trigger notifications for stage
        run_hook('OnboardingStageAdvanced', [
            'client_id' => $clientId,
            'new_stage' => $newStage
        ]);
    }

    private function getInitialSteps(): array
    {
        return [
            ['step' => 'verify_email', 'description' => 'Verify your email address'],
            ['step' => 'complete_profile', 'description' => 'Complete your profile information'],
            ['step' => 'first_login', 'description' => 'Log in to your account']
        ];
    }

    private function getNextSteps(string $currentStage): array
    {
        $steps = [];

        switch ($currentStage) {
            case 'registered':
                $steps[] = ['step' => 'verify_email', 'description' => 'Verify your email address'];
                $steps[] = ['step' => 'complete_profile', 'description' => 'Complete your profile'];
                break;
            case 'email_verified':
                $steps[] = ['step' => 'complete_profile', 'description' => 'Complete your profile'];
                $steps[] = ['step' => 'first_login', 'description' => 'Log in to your account'];
                break;
            case 'profile_completed':
                $steps[] = ['step' => 'first_login', 'description' => 'Log in to your account'];
                $steps[] = ['step' => 'view_dashboard', 'description' => 'Explore your dashboard'];
                break;
            case 'first_login':
                $steps[] = ['step' => 'activate_service', 'description' => 'Activate your first service'];
                break;
            default:
                $steps[] = ['step' => 'complete_setup', 'description' => 'Complete your setup'];
        }

        return $steps;
    }

    private function getCompletedSteps(string $currentStage): array
    {
        $completed = [];

        foreach ($this->stages as $stage => $progress) {
            if ($stage === $currentStage) {
                break;
            }
            $completed[] = $stage;
        }

        return $completed;
    }

    private function getEmailsSent(int $clientId): int
    {
        return Capsule::table('mod_onboarding_email_log')
            ->where('client_id', $clientId)
            ->count();
    }
}
```

## Step 6: Automated Cron Job

```php
<?php
// includes/cron/onboarding_cron.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Database\Capsule;
use WHMCS\Module\Addon\YourModule\Service\WelcomeEmailService;
use WHMCS\Module\Addon\YourModule\Service\ProfileCompletionService;
use WHMCS\Module\Addon\YourModule\Service\OnboardingTrackerService;

echo "=== Onboarding Cron Job ===\n";
echo "Started: " . date('Y-m-d H:i:s') . "\n\n";

// Initialize services
$welcomeService = new WelcomeEmailService();
$profileService = new ProfileCompletionService();
$trackerService = new OnboardingTrackerService();

// Process email queue
echo "Processing email queue...\n";
processEmailQueue();

// Process profile reminders
echo "Processing profile reminders...\n";
processProfileReminders();

// Update stale onboarding records
echo "Updating stale records...\n";
updateStaleRecords();

echo "\n=== Onboarding Cron Complete ===\n";

function processEmailQueue(): void
{
    $pendingEmails = Capsule::table('mod_onboarding_email_queue')
        ->where('status', 'scheduled')
        ->where('send_at', '<=', date('Y-m-d H:i:s'))
        ->limit(100)
        ->get();

    foreach ($pendingEmails as $email) {
        try {
            send_email($email->template, $email->client_id);
            Capsule::table('mod_onboarding_email_queue')
                ->where('id', $email->id)
                ->update([
                    'status' => 'sent',
                    'sent_at' => date('Y-m-d H:i:s')
                ]);
        } catch (\Exception $e) {
            Capsule::table('mod_onboarding_email_queue')
                ->where('id', $email->id)
                ->update([
                    'status' => 'failed',
                    'error' => $e->getMessage()
                ]);
        }
    }

    echo "Processed " . count($pendingEmails) . " emails\n";
}

function processProfileReminders(): void
{
    $clients = Capsule::table('tblclients')
        ->where('datecreated', '>=', date('Y-m-d', strtotime('-7 days')))
        ->where('email_verified', 1)
        ->get();

    $profileService = new ProfileCompletionService();

    foreach ($clients as $client) {
        $percentage = $profileService->calculateCompletionPercentage($client->id);

        if ($percentage < 100) {
            // Check if reminder was already sent recently
            $recentReminder = Capsule::table('mod_onboarding_email_log')
                ->where('client_id', $client->id)
                ->where('email_type', 'profile_completion_reminder')
                ->where('sent_at', '>=', date('Y-m-d H:i:s', strtotime('-3 days')))
                ->first();

            if (!$recentReminder) {
                $profileService->sendCompletionReminder($client->id);
            }
        }
    }
}

function updateStaleRecords(): void
{
    // Mark onboarding as stale after 30 days of inactivity
    Capsule::table('mod_onboarding')
        ->where('updated_at', '<', date('Y-m-d H:i:s', strtotime('-30 days')))
        ->where('stage', '!=', 'completed')
        ->update(['stage' => 'stale']);
}
```

## Step 7: Admin Dashboard Widget

```php
<?php
// admin/onboarding_widget.php

add_hook('AdminHomepage', 1, function() {
    $stats = getOnboardingStats();

    return [
        'templatefile' => 'admin/widgets/onboarding_stats',
        'vars' => [
            'onboarding_stats' => $stats
        ]
    ];
});

function getOnboardingStats(): array
{
    $db = Capsule::connection()->getPdo();

    // Get counts by stage
    $stmt = $db->query("
        SELECT stage, COUNT(*) as count
        FROM mod_onboarding
        GROUP BY stage
    ");
    $stages = $stmt->fetchAll(PDO::FETCH_KEY_PAIR);

    // Get recent signups
    $recentSignups = Capsule::table('mod_onboarding')
        ->orderBy('created_at', 'desc')
        ->limit(10)
        ->get();

    // Get completion rate
    $total = Capsule::table('mod_onboarding')->count();
    $completed = Capsule::table('mod_onboarding')
        ->where('stage', 'completed')
        ->count();

    return [
        'by_stage' => $stages,
        'total' => $total,
        'completed' => $completed,
        'completion_rate' => $total > 0 ? round(($completed / $total) * 100) : 0,
        'recent_signups' => $recentSignups,
        'pending_emails' => Capsule::table('mod_onboarding_email_queue')
            ->where('status', 'scheduled')
            ->count()
    ];
}
```

## Verification Checklist

- [ ] Onboarding table created in database
- [ ] Email queue table created
- [ ] Welcome email templates configured
- [ ] Hooks registered for registration events
- [ ] Profile completion tracking enabled
- [ ] Service provisioning automation configured
- [ ] Onboarding tracker service implemented
- [ ] Cron job configured and scheduled
- [ ] Admin dashboard widget created
- [ ] Test client onboarded successfully
- [ ] Email sequence verified working
- [ ] Progress tracking verified

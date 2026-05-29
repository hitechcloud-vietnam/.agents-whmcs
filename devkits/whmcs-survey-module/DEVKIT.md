# WHMCS Survey Module

## Overview
Customer satisfaction survey module for WHMCS.

## Module File: survey.php

```php
<?php
/**
 * WHMCS Survey Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Database\Capsule;

class WHMCS_Survey
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    /**
     * Create survey request
     */
    public function createSurveyRequest(int $ticketId, int $userId): int
    {
        return Capsule::table('mod_survey_requests')->insertGetId([
            'ticket_id' => $ticketId,
            'user_id' => $userId,
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Submit survey response
     */
    public function submitResponse(int $requestId, int $rating, ?string $feedback): bool
    {
        Capsule::table('mod_survey_requests')
            ->where('id', $requestId)
            ->update([
                'rating' => $rating,
                'feedback' => $feedback,
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);

        // Log response
        Capsule::table('mod_survey_responses')->insert([
            'request_id' => $requestId,
            'rating' => $rating,
            'feedback' => $feedback,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return true;
    }

    /**
     * Get survey statistics
     */
    public function getStatistics(): array
    {
        $stats = Capsule::table('mod_survey_requests')
            ->selectRaw('
                COUNT(*) as total,
                AVG(rating) as avg_rating,
                SUM(CASE WHEN rating >= 4 THEN 1 ELSE 0 END) as positive,
                SUM(CASE WHEN rating <= 2 THEN 1 ELSE 0 END) as negative
            ')
            ->first();

        return [
            'total_responses' => $stats->total ?? 0,
            'average_rating' => round($stats->avg_rating ?? 0, 2),
            'satisfaction_rate' => $stats->total > 0 
                ? round(($stats->positive / $stats->total) * 100, 1) 
                : 0,
        ];
    }
}

// Hook to send survey after ticket close
add_hook('TicketClose', 1, function($params) {
    $survey = new WHMCS_Survey();
    $survey->createSurveyRequest($params['ticketid'], $params['userid']);
});

function whmcs_survey_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_survey_requests')) {
            Capsule::schema()->create('mod_survey_requests', function ($table) {
                $table->increments('id');
                $table->integer('ticket_id')->unsigned();
                $table->integer('user_id')->unsigned();
                $table->tinyInteger('rating')->nullable();
                $table->text('feedback')->nullable();
                $table->enum('status', ['pending', 'sent', 'completed', 'declined'])->default('pending');
                $table->timestamp('created_at')->useCurrent();
                $table->timestamp('completed_at')->nullable();
            });
        }

        if (!Capsule::schema()->hasTable('mod_survey_responses')) {
            Capsule::schema()->create('mod_survey_responses', function ($table) {
                $table->increments('id');
                $table->integer('request_id')->unsigned();
                $table->tinyInteger('rating');
                $table->text('feedback')->nullable();
                $table->string('ip_address', 45);
                $table->timestamp('created_at')->useCurrent();
            });
        }

        return ['status' => 'success', 'description' => 'Survey Module activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_survey_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Survey Module deactivated'];
}

function whmcs_survey_config(): array
{
    return [
        'auto_send' => ['FriendlyName' => 'Auto Send Survey', 'Type' => 'yesno', 'Description' => 'Send survey when ticket is closed'],
        'delay_hours' => ['FriendlyName' => 'Delay (hours)', 'Type' => 'text', 'Default' => '24'],
    ];
}
```

## Configuration File: config.php

```php
<?php
return [
    'auto_send' => true,
    'delay_hours' => 24,
];
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+

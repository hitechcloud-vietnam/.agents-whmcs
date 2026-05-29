# WHMCS Survey Builder DevKit

## Overview

Survey builder and management system for WHMCS enabling creation of custom surveys, polls, and feedback forms with analytics.

## Features

- Drag-drop survey builder
- Multiple question types
- Conditional logic
- Branching surveys
- Anonymous responses
- Partial completion
- Export to CSV/Excel
- Response analytics
- NPS scoring

## Module Files

```php
<?php
/**
 * WHMCS Survey Builder Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/SurveyBuilder.php';

function whmcs_survey_builder_activate() {
    $builder = new SurveyBuilder();
    return $builder->activate();
}

function whmcs_survey_create($data) {
    $builder = new SurveyBuilder();
    return $builder->createSurvey($data);
}

function whmcs_survey_submit($surveyId, $responses) {
    $builder = new SurveyBuilder();
    return $builder->submitResponse($surveyId, $responses);
}
```

### lib/SurveyBuilder.php

```php
<?php
namespace WHMCS\Module\SurveyBuilder;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class SurveyBuilder {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Survey Builder module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_surveys` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `survey_name` VARCHAR(255) NOT NULL,
                `survey_code` VARCHAR(50) NOT NULL,
                `description` TEXT NULL,
                `questions` JSON NOT NULL,
                `settings` JSON NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `start_date` DATETIME NULL,
                `end_date` DATETIME NULL,
                `allow_anonymous` TINYINT(1) NOT NULL DEFAULT 0,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_survey_responses` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `survey_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NULL,
                `responses` JSON NOT NULL,
                `completion_percentage` DECIMAL(5,2) DEFAULT 0,
                `nps_score` INT NULL,
                `submitted_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createSurvey($data) {
        $id = Capsule::table('mod_surveys')->insertGetId([
            'survey_name' => $data['name'],
            'survey_code' => 'SURV-' . strtoupper(substr(uniqid(), -6)),
            'description' => $data['description'] ?? null,
            'questions' => json_encode($data['questions']),
            'settings' => json_encode($data['settings'] ?? []),
            'start_date' => $data['start_date'] ?? null,
            'end_date' => $data['end_date'] ?? null,
        ]);
        
        return ['success' => true, 'survey_id' => $id];
    }
    
    public function submitResponse($surveyId, $responses, $userId = null) {
        $responseId = Capsule::table('mod_survey_responses')->insertGetId([
            'survey_id' => $surveyId,
            'user_id' => $userId,
            'responses' => json_encode($responses),
            'submitted_at' => Carbon::now(),
            'completion_percentage' => 100,
        ]);
        
        return ['success' => true, 'response_id' => $responseId];
    }
    
    public function getResults($surveyId) {
        $responses = Capsule::table('mod_survey_responses')
            ->where('survey_id', $surveyId)
            ->get();
        
        $survey = Capsule::table('mod_surveys')->where('id', $surveyId)->first();
        $questions = json_decode($survey->questions, true);
        
        $results = [];
        foreach ($questions as $q) {
            $results[$q['id']] = [
                'question' => $q['text'],
                'type' => $q['type'],
                'responses' => [],
            ];
        }
        
        foreach ($responses as $response) {
            $respData = json_decode($response->responses, true);
            foreach ($respData as $questionId => $answer) {
                if (isset($results[$questionId])) {
                    $results[$questionId]['responses'][] = $answer;
                }
            }
        }
        
        return [
            'total_responses' => count($responses),
            'questions' => $results,
        ];
    }
}
```

## API Endpoints

```
POST /api/v1/surveys                     - Create survey
GET  /api/v1/surveys                    - List surveys
GET  /api/v1/surveys/{id}              - Get survey
POST /api/v1/surveys/{id}/submit         - Submit response
GET  /api/v1/surveys/{id}/results        - Get results
```

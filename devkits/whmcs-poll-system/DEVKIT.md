# WHMCS Poll System DevKit

## Overview

Poll and voting system for WHMCS enabling creation of polls, voting, and result analysis.

## Module Files

```php
<?php
/**
 * WHMCS Poll System Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/PollSystem.php';

function whmcs_poll_system_activate() {
    $poll = new PollSystem();
    return $poll->activate();
}

function whmcs_poll_create($data) {
    $poll = new PollSystem();
    return $poll->createPoll($data);
}

function whmcs_poll_vote($pollId, $optionId, $userId) {
    $poll = new PollSystem();
    return $poll->vote($pollId, $optionId, $userId);
}
```

### lib/PollSystem.php

```php
<?php
namespace WHMCS\Module\PollSystem;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class PollSystem {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Poll System module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_polls` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `poll_name` VARCHAR(255) NOT NULL,
                `question` TEXT NOT NULL,
                `options` JSON NOT NULL,
                `is_multiple_choice` TINYINT(1) NOT NULL DEFAULT 0,
                `is_anonymous` TINYINT(1) NOT NULL DEFAULT 0,
                `start_date` DATETIME NOT NULL,
                `end_date` DATETIME NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_poll_votes` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `poll_id` INT UNSIGNED NOT NULL,
                `option_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NULL,
                `voted_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_poll_user` (`poll_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createPoll($data) {
        $id = Capsule::table('mod_polls')->insertGetId([
            'poll_name' => $data['name'],
            'question' => $data['question'],
            'options' => json_encode($data['options']),
            'start_date' => $data['start_date'] ?? Carbon::now(),
            'end_date' => $data['end_date'] ?? null,
        ]);
        
        return ['success' => true, 'poll_id' => $id];
    }
    
    public function vote($pollId, $optionId, $userId) {
        Capsule::table('mod_poll_votes')->insert([
            'poll_id' => $pollId,
            'option_id' => $optionId,
            'user_id' => $userId,
        ]);
        
        return ['success' => true];
    }
    
    public function getResults($pollId) {
        $poll = Capsule::table('mod_polls')->where('id', $pollId)->first();
        $votes = Capsule::table('mod_poll_votes')
            ->where('poll_id', $pollId)
            ->get();
        
        $options = json_decode($poll->options, true);
        $results = [];
        
        foreach ($options as $index => $option) {
            $count = $votes->where('option_id', $index)->count();
            $results[$option] = [
                'votes' => $count,
                'percentage' => $votes->count() > 0 ? ($count / $votes->count()) * 100 : 0,
            ];
        }
        
        return [
            'question' => $poll->question,
            'total_votes' => $votes->count(),
            'results' => $results,
        ];
    }
}
```

## API Endpoints

```
POST /api/v1/polls                        - Create poll
GET  /api/v1/polls                      - List polls
POST /api/v1/polls/{id}/vote            - Vote
GET  /api/v1/polls/{id}/results         - Get results
```

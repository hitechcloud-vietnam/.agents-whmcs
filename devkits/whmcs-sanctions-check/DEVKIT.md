# WHMCS Sanctions Check DevKit

## Overview

Sanctions screening and compliance verification system for WHMCS enabling real-time screening against OFAC, EU, UN, and other sanctions lists.

## Features

- Real-time sanctions screening
- Multi-list support (OFAC, EU, UN)
- Batch screening
- Watchlist management
- Alert management
- Audit trail
- Match scoring

## Module Files

```php
<?php
/**
 * WHMCS Sanctions Check Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/SanctionsChecker.php';

function whmcs_sanctions_check_activate() {
    $checker = new SanctionsChecker();
    return $checker->activate();
}

function whmcs_sanctions_check_screen($name, $entityType = 'individual') {
    $checker = new SanctionsChecker();
    return $checker->screen($name, $entityType);
}
```

### lib/SanctionsChecker.php

```php
<?php
namespace WHMCS\Module\SanctionsCheck;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class SanctionsChecker {
    
    protected $sanctionsLists = [
        'OFAC' => 'ofac_sdn',
        'EU' => 'eu_sanctions',
        'UN' => 'un_sanctions',
    ];
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Sanctions Check module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sanctions_screenings` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `screening_id` VARCHAR(64) NOT NULL,
                `user_id` INT UNSIGNED NULL,
                `entity_name` VARCHAR(255) NOT NULL,
                `entity_type` VARCHAR(50) NOT NULL,
                `match_score` DECIMAL(5,2) DEFAULT 0,
                `matched_lists` JSON NULL,
                `status` ENUM('clear', 'potential_match', 'confirmed_match') NOT NULL DEFAULT 'clear',
                `screened_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sanctions_watchlist` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `list_name` VARCHAR(50) NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `aliases` JSON NULL,
                `address` TEXT NULL,
                `dates_of_birth` JSON NULL,
                `nationality` VARCHAR(100) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function screen($name, $entityType = 'individual') {
        $screeningId = 'SCR-' . strtoupper(substr(md5(uniqid()), 0, 12));
        
        $matches = $this->checkAgainstWatchlist($name);
        
        $maxScore = 0;
        $matchedLists = [];
        
        foreach ($matches as $match) {
            $score = $this->calculateMatchScore($name, $match->name);
            if ($score > $maxScore) {
                $maxScore = $score;
            }
            if ($score > 70) {
                $matchedLists[] = $match->list_name;
            }
        }
        
        $status = $maxScore >= 90 ? 'confirmed_match' : ($maxScore >= 70 ? 'potential_match' : 'clear');
        
        $id = Capsule::table('mod_sanctions_screenings')->insertGetId([
            'screening_id' => $screeningId,
            'entity_name' => $name,
            'entity_type' => $entityType,
            'match_score' => $maxScore,
            'matched_lists' => json_encode($matchedLists),
            'status' => $status,
        ]);
        
        return [
            'screening_id' => $screeningId,
            'status' => $status,
            'match_score' => $maxScore,
            'matched_lists' => $matchedLists,
        ];
    }
    
    protected function checkAgainstWatchlist($name) {
        $normalizedName = strtolower(preg_replace('/[^a-z0-9]/', '', $name));
        
        $watchlist = Capsule::table('mod_sanctions_watchlist')->get();
        
        return $watchlist->filter(function($entry) use ($normalizedName) {
            $entryName = strtolower(preg_replace('/[^a-z0-9]/', '', $entry->name));
            similar_text($normalizedName, $entryName, $percent);
            return $percent > 60;
        });
    }
    
    protected function calculateMatchScore($inputName, $watchlistName) {
        similar_text(strtolower($inputName), strtolower($watchlistName), $percent);
        return $percent;
    }
}
```

## API Endpoints

```
POST /api/v1/sanctions/screen            - Screen entity
GET  /api/v1/sanctions/screening/{id}    - Get screening result
GET  /api/v1/sanctions/watchlist        - Get watchlist
POST /api/v1/sanctions/watchlist        - Add to watchlist
```

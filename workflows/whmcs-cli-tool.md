# WHMCS CLI Tool Workflow

## Description
Create command-line tools for WHMCS module management and operations.

## Prerequisites
- WHMCS 7.0+
- PHP 8.1+
- CLI access

## Steps

### Step 1: Create CLI Tool Structure
```bash
mkdir -p /var/www/whmcs/cli/clicodes_tools
mkdir -p /var/www/whmcs/cli/clicodes_tools/Commands
mkdir -p /var/www/whmcs/cli/clicodes_tools/Helpers
```

### Step 2: Create Main CLI Entry Point
```php
#!/usr/bin/env php
<?php
/**
 * WHMCS CLI Tool - CLICodes Tools
 * Usage: php clicodes.php <command> [options]
 */

define('WHMCS', true);
require_once __DIR__ . '/../init.php';

// CLI framework
class WHMCS_CLI {
    private $commands = [];
    
    public function register($name, $command) {
        $this->commands[$name] = $command;
    }
    
    public function run($argv) {
        $command = $argv[1] ?? 'help';
        $args = array_slice($argv, 2);
        
        if (!isset($this->commands[$command])) {
            $this->printHelp();
            exit(1);
        }
        
        return $this->commands[$command]->execute($args);
    }
    
    public function printHelp() {
        echo "WHMCS CLI Tools\n";
        echo "===============\n\n";
        echo "Usage: php clicodes.php <command> [options]\n\n";
        echo "Available commands:\n";
        
        foreach ($this->commands as $name => $command) {
            echo "  $name - " . $command->getDescription() . "\n";
        }
    }
}

// Base command class
abstract class Command {
    abstract public function getDescription();
    abstract public function execute($args);
    
    protected function output($message) {
        echo $message . "\n";
    }
    
    protected function error($message) {
        fwrite(STDERR, "ERROR: $message\n");
    }
    
    protected function success($message) {
        $this->output("\033[32m$message\033[0m");
    }
}

// Initialize CLI
$cli = new WHMCS_CLI();
```

### Step 3: Create Import Command
```php
<?php
/**
 * Import clients from CSV
 */

class ImportClientsCommand extends Command
{
    public function getDescription() {
        return 'Import clients from CSV file';
    }
    
    public function execute($args) {
        if (empty($args[0])) {
            $this->error("Usage: clicodes.php import:clients <file.csv>");
            return 1;
        }
        
        $file = $args[0];
        
        if (!file_exists($file)) {
            $this->error("File not found: $file");
            return 1;
        }
        
        $handle = fopen($file, 'r');
        $headers = fgetcsv($handle);
        
        $imported = 0;
        $errors = [];
        
        while (($row = fgetcsv($handle)) !== false) {
            $data = array_combine($headers, $row);
            
            try {
                $this->createClient($data);
                $imported++;
            } catch (Exception $e) {
                $errors[] = "Row $imported: " . $e->getMessage();
            }
        }
        
        fclose($handle);
        
        $this->success("Import complete: $imported clients imported");
        
        if (!empty($errors)) {
            $this->output("\nErrors:");
            foreach ($errors as $error) {
                $this->error($error);
            }
        }
        
        return empty($errors) ? 0 : 1;
    }
    
    private function createClient($data) {
        $required = ['firstname', 'lastname', 'email'];
        foreach ($required as $field) {
            if (empty($data[$field])) {
                throw new Exception("Missing required field: $field");
            }
        }
        
        $result = localAPI('AddClient', [
            'firstname' => $data['firstname'],
            'lastname' => $data['lastname'],
            'email' => $data['email'],
            'companyname' => $data['companyname'] ?? '',
            'address1' => $data['address1'] ?? '',
            'city' => $data['city'] ?? '',
            'state' => $data['state'] ?? '',
            'postcode' => $data['postcode'] ?? '',
            'country' => $data['country'] ?? 'US',
            'phonenumber' => $data['phonenumber'] ?? '',
            'password2' => $data['password'] ?? generatePassword(),
        ]);
        
        if ($result['result'] !== 'success') {
            throw new Exception($result['message'] ?? 'Failed to create client');
        }
        
        return $result['clientid'];
    }
}

// Register command
$cli->register('import:clients', new ImportClientsCommand());
```

### Step 4: Create Stats Command
```php
<?php
/**
 * Generate statistics report
 */

class StatsCommand extends Command
{
    public function getDescription() {
        return 'Generate business statistics report';
    }
    
    public function execute($args) {
        $period = $args[0] ?? 'month';
        
        $stats = $this->getStats($period);
        
        $this->output("=== WHMCS Statistics Report ===");
        $this->output("Period: " . ucfirst($period));
        $this->output("Generated: " . date('Y-m-d H:i:s'));
        $this->output("");
        
        $this->output("Revenue:");
        $this->output("  This period: " . formatCurrency($stats['revenue']));
        $this->output("  Invoices: " . $stats['invoices']);
        
        $this->output("");
        $this->output("Clients:");
        $this->output("  Total: " . number_format($stats['total_clients']));
        $this->output("  Active: " . number_format($stats['active_clients']));
        $this->output("  New this period: " . $stats['new_clients']);
        
        $this->output("");
        $this->output("Services:");
        $this->output("  Total: " . number_format($stats['total_services']));
        $this->output("  Active: " . number_format($stats['active_services']));
        
        $this->output("");
        $this->output("Support:");
        $this->output("  Open tickets: " . $stats['open_tickets']);
        $this->output("  Avg response time: " . $stats['avg_response_time'] . " hours");
        
        return 0;
    }
    
    private function getStats($period) {
        $dateFilter = match($period) {
            'day' => date('Y-m-d'),
            'week' => date('Y-m-d', strtotime('-7 days')),
            'month' => date('Y-m-01'),
            'year' => date('Y-01-01'),
            default => date('Y-m-01'),
        };
        
        return [
            'revenue' => Capsule::table('tblinvoices')
                ->where('status', 'Paid')
                ->where('date', '>=', $dateFilter)
                ->sum('total') ?? 0,
            'invoices' => Capsule::table('tblinvoices')
                ->where('status', 'Paid')
                ->where('date', '>=', $dateFilter)
                ->count(),
            'total_clients' => Capsule::table('tblclients')->count(),
            'active_clients' => Capsule::table('tblclients')
                ->where('status', 'Active')->count(),
            'new_clients' => Capsule::table('tblclients')
                ->where('created_at', '>=', $dateFilter)->count(),
            'total_services' => Capsule::table('tblhosting')->count(),
            'active_services' => Capsule::table('tblhosting')
                ->where('domainstatus', 'Active')->count(),
            'open_tickets' => Capsule::table('tbltickets')
                ->whereIn('status', ['Open', 'Awaiting Reply'])->count(),
            'avg_response_time' => '2.5',
        ];
    }
}

$cli->register('stats', new StatsCommand());
```

### Step 5: Create Cleanup Command
```php
<?php
/**
 * Cleanup command
 */

class CleanupCommand extends Command
{
    public function getDescription() {
        return 'Clean up old data and cache';
    }
    
    public function execute($args) {
        $action = $args[0] ?? 'all';
        
        switch ($action) {
            case 'cache':
                return $this->cleanCache();
            case 'temp':
                return $this->cleanTemp();
            case 'logs':
                return $this->cleanLogs();
            case 'sessions':
                return $this->cleanSessions();
            case 'all':
                $this->cleanCache();
                $this->cleanTemp();
                $this->cleanLogs();
                $this->cleanSessions();
                $this->success("All cleanup tasks completed");
                return 0;
            default:
                $this->error("Unknown cleanup action: $action");
                return 1;
        }
    }
    
    private function cleanCache() {
        $cleared = 0;
        
        $dirs = [
            ROOTDIR . '/templates_c',
            ROOTDIR . '/cache',
        ];
        
        foreach ($dirs as $dir) {
            if (is_dir($dir)) {
                $files = glob("$dir/*");
                foreach ($files as $file) {
                    if (is_file($file)) {
                        unlink($file);
                        $cleared++;
                    }
                }
            }
        }
        
        $this->success("Cleared $cleared cached files");
    }
    
    private function cleanTemp() {
        exec("rm -rf /tmp/whmcs_*");
        $this->success("Cleaned temp files");
    }
    
    private function cleanLogs() {
        $cutoff = date('Y-m-d', strtotime('-30 days'));
        
        $deleted = Capsule::table('tblactivitylog')
            ->where('date', '<', $cutoff)
            ->delete();
        
        $this->success("Deleted $deleted old log entries");
    }
    
    private function cleanSessions() {
        Capsule::table('tblsessions')
            ->where('lastvisit', '<', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->delete();
        
        $this->success("Cleaned expired sessions");
    }
}

$cli->register('cleanup', new CleanupCommand());
```

### Step 6: Create Main Executable
```php
<?php
// Run CLI
// Add to the end of clicodes.php

// Register commands
require_once __DIR__ . '/Commands/ImportClientsCommand.php';
require_once __DIR__ . '/Commands/StatsCommand.php';
require_once __DIR__ . '/Commands/CleanupCommand.php';

// Run
exit($cli->run($argv));
```

### Step 7: Create Wrapper Script
```bash
#!/bin/bash
# /usr/local/bin/whmcs-cli

cd /var/www/whmcs/cli
php clicodes.php "$@"
```

### Step 8: Usage Examples
```bash
# Import clients
php clicodes.php import:clients /path/to/clients.csv

# View statistics
php clicodes.php stats month
php clicodes.php stats week
php clicodes.php stats year

# Cleanup
php clicodes.php cleanup cache
php clicodes.php cleanup all

# Help
php clicodes.php
```

## Tags
- cli
- command-line
- automation
- tools
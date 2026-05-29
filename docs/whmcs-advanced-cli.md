# WHMCS Command Line Interface

Complete guide to CLI development for WHMCS.

## Overview

Create command-line tools for WHMCS automation.

## CLI Structure

### Basic CLI Entry Point

```php
<?php
/**
 * CLI entry point
 */
require_once __DIR__ . '/init.php';

// Get command from arguments
$command = $argv[1] ?? 'help';
$args = array_slice($argv, 2);

// Route command
$cli = new CLI();
$cli->run($command, $args);

/**
 * CLI Router
 */
class CLI
{
    private array $commands = [];
    
    public function __construct()
    {
        // Register commands
        $this->register('help', 'Show help information', [$this, 'showHelp']);
        $this->register('clients:list', 'List all clients', [$this, 'listClients']);
        $this->register('clients:create', 'Create a new client', [$this, 'createClient']);
        $this->register('services:sync', 'Sync services', [$this, 'syncServices']);
        $this->register('backup:run', 'Run backup', [$this, 'runBackup']);
    }
    
    /**
     * Run command
     */
    public function run(string $command, array $args): void
    {
        if (!isset($this->commands[$command])) {
            echo "Error: Unknown command '{$command}'\n";
            $this->showHelp();
            exit(1);
        }
        
        $handler = $this->commands[$command]['handler'];
        $handler($args);
    }
    
    /**
     * Register command
     */
    public function register(string $name, string $description, callable $handler): void
    {
        $this->commands[$name] = [
            'name' => $name,
            'description' => $description,
            'handler' => $handler,
        ];
    }
    
    /**
     * Show help
     */
    public function showHelp(): void
    {
        echo "Available commands:\n\n";
        
        foreach ($this->commands as $command) {
            echo sprintf("  %-20s %s\n", $command['name'], $command['description']);
        }
        
        echo "\n";
    }
}
```

## Client Commands

### List Clients

```php
<?php
/**
 * List clients command
 */
public function listClients(array $args): void
{
    $options = $this->parseOptions($args);
    
    $query = Capsule::table('tblclients');
    
    // Apply filters
    if (isset($options['status'])) {
        $query->where('status', $options['status']);
    }
    
    if (isset($options['search'])) {
        $search = $options['search'];
        $query->where(function($q) use ($search) {
            $q->where('email', 'LIKE', "%{$search}%")
              ->orWhere('firstname', 'LIKE', "%{$search}%")
              ->orWhere('lastname', 'LIKE', "%{$search}%");
        });
    }
    
    $clients = $query->limit($options['limit'] ?? 50)->get();
    
    // Output as table
    $this->outputTable(
        ['ID', 'Email', 'Name', 'Status', 'Created'],
        array_map(function($client) {
            return [
                $client->id,
                $client->email,
                "{$client->firstname} {$client->lastname}",
                $client->status,
                $client->datecreated,
            ];
        }, $clients->toArray())
    );
}

/**
 * Parse command options
 */
private function parseOptions(array $args): array
{
    $options = [];
    
    foreach ($args as $arg) {
        if (strpos($arg, '--') === 0) {
            $parts = explode('=', substr($arg, 2), 2);
            $options[$parts[0]] = $parts[1] ?? true;
        }
    }
    
    return $options;
}

/**
 * Output table
 */
private function outputTable(array $headers, array $rows): void
{
    // Calculate column widths
    $widths = array_map('strlen', $headers);
    
    foreach ($rows as $row) {
        foreach ($row as $i => $cell) {
            $widths[$i] = max($widths[$i], strlen((string)$cell));
        }
    }
    
    // Print headers
    $this->printRow($headers, $widths);
    $this->printSeparator($widths);
    
    // Print rows
    foreach ($rows as $row) {
        $this->printRow($row, $widths);
    }
}

private function printRow(array $row, array $widths): void
{
    foreach ($row as $i => $cell) {
        echo '| ' . str_pad((string)$cell, $widths[$i]) . ' ';
    }
    echo "|\n";
}

private function printSeparator(array $widths): void
{
    foreach ($widths as $width) {
        echo '+' . str_repeat('-', $width + 2);
    }
    echo "+\n";
}
```

### Create Client

```php
<?php
/**
 * Create client command
 */
public function createClient(array $args): void
{
    $data = $this->parseDataArgs($args);
    
    // Validate required fields
    $required = ['email', 'firstname', 'lastname'];
    foreach ($required as $field) {
        if (empty($data[$field])) {
            echo "Error: Missing required field: {$field}\n";
            exit(1);
        }
    }
    
    // Check if email exists
    $exists = Capsule::table('tblclients')
        ->where('email', $data['email'])
        ->exists();
    
    if ($exists) {
        echo "Error: Client with email {$data['email']} already exists\n";
        exit(1);
    }
    
    // Create client
    $clientId = Capsule::table('tblclients')->insertGetId([
        'email' => $data['email'],
        'firstname' => $data['firstname'],
        'lastname' => $data['lastname'],
        'companyname' => $data['company'] ?? '',
        'address1' => $data['address1'] ?? '',
        'city' => $data['city'] ?? '',
        'state' => $data['state'] ?? '',
        'postcode' => $data['postcode'] ?? '',
        'country' => $data['country'] ?? 'US',
        'phonenumber' => $data['phone'] ?? '',
        'password' => password_hash($data['password'] ?? bin2hex(random_bytes(8)), PASSWORD_DEFAULT),
        'datecreated' => date('Y-m-d'),
        'status' => 'Active',
    ]);
    
    echo "Client created successfully!\n";
    echo "Client ID: {$clientId}\n";
}

/**
 * Parse data arguments
 */
private function parseDataArgs(array $args): array
{
    $data = [];
    
    foreach ($args as $arg) {
        if (strpos($arg, '--') === 0) {
            $parts = explode('=', substr($arg, 2), 2);
            $data[$parts[0]] = $parts[1] ?? '';
        }
    }
    
    return $data;
}
```

## Service Commands

### Sync Services

```php
<?php
/**
 * Sync services command
 */
public function syncServices(array $args): void
{
    $options = $this->parseOptions($args);
    $force = isset($options['force']);
    
    echo "Starting service sync...\n";
    
    $services = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->get();
    
    $synced = 0;
    $failed = 0;
    
    foreach ($services as $service) {
        try {
            $this->syncService($service, $force);
            $synced++;
            
            if ($options['verbose'] ?? false) {
                echo "Synced: {$service->domain}\n";
            }
        } catch (Exception $e) {
            $failed++;
            echo "Failed: {$service->domain} - {$e->getMessage()}\n";
        }
    }
    
    echo "\nSync complete!\n";
    echo "Synced: {$synced}\n";
    echo "Failed: {$failed}\n";
}

/**
 * Sync single service
 */
private function syncService($service, bool $force): void
{
    // Get module
    $product = Capsule::table('tblproducts')
        ->where('id', $service->packageid)
        ->first();
    
    if (!$product || !$product->servertype) {
        throw new Exception('No module configured');
    }
    
    // Simulate module sync
    logActivity("Service {$service->id} synced via CLI");
    
    // Update last sync time
    Capsule::table('tblhosting')
        ->where('id', $service->id)
        ->update([
            'lastupdate' => date('Y-m-d H:i:s'),
        ]);
}
```

## Batch Commands

### Import Command

```php
<?php
/**
 * Import clients from CSV
 */
public function importClients(array $args): void
{
    $file = $args[0] ?? null;
    
    if (!$file || !file_exists($file)) {
        echo "Error: CSV file not found\n";
        exit(1);
    }
    
    echo "Importing clients from {$file}...\n";
    
    $handle = fopen($file, 'r');
    $headers = fgetcsv($handle);
    
    $imported = 0;
    $skipped = 0;
    $errors = 0;
    
    while (($row = fgetcsv($handle)) !== false) {
        $data = array_combine($headers, $row);
        
        try {
            // Check if exists
            $exists = Capsule::table('tblclients')
                ->where('email', $data['email'] ?? '')
                ->exists();
            
            if ($exists) {
                $skipped++;
                continue;
            }
            
            // Create client
            Capsule::table('tblclients')->insert([
                'email' => $data['email'],
                'firstname' => $data['first_name'] ?? '',
                'lastname' => $data['last_name'] ?? '',
                'companyname' => $data['company'] ?? '',
                'datecreated' => date('Y-m-d'),
                'status' => 'Active',
            ]);
            
            $imported++;
            
        } catch (Exception $e) {
            $errors++;
            echo "Error importing row: {$e->getMessage()}\n";
        }
    }
    
    fclose($handle);
    
    echo "\nImport complete!\n";
    echo "Imported: {$imported}\n";
    echo "Skipped: {$skipped}\n";
    echo "Errors: {$errors}\n";
}
```

## Cron Commands

```php
<?php
/**
 * Run scheduled tasks
 */
public function runCron(array $args): void
{
    $options = $this->parseOptions($args);
    
    echo "Running WHMCS cron tasks...\n";
    echo "Started at: " . date('Y-m-d H:i:s') . "\n\n";
    
    $tasks = [
        'invoices' => 'Process overdue invoices',
        'suspend' => 'Suspend overdue services',
        'terminate' => 'Terminate expired services',
        'backup' => 'Run backups',
        'cleanup' => 'Clean up temp files',
    ];
    
    foreach ($tasks as $task => $description) {
        if (isset($options['task']) && $options['task'] !== $task) {
            continue;
        }
        
        echo "[{$task}] {$description}... ";
        
        try {
            $start = microtime(true);
            $this->runTask($task);
            $duration = round((microtime(true) - $start) * 1000, 2);
            
            echo "Done ({$duration}ms)\n";
            
        } catch (Exception $e) {
            echo "FAILED - {$e->getMessage()}\n";
        }
    }
    
    echo "\nCompleted at: " . date('Y-m-d H:i:s') . "\n";
}

/**
 * Run specific task
 */
private function runTask(string $task): void
{
    switch ($task) {
        case 'invoices':
            // Invoice processing
            break;
        case 'suspend':
            // Suspension logic
            break;
        case 'terminate':
            // Termination logic
            break;
        case 'backup':
            $this->runBackup([]);
            break;
        case 'cleanup':
            $this->cleanupTempFiles();
            break;
    }
}
```

## Interactive Mode

```php
<?php
/**
 * Interactive mode
 */
public function interactiveMode(): void
{
    echo "WHMCS CLI - Interactive Mode\n";
    echo "Type 'help' for available commands, 'exit' to quit\n\n";
    
    while (true) {
        echo "> ";
        $input = trim(fgets(STDIN));
        
        if ($input === 'exit' || $input === 'quit') {
            break;
        }
        
        if (empty($input)) {
            continue;
        }
        
        $parts = explode(' ', $input);
        $command = $parts[0];
        $args = array_slice($parts, 1);
        
        try {
            $this->run($command, $args);
        } catch (Exception $e) {
            echo "Error: {$e->getMessage()}\n";
        }
    }
    
    echo "Goodbye!\n";
}
```

## Best Practices

1. **Help text** - Always provide helpful documentation
2. **Error handling** - Handle failures gracefully
3. **Progress indicators** - Show progress for long operations
4. **Verbose mode** - Allow detailed output
5. **Dry run** - Support preview mode
6. **Exit codes** - Use proper exit codes

## Related Documentation

- [whmcs-advanced-automation.md](whmcs-advanced-automation.md)
- [whmcs-integration-automation.md](whmcs-integration-automation.md)

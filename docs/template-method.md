# Template Method Pattern in WHMCS

The Template Method Pattern defines the skeleton of an algorithm in an operation, deferring some steps to subclasses. In WHMCS, this pattern is excellent for creating standardized processing workflows, report generation, and data import/export processes.

## Overview

Template method pattern defines the steps of an algorithm:
- Defines algorithm skeleton in base class
- Allows subclasses to override specific steps
- Prevents changing the algorithm structure
- Promotes code reuse
- Enforces consistent processing flow

## Core Structure

### Base Template Class

```php
<?php
// includes/TemplateMethod/AbstractProcessor.php

namespace CustomModule\TemplateMethod;

abstract class AbstractProcessor
{
    /**
     * Template method - defines the algorithm skeleton
     */
    final public function process(array $data): ProcessingResult
    {
        // Validate input
        if (!$this->validateInput($data)) {
            return $this->handleValidationError($data);
        }

        // Pre-processing
        $preparedData = $this->prepareData($data);

        // Main processing
        $processedData = $this->doProcess($preparedData);

        // Post-processing
        $result = $this->finalize($processedData);

        return $result;
    }

    protected function validateInput(array $data): bool
    {
        // Default validation - can be overridden
        return !empty($data);
    }

    protected function handleValidationError(array $data): ProcessingResult
    {
        return new ProcessingResult([
            'success' => false,
            'error' => 'Validation failed'
        ]);
    }

    protected function prepareData(array $data): array
    {
        // Default preparation - can be overridden
        return $data;
    }

    abstract protected function doProcess(array $data): array;

    protected function finalize(array $data): ProcessingResult
    {
        // Default finalization
        return new ProcessingResult([
            'success' => true,
            'data' => $data,
            'processed_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Real-World WHMCS Examples

### Data Import Template

```php
<?php
// includes/TemplateMethod/AbstractDataImporter.php

namespace CustomModule\TemplateMethod;

use WHMCS\Database\Capsule;

abstract class AbstractDataImporter
{
    protected int $importBatchSize = 100;
    protected int $totalRecords = 0;
    protected int $processedRecords = 0;
    protected int $failedRecords = 0;
    protected array $errors = [];
    protected array $warnings = [];

    /**
     * Template method for data import
     */
    final public function import(array $sourceData): ImportResult
    {
        $this->initialize();

        try {
            // Step 1: Validate source data
            if (!$this->validateSource($sourceData)) {
                return $this->createErrorResult('Source validation failed');
            }

            // Step 2: Transform data to standard format
            $transformedData = $this->transform($sourceData);

            // Step 3: Process in batches
            $result = $this->processInBatches($transformedData);

            // Step 4: Post-import cleanup
            $this->cleanup();

            return $result;
        } catch (\Exception $e) {
            $this->rollback();
            return $this->createErrorResult($e->getMessage());
        }
    }

    protected function initialize(): void
    {
        $this->totalRecords = 0;
        $this->processedRecords = 0;
        $this->failedRecords = 0;
        $this->errors = [];
        $this->warnings = [];

        $this->onInitialize();
    }

    protected function onInitialize(): void
    {
        // Hook for subclass-specific initialization
    }

    protected function validateSource(array $sourceData): bool
    {
        if (empty($sourceData)) {
            $this->errors[] = 'Empty source data';
            return false;
        }

        $this->totalRecords = count($sourceData);

        // Run custom validation
        return $this->validateSourceData($sourceData);
    }

    abstract protected function validateSourceData(array $sourceData): bool;

    protected function transform(array $sourceData): array
    {
        $transformed = [];

        foreach ($sourceData as $record) {
            $transformedRecord = $this->transformRecord($record);

            if ($transformedRecord !== null) {
                $transformed[] = $transformedRecord;
            }
        }

        return $transformed;
    }

    abstract protected function transformRecord(array $record): ?array;

    protected function processInBatches(array $data): ImportResult
    {
        $batches = array_chunk($data, $this->importBatchSize);
        $batchNumber = 0;

        foreach ($batches as $batch) {
            $batchNumber++;
            $this->processBatch($batch, $batchNumber);
        }

        return $this->createResult();
    }

    protected function processBatch(array $batch, int $batchNumber): void
    {
        $this->onBeforeBatch($batch, $batchNumber);

        foreach ($batch as $record) {
            try {
                $this->processRecord($record);
                $this->processedRecords++;
            } catch (\Exception $e) {
                $this->failedRecords++;
                $this->errors[] = [
                    'record' => $record,
                    'error' => $e->getMessage()
                ];
            }
        }

        $this->onAfterBatch($batch, $batchNumber);
    }

    abstract protected function processRecord(array $record): void;

    protected function onBeforeBatch(array $batch, int $batchNumber): void
    {
        // Hook for batch start processing
    }

    protected function onAfterBatch(array $batch, int $batchNumber): void
    {
        // Hook for batch completion
    }

    protected function cleanup(): void
    {
        $this->onCleanup();
    }

    protected function onCleanup(): void
    {
        // Hook for cleanup operations
    }

    protected function rollback(): void
    {
        // Default rollback - can be overridden
    }

    protected function createResult(): ImportResult
    {
        return new ImportResult([
            'success' => true,
            'total_records' => $this->totalRecords,
            'processed_records' => $this->processedRecords,
            'failed_records' => $this->failedRecords,
            'errors' => $this->errors,
            'warnings' => $this->warnings
        ]);
    }

    protected function createErrorResult(string $message): ImportResult
    {
        return new ImportResult([
            'success' => false,
            'error' => $message,
            'total_records' => $this->totalRecords,
            'processed_records' => $this->processedRecords,
            'failed_records' => $this->failedRecords
        ]);
    }
}
```

```php
<?php
// includes/TemplateMethod/ClientImporter.php

namespace CustomModule\TemplateMethod;

class ClientImporter extends AbstractDataImporter
{
    protected array $existingEmails = [];
    protected int $defaultClientGroup = 1;

    protected function onInitialize(): void
    {
        // Load existing emails for duplicate checking
        $this->existingEmails = Capsule::table('tblclients')
            ->pluck('email')
            ->toArray();
    }

    protected function validateSourceData(array $sourceData): bool
    {
        $requiredFields = ['email', 'firstname', 'lastname'];

        foreach ($sourceData as $index => $record) {
            foreach ($requiredFields as $field) {
                if (empty($record[$field])) {
                    $this->errors[] = "Row {$index}: Missing required field '{$field}'";
                }
            }

            // Validate email format
            if (!filter_var($record['email'] ?? '', FILTER_VALIDATE_EMAIL)) {
                $this->errors[] = "Row {$index}: Invalid email format";
            }
        }

        return empty($this->errors);
    }

    protected function transformRecord(array $record): ?array
    {
        return [
            'firstname' => trim($record['firstname']),
            'lastname' => trim($record['lastname']),
            'email' => strtolower(trim($record['email'])),
            'companyname' => $record['company'] ?? '',
            'phonenumber' => $record['phone'] ?? '',
            'address1' => $record['address'] ?? '',
            'city' => $record['city'] ?? '',
            'state' => $record['state'] ?? '',
            'country' => $record['country'] ?? 'US',
            'password' => $record['password'] ?? $this->generateTemporaryPassword(),
            'clientgroupid' => $record['group'] ?? $this->defaultClientGroup
        ];
    }

    protected function processRecord(array $record): void
    {
        // Check for duplicate email
        if (in_array($record['email'], $this->existingEmails)) {
            $this->warnings[] = "Skipping duplicate email: {$record['email']}";
            return;
        }

        // Hash password
        $record['password'] = $this->hashPassword($record['password']);
        $record['created_at'] = date('Y-m-d H:i:s');

        // Insert client
        $clientId = Capsule::table('tblclients')->insertGetId($record);

        // Set custom fields
        $this->setCustomFields($clientId, $record);

        // Log import
        $this->logImport($clientId, $record);
    }

    protected function hashPassword(string $password): string
    {
        return password_hash($password, PASSWORD_DEFAULT);
    }

    protected function setCustomFields(int $clientId, array $record): void
    {
        $customFields = ['custom_field_1', 'custom_field_2'];

        foreach ($customFields as $fieldName) {
            if (!empty($record[$fieldName])) {
                $field = Capsule::table('tblcustomfields')
                    ->where('fieldname', $fieldName)
                    ->where('type', 'client')
                    ->first();

                if ($field) {
                    Capsule::table('tblcustomfieldsvalues')->insert([
                        'relid' => $clientId,
                        'fieldid' => $field->id,
                        'value' => $record[$fieldName]
                    ]);
                }
            }
        }
    }

    protected function logImport(int $clientId, array $record): void
    {
        Capsule::table('mod_import_log')->insert([
            'type' => 'client',
            'record_id' => $clientId,
            'email' => $record['email'],
            'imported_at' => date('Y-m-d H:i:s')
        ]);
    }

    protected function generateTemporaryPassword(): string
    {
        return bin2hex(random_bytes(8));
    }
}
```

### Report Generation Template

```php
<?php
// includes/TemplateMethod/AbstractReportGenerator.php

namespace CustomModule\TemplateMethod;

abstract class AbstractReportGenerator
{
    protected string $reportTitle;
    protected array $filters = [];
    protected array $columns = [];
    protected int $pageSize = 50;
    protected string $outputFormat = 'html';

    final public function generate(): ReportResult
    {
        // Step 1: Validate and prepare
        if (!$this->validateConfiguration()) {
            return new ReportResult(['success' => false, 'error' => 'Invalid configuration']);
        }

        // Step 2: Fetch data
        $data = $this->fetchData();

        // Step 3: Process and format
        $formattedData = $this->formatData($data);

        // Step 4: Generate output
        $output = $this->generateOutput($formattedData);

        // Step 5: Finalize
        return $this->finalizeReport($output);
    }

    protected function validateConfiguration(): bool
    {
        return !empty($this->reportTitle);
    }

    abstract protected function fetchData(): array;

    protected function formatData(array $data): array
    {
        $formatted = [];

        foreach ($data as $row) {
            $formattedRow = [];

            foreach ($this->columns as $column) {
                $formattedRow[$column] = $this->formatCell($row, $column);
            }

            $formatted[] = $formattedRow;
        }

        return $formatted;
    }

    protected function formatCell(array $row, string $column): string
    {
        $value = $row[$column] ?? '';

        // Apply column-specific formatting
        return match($column) {
            'amount', 'total', 'price' => number_format($value, 2),
            'date', 'created_at' => date('Y-m-d', strtotime($value)),
            'status' => $this->formatStatus($value),
            default => $value
        };
    }

    protected function formatStatus(string $status): string
    {
        $colors = [
            'Active' => 'green',
            'Suspended' => 'orange',
            'Terminated' => 'red',
            'Pending' => 'yellow'
        ];

        return '<span class="status-' . ($colors[$status] ?? 'gray') . '">' . $status . '</span>';
    }

    abstract protected function generateOutput(array $data): string;

    protected function finalizeReport(string $output): ReportResult
    {
        return new ReportResult([
            'success' => true,
            'title' => $this->reportTitle,
            'output' => $output,
            'generated_at' => date('Y-m-d H:i:s'),
            'record_count' => $this->getRecordCount()
        ]);
    }

    protected function getRecordCount(): int
    {
        return 0; // Override in subclass
    }

    // Setter methods for configuration
    public function setTitle(string $title): self
    {
        $this->reportTitle = $title;
        return $this;
    }

    public function setFilters(array $filters): self
    {
        $this->filters = $filters;
        return $this;
    }

    public function setColumns(array $columns): self
    {
        $this->columns = $columns;
        return $this;
    }

    public function setPageSize(int $size): self
    {
        $this->pageSize = $size;
        return $this;
    }

    public function setOutputFormat(string $format): self
    {
        $this->outputFormat = $format;
        return $this;
    }
}
```

```php
<?php
// includes/TemplateMethod/SalesReportGenerator.php

namespace CustomModule\TemplateMethod;

use WHMCS\Database\Capsule;

class SalesReportGenerator extends AbstractReportGenerator
{
    protected string $dateRange;
    protected array $groupBy = [];

    protected function fetchData(): array
    {
        $query = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->where('date', '>=', $this->filters['from_date'] ?? date('Y-m-01'))
            ->where('date', '<=', $this->filters['to_date'] ?? date('Y-m-t'));

        if (!empty($this->filters['client_id'])) {
            $query->where('userid', $this->filters['client_id']);
        }

        $results = $query->get();

        return $results->map(function($invoice) {
            $client = Capsule::table('tblclients')
                ->where('id', $invoice->userid)
                ->first();

            return [
                'invoice_id' => $invoice->id,
                'invoice_num' => $invoice->invoicenum,
                'date' => $invoice->date,
                'client_name' => ($client->firstname ?? '') . ' ' . ($client->lastname ?? ''),
                'client_email' => $client->email ?? '',
                'amount' => $invoice->total,
                'payment_method' => $invoice->paymentmethod
            ];
        })->toArray();
    }

    protected function generateOutput(array $data): string
    {
        return match($this->outputFormat) {
            'html' => $this->generateHtml($data),
            'csv' => $this->generateCsv($data),
            'json' => json_encode($data),
            default => $this->generateHtml($data)
        };
    }

    protected function generateHtml(array $data): string
    {
        $html = '<table class="report-table">';
        $html .= '<thead><tr>';

        foreach ($this->columns as $column) {
            $html .= '<th>' . ucwords(str_replace('_', ' ', $column)) . '</th>';
        }

        $html .= '</tr></thead><tbody>';

        foreach ($data as $row) {
            $html .= '<tr>';
            foreach ($this->columns as $column) {
                $html .= '<td>' . htmlspecialchars($row[$column] ?? '') . '</td>';
            }
            $html .= '</tr>';
        }

        $html .= '</tbody></table>';

        return $html;
    }

    protected function generateCsv(array $data): string
    {
        $output = fopen('php://temp', 'r+');

        fputcsv($output, $this->columns);

        foreach ($data as $row) {
            $line = [];
            foreach ($this->columns as $column) {
                $line[] = $row[$column] ?? '';
            }
            fputcsv($output, $line);
        }

        rewind($output);
        $csv = stream_get_contents($output);
        fclose($output);

        return $csv;
    }

    protected function getRecordCount(): int
    {
        return count($this->fetchData());
    }
}
```

### Payment Processing Template

```php
<?php
// includes/TemplateMethod/AbstractPaymentProcessor.php

namespace CustomModule\TemplateMethod;

abstract class AbstractPaymentProcessor
{
    protected float $amount;
    protected string $currency;
    protected string $paymentMethod;
    protected array $metadata = [];

    final public function process(float $amount, string $currency, array $options = []): PaymentResult
    {
        $this->amount = $amount;
        $this->currency = $currency;
        $this->paymentMethod = $options['payment_method'] ?? 'default';
        $this->metadata = $options['metadata'] ?? [];

        // Step 1: Pre-process validation
        if (!$this->preProcessValidation()) {
            return $this->createFailedResult('Pre-process validation failed');
        }

        // Step 2: Prepare payment data
        $paymentData = $this->preparePaymentData();

        // Step 3: Execute payment
        $executionResult = $this->executePayment($paymentData);

        // Step 4: Handle result
        return $this->handlePaymentResult($executionResult);
    }

    protected function preProcessValidation(): bool
    {
        if ($this->amount <= 0) {
            return false;
        }

        if (strlen($this->currency) !== 3) {
            return false;
        }

        return $this->validatePaymentMethod();
    }

    abstract protected function validatePaymentMethod(): bool;

    protected function preparePaymentData(): array
    {
        return [
            'amount' => $this->amount,
            'currency' => $this->currency,
            'method' => $this->paymentMethod,
            'metadata' => $this->metadata,
            'timestamp' => date('Y-m-d H:i:s')
        ];
    }

    abstract protected function executePayment(array $paymentData): PaymentExecutionResult;

    protected function handlePaymentResult(PaymentExecutionResult $result): PaymentResult
    {
        if ($result->isSuccessful()) {
            return $this->handleSuccessfulPayment($result);
        }

        return $this->handleFailedPayment($result);
    }

    protected function handleSuccessfulPayment(PaymentExecutionResult $result): PaymentResult
    {
        // Log success
        $this->logTransaction($result, 'success');

        // Send confirmation
        $this->sendConfirmation($result);

        return new PaymentResult([
            'success' => true,
            'transaction_id' => $result->getTransactionId(),
            'amount' => $this->amount,
            'currency' => $this->currency
        ]);
    }

    protected function handleFailedPayment(PaymentExecutionResult $result): PaymentResult
    {
        // Log failure
        $this->logTransaction($result, 'failed');

        return new PaymentResult([
            'success' => false,
            'error' => $result->getErrorMessage(),
            'error_code' => $result->getErrorCode()
        ]);
    }

    protected function logTransaction(PaymentExecutionResult $result, string $status): void
    {
        Capsule::table('mod_payment_transactions')->insert([
            'transaction_id' => $result->getTransactionId(),
            'amount' => $this->amount,
            'currency' => $this->currency,
            'method' => $this->paymentMethod,
            'status' => $status,
            'metadata' => json_encode($this->metadata),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    protected function sendConfirmation(PaymentExecutionResult $result): void
    {
        // Send confirmation email
    }

    protected function createFailedResult(string $message): PaymentResult
    {
        return new PaymentResult([
            'success' => false,
            'error' => $message
        ]);
    }
}
```

### Usage Example

```php
<?php
// Using the template method pattern for client import

$importer = new ClientImporter();
$importer->setBatchSize(50);

$result = $importer->import([
    ['firstname' => 'John', 'lastname' => 'Doe', 'email' => 'john@example.com'],
    ['firstname' => 'Jane', 'lastname' => 'Smith', 'email' => 'jane@example.com']
]);

if ($result->isSuccess()) {
    echo "Imported {$result->getProcessedRecords()} records";
} else {
    echo "Import failed: {$result->getError()}";
}

// Using report generator
$report = new SalesReportGenerator();
$report->setTitle('Monthly Sales Report')
    ->setFilters(['from_date' => '2024-01-01', 'to_date' => '2024-01-31'])
    ->setColumns(['invoice_id', 'date', 'client_name', 'amount'])
    ->setOutputFormat('csv');

$result = $report->generate();
$csv = $result->getOutput();
```

## Pros

- **Code Reuse**: Common algorithm in base class
- **Consistency**: Ensures processing steps are followed
- **Extensibility**: Override specific steps without changing structure
- **Maintainability**: Centralized algorithm definition

## Cons

- **Rigidity**: Algorithm structure is fixed
- **Inheritance Coupling**: Relies on inheritance for customization
- **Complexity**: Can create deep class hierarchies

## Best Practices

1. Use final for template method to prevent override
2. Keep base class methods with default implementations
3. Use abstract methods for steps that must be overridden
4. Provide hook methods for optional customization
5. Document the algorithm steps clearly
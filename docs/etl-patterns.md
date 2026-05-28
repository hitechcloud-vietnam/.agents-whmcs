# ETL Patterns for Data Processing

ETL (Extract, Transform, Load) patterns provide systematic approaches for extracting data from sources, transforming it, and loading it into destinations in WHMCS modules.

## ETL Architecture

### ETL Builder

```php
<?php
/**
 * ETL Builder for constructing data pipelines
 */
class EtlBuilder
{
    private ?DataSourceInterface $source = null;
    private array $transformations = [];
    private ?DataDestinationInterface $destination = null;
    private array $validators = [];
    private array $options = [];

    /**
     * Set the data source
     */
    public function from(DataSourceInterface $source): self
    {
        $this->source = $source;
        return $this;
    }

    /**
     * Add a transformation
     */
    public function transform(callable $transformation): self
    {
        $this->transformations[] = $transformation;
        return $this;
    }

    /**
     * Set the destination
     */
    public function to(DataDestinationInterface $destination): self
    {
        $this->destination = $destination;
        return $this;
    }

    /**
     * Add a validator
     */
    public function validate(callable $validator): self
    {
        $this->validators[] = $validator;
        return $this;
    }

    /**
     * Set options
     */
    public function options(array $options): self
    {
        $this->options = $options;
        return $this;
    }

    /**
     * Execute the ETL process
     */
    public function run(): EtlResult
    {
        if (!$this->source) {
            throw new EtlException('Data source not set');
        }

        $result = new EtlResult();
        $startTime = microtime(true);

        try {
            // Extract
            $result->setExtracted($this->source->extract());
            $result->addLog('info', 'Extracted ' . count($result->getExtracted()) . ' records');

            // Validate before transform
            $this->runValidators($result);

            // Transform
            $data = $this->runTransformations($result->getExtracted());
            $result->setTransformed($data);
            $result->addLog('info', 'Transformed ' . count($data) . ' records');

            // Load
            if ($this->destination) {
                $loaded = $this->destination->load($data);
                $result->setLoaded($loaded);
                $result->addLog('info', 'Loaded ' . $loaded . ' records');
            }

            $result->setSuccess(true);
        } catch (\Throwable $e) {
            $result->setSuccess(false);
            $result->setError($e);
            $result->addLog('error', $e->getMessage());
        }

        $result->setDuration(microtime(true) - $startTime);

        return $result;
    }

    private function runValidators(EtlResult $result): void
    {
        foreach ($this->validators as $validator) {
            $errors = $validator($result->getExtracted());

            if (!empty($errors)) {
                $result->addValidationErrors($errors);
                $result->addLog('warning', 'Validation failed with ' . count($errors) . ' errors');
            }
        }
    }

    private function runTransformations(array $data): array
    {
        $transformed = $data;

        foreach ($this->transformations as $transformation) {
            $transformed = $transformation($transformed);

            if ($transformed === null) {
                throw new EtlException('Transformation returned null');
            }
        }

        return $transformed;
    }
}

/**
 * ETL Result
 */
class EtlResult
{
    private bool $success = false;
    private ?\Throwable $error = null;
    private array $extracted = [];
    private array $transformed = [];
    private int $loaded = 0;
    private float $duration = 0;
    private array $logs = [];
    private array $validationErrors = [];

    public function isSuccess(): bool
    {
        return $this->success;
    }

    public function setSuccess(bool $success): void
    {
        $this->success = $success;
    }

    public function getError(): ?\Throwable
    {
        return $this->error;
    }

    public function setError(\Throwable $error): void
    {
        $this->error = $error;
    }

    public function getExtracted(): array
    {
        return $this->extracted;
    }

    public function setExtracted(array $data): void
    {
        $this->extracted = $data;
    }

    public function getTransformed(): array
    {
        return $this->transformed;
    }

    public function setTransformed(array $data): void
    {
        $this->transformed = $data;
    }

    public function getLoaded(): int
    {
        return $this->loaded;
    }

    public function setLoaded(int $count): void
    {
        $this->loaded = $count;
    }

    public function getDuration(): float
    {
        return $this->duration;
    }

    public function setDuration(float $duration): void
    {
        $this->duration = $duration;
    }

    public function addLog(string $level, string $message): void
    {
        $this->logs[] = [
            'timestamp' => date('Y-m-d H:i:s'),
            'level' => $level,
            'message' => $message,
        ];
    }

    public function getLogs(): array
    {
        return $this->logs;
    }

    public function addValidationErrors(array $errors): void
    {
        $this->validationErrors = array_merge($this->validationErrors, $errors);
    }

    public function getValidationErrors(): array
    {
        return $this->validationErrors;
    }

    public function toArray(): array
    {
        return [
            'success' => $this->success,
            'error' => $this->error ? $this->error->getMessage() : null,
            'extracted_count' => count($this->extracted),
            'transformed_count' => count($this->transformed),
            'loaded_count' => $this->loaded,
            'duration_ms' => round($this->duration * 1000, 2),
            'logs' => $this->logs,
            'validation_errors' => $this->validationErrors,
        ];
    }
}

/**
 * ETL Exception
 */
class EtlException extends \Exception
{
}
```

## Data Sources

### API Data Source

```php
<?php
<?php
/**
 * API data source
 */
class ApiDataSource implements DataSourceInterface
{
    private string $baseUrl;
    private string $endpoint;
    private array $headers;
    private string $method;
    private array $body;
    private int $pageSize;
    private int $maxPages;

    public function __construct(string $baseUrl, string $endpoint)
    {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->endpoint = ltrim($endpoint, '/');
        $this->headers = [];
        $this->method = 'GET';
        $this->pageSize = 100;
        $this->maxPages = 100;
    }

    public function headers(array $headers): self
    {
        $this->headers = array_merge($this->headers, $headers);
        return $this;
    }

    public function method(string $method): self
    {
        $this->method = strtoupper($method);
        return $this;
    }

    public function body(array $body): self
    {
        $this->body = $body;
        return $this;
    }

    public function paginate(int $pageSize, int $maxPages = 100): self
    {
        $this->pageSize = $pageSize;
        $this->maxPages = $maxPages;
        return $this;
    }

    public function extract(): array
    {
        $allData = [];
        $page = 1;

        while ($page <= $this->maxPages) {
            $data = $this->fetchPage($page);

            if (empty($data)) {
                break;
            }

            $allData = array_merge($allData, $data);

            if (count($data) < $this->pageSize) {
                break;
            }

            $page++;
        }

        return $allData;
    }

    private function fetchPage(int $page): array
    {
        $url = $this->baseUrl . '/' . $this->endpoint;
        $params = ['page' => $page, 'per_page' => $this->pageSize];

        if ($this->method === 'GET') {
            $url .= '?' . http_build_query($params);
        }

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_CUSTOMREQUEST => $this->method,
        ]);

        if (!empty($this->headers)) {
            $curlHeaders = [];
            foreach ($this->headers as $key => $value) {
                $curlHeaders[] = "{$key}: {$value}";
            }
            curl_setopt($ch, CURLOPT_HTTPHEADER, $curlHeaders);
        }

        if (!empty($this->body) && $this->method !== 'GET') {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($this->body));
        }

        $response = curl_exec($ch);
        $statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($statusCode >= 400) {
            throw new EtlException("API returned status {$statusCode}");
        }

        $decoded = json_decode($response, true);

        // Handle pagination response formats
        if (isset($decoded['data'])) {
            return $decoded['data'];
        }

        if (is_array($decoded) && isset($decoded[0])) {
            return $decoded;
        }

        return [];
    }
}
```

### CSV Data Source

```php
<?php
/**
 * CSV file data source
 */
class CsvDataSource implements DataSourceInterface
{
    private string $filePath;
    private array $columns = [];
    private int $skipRows = 0;
    private string $delimiter = ',';
    private string $enclosure = '"';

    public function __construct(string $filePath)
    {
        $this->filePath = $filePath;
    }

    public function columns(array $columns): self
    {
        $this->columns = $columns;
        return $this;
    }

    public function skipRows(int $count): self
    {
        $this->skipRows = $count;
        return $this;
    }

    public function delimiter(string $delimiter): self
    {
        $this->delimiter = $delimiter;
        return $this;
    }

    public function extract(): array
    {
        if (!file_exists($this->filePath)) {
            throw new EtlException("File not found: {$this->filePath}");
        }

        $data = [];
        $handle = fopen($this->filePath, 'r');

        if ($handle === false) {
            throw new EtlException("Cannot open file: {$this->filePath}");
        }

        $rowIndex = 0;

        while (($row = fgetcsv($handle, 0, $this->delimiter, $this->enclosure)) !== false) {
            if ($rowIndex < $this->skipRows) {
                $rowIndex++;
                continue;
            }

            if (empty($this->columns)) {
                // First row is header
                if ($rowIndex === $this->skipRows) {
                    $this->columns = $row;
                    $rowIndex++;
                    continue;
                }
            }

            if (count($row) === count($this->columns)) {
                $record = [];

                foreach ($this->columns as $i => $column) {
                    $record[$column] = $row[$i] ?? null;
                }

                $data[] = $record;
            }

            $rowIndex++;
        }

        fclose($handle);

        return $data;
    }
}
```

## Data Destinations

### Bulk Insert Destination

```php
<?php
<?php
/**
 * Bulk insert destination
 */
class BulkInsertDestination implements DataDestinationInterface
{
    private string $table;
    private array $columnMapping = [];
    private int $batchSize = 500;
    private bool $ignoreDuplicates = false;
    private array $onDuplicateKeyUpdate = [];

    public function __construct(string $table)
    {
        $this->table = $table;
    }

    public function mapColumns(array $mapping): self
    {
        $this->columnMapping = $mapping;
        return $this;
    }

    public function batchSize(int $size): self
    {
        $this->batchSize = $size;
        return $this;
    }

    public function ignoreDuplicates(bool $ignore = true): self
    {
        $this->ignoreDuplicates = $ignore;
        return $this;
    }

    public function onDuplicateKeyUpdate(array $columns): self
    {
        $this->onDuplicateKeyUpdate = $columns;
        return $this;
    }

    public function load(array $data): int
    {
        if (empty($data)) {
            return 0;
        }

        $totalLoaded = 0;
        $batches = array_chunk($data, $this->batchSize);

        foreach ($batches as $batch) {
            $totalLoaded += $this->insertBatch($batch);
        }

        return $totalLoaded;
    }

    private function insertBatch(array $batch): int
    {
        $mappedBatch = array_map(function ($record) {
            return $this->mapRecord($record);
        }, $batch);

        // Build bulk insert query
        if (empty($mappedBatch)) {
            return 0;
        }

        $columns = array_keys($mappedBatch[0]);
        $placeholders = '(' . implode(', ', array_fill(0, count($columns), '?')) . ')';
        $allPlaceholders = implode(', ', array_fill(0, count($mappedBatch), $placeholders));

        $sql = "INSERT ";

        if ($this->ignoreDuplicates) {
            $sql .= "IGNORE ";
        }

        $sql .= "INTO {$this->table} (" . implode(', ', $columns) . ") VALUES " . $allPlaceholders;

        // Add ON DUPLICATE KEY UPDATE if specified
        if (!empty($this->onDuplicateKeyUpdate)) {
            $updates = [];

            foreach ($this->onDuplicateKeyUpdate as $column) {
                $updates[] = "{$column} = VALUES({$column})";
            }

            $sql .= " ON DUPLICATE KEY UPDATE " . implode(', ', $updates);
        }

        // Flatten values
        $values = [];
        foreach ($mappedBatch as $record) {
            foreach ($record as $value) {
                $values[] = $value;
            }
        }

        Capsule::connection()->affectingStatement($sql, $values);

        return count($batch);
    }

    private function mapRecord(array $record): array
    {
        if (empty($this->columnMapping)) {
            return $record;
        }

        $mapped = [];

        foreach ($this->columnMapping as $source => $dest) {
            $mapped[$dest] = $record[$source] ?? null;
        }

        return $mapped;
    }
}

/**
 * Data Destination Interface
 */
interface DataDestinationInterface
{
    public function load(array $data): int;
}
```

## Transformations

### Data Transformation Library

```php
<?php
<?php
/**
 * Common ETL transformations
 */
class EtlTransforms
{
    /**
     * Rename columns
     */
    public static function renameColumns(array $mapping): callable
    {
        return function (array $data) use ($mapping) {
            return array_map(function ($record) use ($mapping) {
                $renamed = [];

                foreach ($record as $key => $value) {
                    $newKey = $mapping[$key] ?? $key;
                    $renamed[$newKey] = $value;
                }

                return $renamed;
            }, $data);
        };
    }

    /**
     * Add computed column
     */
    public static function addColumn(string $columnName, callable $calculator): callable
    {
        return function (array $data) use ($columnName, $calculator) {
            return array_map(function ($record) use ($columnName, $calculator) {
                $record[$columnName] = $calculator($record);
                return $record;
            }, $data);
        };
    }

    /**
     * Remove columns
     */
    public static function removeColumns(array $columns): callable
    {
        return function (array $data) use ($columns) {
            return array_map(function ($record) use ($columns) {
                foreach ($columns as $column) {
                    unset($record[$column]);
                }
                return $record;
            }, $data);
        };
    }

    /**
     * Cast column types
     */
    public static function castColumn(string $column, string $type): callable
    {
        return function (array $data) use ($column, $type) {
            return array_map(function ($record) use ($column, $type) {
                if (isset($record[$column])) {
                    $record[$column] = self::cast($record[$column], $type);
                }
                return $record;
            }, $data);
        };
    }

    private static function cast($value, string $type)
    {
        switch ($type) {
            case 'int':
            case 'integer':
                return (int) $value;

            case 'float':
            case 'double':
                return (float) $value;

            case 'string':
                return (string) $value;

            case 'bool':
            case 'boolean':
                return (bool) $value;

            case 'date':
                return date('Y-m-d', strtotime($value));

            case 'datetime':
                return date('Y-m-d H:i:s', strtotime($value));

            default:
                return $value;
        }
    }

    /**
     * Format date
     */
    public static function formatDate(string $column, string $format = 'Y-m-d'): callable
    {
        return function (array $data) use ($column, $format) {
            return array_map(function ($record) use ($column, $format) {
                if (isset($record[$column]) && $record[$column]) {
                    $record[$column] = date($format, strtotime($record[$column]));
                }
                return $record;
            }, $data);
        };
    }

    /**
     * Trim strings
     */
    public static function trim(string $column): callable
    {
        return function (array $data) use ($column) {
            return array_map(function ($record) use ($column) {
                if (isset($record[$column])) {
                    $record[$column] = trim($record[$column]);
                }
                return $record;
            }, $data);
        };
    }

    /**
     * Lookup values from database
     */
    public static function lookup(string $column, string $lookupTable, string $lookupColumn, string $returnColumn): callable
    {
        return function (array $data) use ($column, $lookupTable, $lookupColumn, $returnColumn) {
            // Build lookup cache
            $lookupData = Capsule::table($lookupTable)
                ->pluck($returnColumn, $lookupColumn)
                ->toArray();

            return array_map(function ($record) use ($column, $lookupData, $returnColumn) {
                $lookupKey = $record[$column] ?? null;

                if ($lookupKey && isset($lookupData[$lookupKey])) {
                    $record[$returnColumn] = $lookupData[$lookupKey];
                }

                return $record;
            }, $data);
        };
    }

    /**
     * Merge columns
     */
    public static function mergeColumns(array $sourceColumns, string $targetColumn, string $separator = ' '): callable
    {
        return function (array $data) use ($sourceColumns, $targetColumn, $separator) {
            return array_map(function ($record) use ($sourceColumns, $targetColumn, $separator) {
                $values = [];

                foreach ($sourceColumns as $col) {
                    if (!empty($record[$col])) {
                        $values[] = $record[$col];
                    }
                }

                $record[$targetColumn] = implode($separator, $values);

                return $record;
            }, $data);
        };
    }

    /**
     * Split column
     */
    public static function splitColumn(string $sourceColumn, array $targetColumns): callable
    {
        return function (array $data) use ($sourceColumn, $targetColumns) {
            return array_map(function ($record) use ($sourceColumn, $targetColumns) {
                if (isset($record[$sourceColumn])) {
                    $parts = explode(' ', $record[$sourceColumn], count($targetColumns));

                    foreach ($targetColumns as $i => $target) {
                        $record[$target] = $parts[$i] ?? null;
                    }
                }

                return $record;
            }, $data);
        };
    }
}
```

## ETL Examples

### Customer Sync ETL

```php
<?php
/**
 * Customer synchronization ETL
 */
class CustomerSyncEtl
{
    public static function run(array $options = []): EtlResult
    {
        $apiSource = (new ApiDataSource(
            $options['api_url'],
            '/customers'
        ))->headers([
            'Authorization' => 'Bearer ' . $options['api_key'],
            'Content-Type' => 'application/json',
        ])->paginate(100, 50);

        $destination = (new BulkInsertDestination('tblclients'))
            ->mapColumns([
                'external_id' => 'external_id',
                'first_name' => 'firstname',
                'last_name' => 'lastname',
                'email_address' => 'email',
                'company_name' => 'companyname',
                'phone_number' => 'phonenumber',
            ])
            ->ignoreDuplicates(true)
            ->onDuplicateKeyUpdate(['firstname', 'lastname', 'companyname', 'phonenumber'])
            ->batchSize(500);

        return (new EtlBuilder())
            ->from($apiSource)
            ->validate(function ($data) {
                $errors = [];

                foreach ($data as $i => $record) {
                    if (empty($record['email_address'])) {
                        $errors[] = "Row {$i}: Email is required";
                    }

                    if (!filter_var($record['email_address'] ?? '', FILTER_VALIDATE_EMAIL)) {
                        $errors[] = "Row {$i}: Invalid email format";
                    }
                }

                return $errors;
            })
            ->transform(EtlTransforms::trim('first_name'))
            ->transform(EtlTransforms::trim('last_name'))
            ->transform(EtlTransforms::trim('email_address'))
            ->transform(EtlTransforms::castColumn('phone_number', 'string'))
            ->transform(function ($data) {
                return array_map(function ($record) {
                    $record['created_at'] = date('Y-m-d H:i:s');
                    $record['updated_at'] = date('Y-m-d H:i:s');
                    return $record;
                }, $data);
            })
            ->to($destination)
            ->run();
    }
}
```

## Best Practices

1. **Validate early** - Check data quality at the start
2. **Use transactions** - Ensure atomicity of loads
3. **Log everything** - Track ETL execution for debugging
4. **Handle duplicates** - Use upsert strategies
5. **Monitor performance** - Track extraction and load rates
6. **Recover from failures** - Implement restart capabilities
7. **Version your ETL** - Track changes over time
8. **Document transforms** - Make logic clear and maintainable

## Related Patterns

- [Data Pipeline](./data-pipeline.md) - General data processing
- [Queue Processing](./queue-processing.md) - Async ETL execution
- [Data Warehousing](./data-warehousing.md) - Data storage patterns

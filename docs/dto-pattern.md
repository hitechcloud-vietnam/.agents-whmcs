# Data Transfer Objects (DTO)

DTOs are simple objects used to transfer data between layers, decoupling internal structures from external interfaces.

## DTO Structure

### Base DTO Class

```php
<?php
/**
 * Base data transfer object
 */
abstract class DataTransferObject
{
    protected array $attributes = [];

    public function __construct(array $attributes = [])
    {
        $this->attributes = $attributes;
        $this->validate();
    }

    /**
     * Validate the DTO
     */
    protected function validate(): void
    {
        // Override in subclasses
    }

    /**
     * Get an attribute
     */
    public function get(string $key, $default = null)
    {
        return $this->attributes[$key] ?? $default;
    }

    /**
     * Set an attribute
     */
    public function set(string $key, $value): self
    {
        $this->attributes[$key] = $value;
        return $this;
    }

    /**
     * Check if attribute exists
     */
    public function has(string $key): bool
    {
        return isset($this->attributes[$key]);
    }

    /**
     * Get all attributes
     */
    public function all(): array
    {
        return $this->attributes;
    }

    /**
     * Convert to array
     */
    public function toArray(): array
    {
        return $this->attributes;
    }

    /**
     * Convert to JSON
     */
    public function toJson(): string
    {
        return json_encode($this->attributes);
    }

    /**
     * Create from array
     */
    public static function fromArray(array $data): static
    {
        return new static($data);
    }

    /**
     * Create from JSON
     */
    public static function fromJson(string $json): static
    {
        return new static(json_decode($json, true) ?? []);
    }
}
```

## Client DTOs

### Create Client DTO

```php
<?php
/**
 * DTO for creating a client
 */
class CreateClientDTO extends DataTransferObject
{
    private array $rules = [
        'firstname' => 'required|string|max:100',
        'lastname' => 'required|string|max:100',
        'email' => 'required|email',
        'phonenumber' => 'nullable|string|max:20',
        'companyname' => 'nullable|string|max:200',
        'address1' => 'nullable|string|max:255',
        'city' => 'nullable|string|max:100',
        'state' => 'nullable|string|max:100',
        'postcode' => 'nullable|string|max:20',
        'country' => 'nullable|string|size:2',
    ];

    protected function validate(): void
    {
        $errors = [];

        foreach ($this->rules as $field => $rule) {
            if (!isset($this->attributes[$field]) && str_contains($rule, 'required')) {
                $errors[] = "$field is required";
                continue;
            }

            $value = $this->attributes[$field] ?? null;

            if ($value !== null) {
                $ruleParts = explode('|', $rule);

                foreach ($ruleParts as $part) {
                    $result = $this->validateRule($field, $value, $part);
                    if ($result !== true) {
                        $errors[] = $result;
                    }
                }
            }
        }

        if (!empty($errors)) {
            throw new DTOValidationException($errors);
        }
    }

    private function validateRule(string $field, $value, string $rule): bool|string
    {
        switch ($rule) {
            case 'required':
                return !empty($value) ? true : "$field is required";

            case 'email':
                return filter_var($value, FILTER_VALIDATE_EMAIL)
                    ? true
                    : "$field must be a valid email";

            case str_starts_with($rule, 'max:'):
                $max = (int) substr($rule, 4);
                return strlen((string) $value) <= $max
                    ? true
                    : "$field must not exceed $max characters";

            case 'string':
                return is_string($value) ? true : "$field must be a string";

            case str_starts_with($rule, 'size:'):
                $size = (int) substr($rule, 5);
                return strlen((string) $value) === $size
                    ? true
                    : "$field must be exactly $size characters";

            default:
                return true;
        }
    }

    public function getFirstname(): string
    {
        return $this->attributes['firstname'];
    }

    public function getLastname(): string
    {
        return $this->attributes['lastname'];
    }

    public function getEmail(): string
    {
        return $this->attributes['email'];
    }

    public function getFullName(): string
    {
        return $this->getFirstname() . ' ' . $this->getLastname();
    }
}
```

### Order DTO

```php
<?php
/**
 * DTO for creating an order
 */
class CreateOrderDTO extends DataTransferObject
{
    protected function validate(): void
    {
        if (empty($this->attributes['client_id'])) {
            throw new DTOValidationException(['client_id is required']);
        }

        if (empty($this->attributes['items']) || !is_array($this->attributes['items'])) {
            throw new DTOValidationException(['items must be a non-empty array']);
        }

        foreach ($this->attributes['items'] as $index => $item) {
            if (!isset($item['product_id'])) {
                throw new DTOValidationException([
                    "items[$index].product_id is required"
                ]);
            }
        }
    }

    public function getClientId(): int
    {
        return (int) $this->attributes['client_id'];
    }

    public function getItems(): array
    {
        return $this->attributes['items'];
    }

    public function getPaymentMethod(): ?string
    {
        return $this->attributes['payment_method'] ?? null;
    }

    public function getNotes(): ?string
    {
        return $this->attributes['notes'] ?? null;
    }

    public function hasNotes(): bool
    {
        return !empty($this->attributes['notes']);
    }
}
```

## Response DTOs

### API Response DTO

```php
<?php
/**
 * API response DTO
 */
class ApiResponseDTO extends DataTransferObject
{
    private bool $success;
    private ?array $data;
    private ?array $errors;
    private ?array $meta;

    public function __construct(
        bool $success,
        ?array $data = null,
        ?array $errors = null,
        ?array $meta = null
    ) {
        $this->success = $success;
        $this->data = $data;
        $this->errors = $errors;
        $this->meta = $meta;

        parent::__construct([]);
    }

    public static function success($data = null, ?array $meta = null): self
    {
        return new self(true, $data, null, $meta);
    }

    public static function error($errors, ?array $meta = null): self
    {
        return new self(false, null, is_array($errors) ? $errors : [$errors], $meta);
    }

    public function isSuccess(): bool
    {
        return $this->success;
    }

    public function getData(): ?array
    {
        return $this->data;
    }

    public function getErrors(): ?array
    {
        return $this->errors;
    }

    public function getMeta(): ?array
    {
        return $this->meta;
    }

    public function toArray(): array
    {
        $response = ['success' => $this->success];

        if ($this->data !== null) {
            $response['data'] = $this->data;
        }

        if ($this->errors !== null) {
            $response['errors'] = $this->errors;
        }

        if ($this->meta !== null) {
            $response['meta'] = $this->meta;
        }

        return $response;
    }
}
```

### Paginated Response DTO

```php
<?php
/**
 * Paginated response DTO
 */
class PaginatedResponseDTO extends DataTransferObject
{
    private array $items;
    private int $total;
    private int $perPage;
    private int $currentPage;
    private int $lastPage;

    public function __construct(
        array $items,
        int $total,
        int $perPage,
        int $currentPage
    ) {
        $this->items = $items;
        $this->total = $total;
        $this->perPage = $perPage;
        $this->currentPage = $currentPage;
        $this->lastPage = (int) ceil($total / $perPage);

        parent::__construct([]);
    }

    public function getItems(): array
    {
        return $this->items;
    }

    public function getTotal(): int
    {
        return $this->total;
    }

    public function getPerPage(): int
    {
        return $this->perPage;
    }

    public function getCurrentPage(): int
    {
        return $this->currentPage;
    }

    public function getLastPage(): int
    {
        return $this->lastPage;
    }

    public function hasMorePages(): bool
    {
        return $this->currentPage < $this->lastPage;
    }

    public function toArray(): array
    {
        return [
            'data' => $this->items,
            'meta' => [
                'total' => $this->total,
                'per_page' => $this->perPage,
                'current_page' => $this->currentPage,
                'last_page' => $this->lastPage,
                'has_more' => $this->hasMorePages(),
            ],
        ];
    }
}
```

## Collection DTOs

### DTO Collection

```php
<?php
/**
 * Collection of DTOs
 */
class DTOCollection implements IteratorAggregate, Countable
{
    private array $items;
    private string $dtoClass;

    public function __construct(string $dtoClass, array $items = [])
    {
        $this->dtoClass = $dtoClass;
        $this->items = array_map(
            fn($item) => $item instanceof $dtoClass ? $item : new $dtoClass($item),
            $items
        );
    }

    public function getIterator(): ArrayIterator
    {
        return new ArrayIterator($this->items);
    }

    public function count(): int
    {
        return count($this->items);
    }

    public function isEmpty(): bool
    {
        return empty($this->items);
    }

    public function first(): ?DataTransferObject
    {
        return $this->items[0] ?? null;
    }

    public function last(): ?DataTransferObject
    {
        return empty($this->items) ? null : end($this->items);
    }

    public function map(callable $callback): array
    {
        return array_map($callback, $this->items);
    }

    public function filter(callable $callback): self
    {
        return new self(
            $this->dtoClass,
            array_filter($this->items, $callback)
        );
    }

    public function toArray(): array
    {
        return array_map(fn($item) => $item->toArray(), $this->items);
    }
}
```

## Mapper

### DTO Mapper

```php
<?php
/**
 * DTO mapper for converting entities to DTOs
 */
class DTOMapper
{
    private array $mappings = [];

    /**
     * Register a mapping
     */
    public function register(string $entityClass, string $dtoClass): void
    {
        $this->mappings[$entityClass] = $dtoClass;
    }

    /**
     * Map entity to DTO
     */
    public function map($entity, ?string $dtoClass = null): DataTransferObject
    {
        $dtoClass = $dtoClass ?? $this->mappings[get_class($entity)] ?? null;

        if (!$dtoClass) {
            throw new MapperException('No mapping found for ' . get_class($entity));
        }

        return new $dtoClass($entity->toArray());
    }

    /**
     * Map collection
     */
    public function mapCollection(array $entities, string $dtoClass): DTOCollection
    {
        return new DTOCollection($dtoClass, array_map(
            fn($e) => $e->toArray(),
            $entities
        ));
    }
}
```

## Usage Example

### Service with DTOs

```php
<?php
/**
 * Client service using DTOs
 */
class ClientService
{
    private ClientRepositoryInterface $repository;
    private DTOMapper $mapper;

    public function __construct(
        ClientRepositoryInterface $repository,
        DTOMapper $mapper
    ) {
        $this->repository = $repository;
        $this->mapper = $mapper;
    }

    /**
     * Create client from DTO
     */
    public function createFromDTO(CreateClientDTO $dto): Client
    {
        $client = Client::create([
            'firstname' => $dto->getFirstname(),
            'lastname' => $dto->getLastname(),
            'email' => $dto->getEmail(),
            'phonenumber' => $dto->get('phonenumber'),
            'companyname' => $dto->get('companyname'),
            'address1' => $dto->get('address1'),
            'city' => $dto->get('city'),
            'state' => $dto->get('state'),
            'postcode' => $dto->get('postcode'),
            'country' => $dto->get('country'),
        ]);

        $this->repository->save($client);

        return $client;
    }

    /**
     * Get clients as paginated response
     */
    public function getClientsPaginated(int $page = 1, int $perPage = 20): PaginatedResponseDTO
    {
        $clients = $this->repository->findAll([], $perPage, ($page - 1) * $perPage);
        $total = $this->repository->count();

        $items = array_map(
            fn($c) => $this->mapper->map($c, ClientDTO::class)->toArray(),
            $clients
        );

        return new PaginatedResponseDTO($items, $total, $perPage, $page);
    }
}
```

## Best Practices

1. **Keep DTOs simple** - No business logic, just data
2. **Validate early** - Validate in constructor
3. **Immutable by default** - Use readonly where possible
4. **Type hints** - Use PHP type declarations
5. **Document fields** - Add docblock annotations
6. **Use collections** - Group related DTOs
7. **Separate read/write** - Different DTOs for input/output

## Related Patterns

- [Value Objects](./value-objects.md) - Immutable domain values
- [Service Layer](./service-layer.md) - Using DTOs in services
- [Repository Pattern](./repository-pattern.md) - Entity to DTO mapping
# Value Objects in PHP

Value Objects are immutable objects that represent descriptive aspects of the domain with no identity.

## Value Object Base

### Base Value Object

```php
<?php
/**
 * Base value object
 */
abstract class ValueObject
{
    protected $value;

    public function __construct($value)
    {
        $this->value = $this->validate($value);
    }

    /**
     * Validate the value
     */
    abstract protected function validate($value): mixed;

    /**
     * Get the value
     */
    public function getValue()
    {
        return $this->value;
    }

    /**
     * Check equality
     */
    public function equals(ValueObject $other): bool
    {
        return static::class === get_class($other) && $this->value === $other->value;
    }

    /**
     * Convert to string
     */
    public function __toString(): string
    {
        return (string) $this->value;
    }

    /**
     * Serialize
     */
    public function toArray(): array
    {
        return ['value' => $this->value];
    }

    /**
     * Deserialize
     */
    public static function fromArray(array $data): static
    {
        return new static($data['value']);
    }
}
```

## Common Value Objects

### Email Value Object

```php
<?php
/**
 * Email value object
 */
class Email extends ValueObject
{
    protected function validate($value): string
    {
        $value = trim(strtolower($value));

        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new ValueObjectException('Invalid email address: ' . $value);
        }

        return $value;
    }

    /**
     * Get domain part
     */
    public function getDomain(): string
    {
        return substr($this->value, strpos($this->value, '@') + 1);
    }

    /**
     * Get local part
     */
    public function getLocalPart(): string
    {
        return substr($this->value, 0, strpos($this->value, '@'));
    }

    /**
     * Check if is from specific domain
     */
    public function isFromDomain(string $domain): bool
    {
        return $this->getDomain() === strtolower($domain);
    }
}
```

### Money Value Object

```php
<?php
/**
 * Money value object
 */
class Money
{
    private float $amount;
    private string $currency;

    private const DECIMAL_PLACES = 2;

    public function __construct(float $amount, string $currency = 'USD')
    {
        $this->validateCurrency($currency);

        $this->amount = round($amount, self::DECIMAL_PLACES);
        $this->currency = strtoupper($currency);
    }

    public function getAmount(): float
    {
        return $this->amount;
    }

    public function getCurrency(): string
    {
        return $this->currency;
    }

    public function add(Money $other): Money
    {
        $this->assertSameCurrency($other);
        return new Money($this->amount + $other->amount, $this->currency);
    }

    public function subtract(Money $other): Money
    {
        $this->assertSameCurrency($other);
        return new Money($this->amount - $other->amount, $this->currency);
    }

    public function multiply(float $factor): Money
    {
        return new Money($this->amount * $factor, $this->currency);
    }

    public function isZero(): bool
    {
        return $this->amount === 0.0;
    }

    public function isPositive(): bool
    {
        return $this->amount > 0;
    }

    public function isNegative(): bool
    {
        return $this->amount < 0;
    }

    public function equals(Money $other): bool
    {
        return $this->currency === $other->currency
            && $this->amount === $other->amount;
    }

    public function greaterThan(Money $other): bool
    {
        $this->assertSameCurrency($other);
        return $this->amount > $other->amount;
    }

    public function lessThan(Money $other): bool
    {
        $this->assertSameCurrency($other);
        return $this->amount < $other->amount;
    }

    public function format(): string
    {
        return number_format($this->amount, self::DECIMAL_PLACES);
    }

    public function __toString(): string
    {
        return $this->currency . ' ' . $this->format();
    }

    public function toArray(): array
    {
        return [
            'amount' => $this->amount,
            'currency' => $this->currency,
        ];
    }

    public static function fromArray(array $data): Money
    {
        return new Money($data['amount'], $data['currency']);
    }

    private function assertSameCurrency(Money $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new ValueObjectException(
                "Cannot operate on different currencies: {$this->currency} and {$other->currency}"
            );
        }
    }

    private function validateCurrency(string $currency): void
    {
        if (strlen($currency) !== 3) {
            throw new ValueObjectException('Currency must be 3 characters');
        }
    }

    public static function zero(string $currency = 'USD'): Money
    {
        return new Money(0.0, $currency);
    }
}
```

### Phone Number Value Object

```php
<?php
/**
 * Phone number value object
 */
class PhoneNumber extends ValueObject
{
    private string $countryCode;
    private string $number;

    public function __construct(string $phoneNumber, ?string $countryCode = null)
    {
        $parsed = $this->parse($phoneNumber, $countryCode);
        $this->countryCode = $parsed['country_code'];
        $this->number = $parsed['number'];

        $this->validate($this->countryCode . $this->number);
    }

    public function getCountryCode(): string
    {
        return $this->countryCode;
    }

    public function getNumber(): string
    {
        return $this->number;
    }

    public function getInternationalFormat(): string
    {
        return '+' . $this->countryCode . $this->number;
    }

    public function getNationalFormat(): string
    {
        // Would need country-specific formatting
        return $this->number;
    }

    protected function validate($value): string
    {
        $digits = preg_replace('/[^0-9]/', '', $value);

        if (strlen($digits) < 7 || strlen($digits) > 15) {
            throw new ValueObjectException(
                'Phone number must be between 7 and 15 digits'
            );
        }

        return $value;
    }

    private function parse(string $phoneNumber, ?string $countryCode): array
    {
        // Remove spaces and special characters
        $clean = preg_replace('/[\s\-\(\)]+/', '', $phoneNumber);

        // Check for leading +
        if (str_starts_with($clean, '+')) {
            // International format
            $digits = substr($clean, 1);

            // Extract country code (1-3 digits)
            if (strlen($digits) >= 11) {
                return [
                    'country_code' => substr($digits, 0, 1),
                    'number' => substr($digits, 1),
                ];
            }

            return [
                'country_code' => substr($digits, 0, 2),
                'number' => substr($digits, 2),
            ];
        }

        // No country code specified
        return [
            'country_code' => $countryCode ?? '1',
            'number' => $clean,
        ];
    }
}
```

### Date Range Value Object

```php
<?php
/**
 * Date range value object
 */
class DateRange
{
    private DateTimeImmutable $start;
    private DateTimeImmutable $end;

    public function __construct(DateTimeImmutable $start, DateTimeImmutable $end)
    {
        if ($start > $end) {
            throw new ValueObjectException('Start date must be before or equal to end date');
        }

        $this->start = $start;
        $this->end = $end;
    }

    public function getStart(): DateTimeImmutable
    {
        return $this->start;
    }

    public function getEnd(): DateTimeImmutable
    {
        return $this->end;
    }

    public function contains(DateTimeImmutable $date): bool
    {
        return $date >= $this->start && $date <= $this->end;
    }

    public function overlaps(DateRange $other): bool
    {
        return $this->start <= $other->end && $this->end >= $other->start;
    }

    public function getDurationInDays(): int
    {
        return $this->start->diff($this->end)->days;
    }

    public function getDurationInSeconds(): int
    {
        return $this->end->getTimestamp() - $this->start->getTimestamp();
    }

    public function equals(DateRange $other): bool
    {
        return $this->start == $other->start && $this->end == $other->end;
    }

    public function __toString(): string
    {
        return $this->start->format('Y-m-d') . ' to ' . $this->end->format('Y-m-d');
    }

    public function toArray(): array
    {
        return [
            'start' => $this->start->format('Y-m-d H:i:s'),
            'end' => $this->end->format('Y-m-d H:i:s'),
        ];
    }

    public static function fromArray(array $data): DateRange
    {
        return new self(
            new DateTimeImmutable($data['start']),
            new DateTimeImmutable($data['end'])
        );
    }

    public static function createDaysFromNow(int $days): self
    {
        $now = new DateTimeImmutable();
        return new self($now, $now->modify("+{$days} days"));
    }

    public static function lastNDays(int $days): self
    {
        $now = new DateTimeImmutable();
        return new self($now->modify("-{$days} days"), $now);
    }

    public static function today(): self
    {
        $today = new DateTimeImmutable('today');
        return new self($today, $today->modify('+1 day')->modify('-1 second'));
    }

    public static function thisMonth(): self
    {
        $start = new DateTimeImmutable('first day of this month');
        $end = new DateTimeImmutable('last day of this month')->modify('23:59:59');
        return new self($start, $end);
    }
}
```

## Composite Value Objects

### Address Value Object

```php
<?php
/**
 * Address value object
 */
class Address implements JsonSerializable
{
    private string $street;
    private ?string $street2;
    private string $city;
    private string $state;
    private string $postcode;
    private string $country;
    private ?string $company;

    public function __construct(
        string $street,
        string $city,
        string $state,
        string $postcode,
        string $country,
        ?string $street2 = null,
        ?string $company = null
    ) {
        $this->street = $street;
        $this->street2 = $street2;
        $this->city = $city;
        $this->state = $state;
        $this->postcode = $postcode;
        $this->country = strtoupper($country);
        $this->company = $company;
    }

    public function getStreet(): string
    {
        return $this->street;
    }

    public function getStreet2(): ?string
    {
        return $this->street2;
    }

    public function getCity(): string
    {
        return $this->city;
    }

    public function getState(): string
    {
        return $this->state;
    }

    public function getPostcode(): string
    {
        return $this->postcode;
    }

    public function getCountry(): string
    {
        return $this->country;
    }

    public function getCompany(): ?string
    {
        return $this->company;
    }

    public function hasCompany(): bool
    {
        return !empty($this->company);
    }

    public function hasStreet2(): bool
    {
        return !empty($this->street2);
    }

    public function getFullAddress(): string
    {
        $parts = [];

        if ($this->company) {
            $parts[] = $this->company;
        }

        $parts[] = $this->street;

        if ($this->street2) {
            $parts[] = $this->street2;
        }

        $parts[] = $this->city . ', ' . $this->state . ' ' . $this->postcode;
        $parts[] = $this->country;

        return implode("\n", $parts);
    }

    public function equals(Address $other): bool
    {
        return $this->street === $other->street
            && $this->street2 === $other->street2
            && $this->city === $other->city
            && $this->state === $other->state
            && $this->postcode === $other->postcode
            && $this->country === $other->country
            && $this->company === $other->company;
    }

    public function toArray(): array
    {
        return [
            'street' => $this->street,
            'street2' => $this->street2,
            'city' => $this->city,
            'state' => $this->state,
            'postcode' => $this->postcode,
            'country' => $this->country,
            'company' => $this->company,
        ];
    }

    public static function fromArray(array $data): self
    {
        return new self(
            $data['street'],
            $data['city'],
            $data['state'],
            $data['postcode'],
            $data['country'],
            $data['street2'] ?? null,
            $data['company'] ?? null
        );
    }

    public function jsonSerialize(): array
    {
        return $this->toArray();
    }
}
```

## Custom Value Objects

### Client ID Value Object

```php
<?php
/**
 * Client ID value object
 */
class ClientId extends ValueObject
{
    protected function validate($value): int
    {
        $value = (int) $value;

        if ($value <= 0) {
            throw new ValueObjectException('Client ID must be a positive integer');
        }

        return $value;
    }
}
```

### Order ID Value Object

```php
<?php
/**
 * Order ID value object
 */
class OrderId extends ValueObject
{
    protected function validate($value): int
    {
        $value = (int) $value;

        if ($value <= 0) {
            throw new ValueObjectException('Order ID must be a positive integer');
        }

        return $value;
    }
}
```

## Value Object Collections

### Immutable Collection

```php
<?php
/**
 * Immutable collection of value objects
 */
class ImmutableCollection implements IteratorAggregate, Countable, JsonSerializable
{
    private array $items;

    public function __construct(array $items = [])
    {
        $this->items = array_values($items);
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

    public function get(int $index): mixed
    {
        return $this->items[$index] ?? null;
    }

    public function contains(mixed $item): bool
    {
        return in_array($item, $this->items, true);
    }

    public function map(callable $callback): array
    {
        return array_map($callback, $this->items);
    }

    public function filter(callable $callback): self
    {
        return new self(array_filter($this->items, $callback));
    }

    public function toArray(): array
    {
        return array_map(
            fn($item) => $item instanceof ValueObject ? $item->toArray() : $item,
            $this->items
        );
    }

    public function jsonSerialize(): array
    {
        return $this->toArray();
    }
}
```

## Best Practices

1. **Make them immutable** - No setters, create new instances
2. **Implement equality** - Compare by value, not identity
3. **Validate in constructor** - Reject invalid values
4. **Use for identity-less concepts** - Emails, money, dates, addresses
5. **Keep them small** - Single responsibility
6. **Add helpful methods** - Business-relevant operations
7. **Implement toArray/fromArray** - For serialization

## Related Patterns

- [DTO Pattern](./dto-pattern.md) - Data transfer objects
- [Service Layer](./service-layer.md) - Using value objects in services
- [Entity Design](./entity-design.md) - Entities vs Value Objects
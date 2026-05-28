# Interpreter Pattern in WHMCS

The Interpreter Pattern defines a representation for a grammar and an interpreter to handle sentences in this language. In WHMCS, this pattern is useful for building configuration languages, query builders, rule engines, and DSLs for business logic.

## Overview

Interpreter pattern defines a grammar and an interpreter for that grammar:
- Define a language grammar
- Represent sentences in the language
- Interpret sentences using the grammar
- Handle complex expression evaluation
- Build extensible query systems

## Core Structure

### Abstract Expression

```php
<?php
// includes/Interpreter/Expression.php

namespace CustomModule\Interpreter;

interface Expression
{
    public function interpret(Context $context): mixed;
    public function toString(): string;
}
```

## Real-World WHMCS Examples

### Query Builder DSL

```php
<?php
// includes/Interpreter/QueryContext.php

namespace CustomModule\Interpreter;

use WHMCS\Database\Capsule;

class QueryContext
{
    protected array $bindings = [];
    protected array $results = [];

    public function set(string $key, $value): void
    {
        $this->bindings[$key] = $value;
    }

    public function get(string $key, $default = null)
    {
        return $this->bindings[$key] ?? $default;
    }

    public function setResults(array $results): void
    {
        $this->results = $results;
    }

    public function getResults(): array
    {
        return $this->results;
    }
}
```

```php
<?php
// includes/Interpreter/Expressions/FieldExpression.php

namespace CustomModule\Interpreter\Expressions;

use CustomModule\Interpreter\Expression;
use CustomModule\Interpreter\QueryContext;

class FieldExpression implements Expression
{
    protected string $field;
    protected string $operator;
    protected $value;

    public function __construct(string $field, string $operator, $value)
    {
        $this->field = $field;
        $this->operator = $operator;
        $this->value = $value;
    }

    public function interpret(QueryContext $context): bool
    {
        $fieldValue = $context->get('record:' . $this->field);
        $searchValue = $this->resolveValue($context);

        return $this->evaluate($fieldValue, $searchValue);
    }

    protected function evaluate($fieldValue, $searchValue): bool
    {
        return match($this->operator) {
            '=' => $fieldValue == $searchValue,
            '!=' => $fieldValue != $searchValue,
            '>' => $fieldValue > $searchValue,
            '<' => $fieldValue < $searchValue,
            '>=', 'gte' => $fieldValue >= $searchValue,
            '<=', 'lte' => $fieldValue <= $searchValue,
            'contains' => strpos($fieldValue, $searchValue) !== false,
            'starts_with' => strpos($fieldValue, $searchValue) === 0,
            'ends_with' => substr($fieldValue, -strlen($searchValue)) === $searchValue,
            'in' => in_array($fieldValue, (array)$searchValue),
            'like' => $this->likeMatch($fieldValue, $searchValue),
            default => false
        };
    }

    protected function likeMatch(string $value, string $pattern): bool
    {
        $regex = str_replace(['%', '_'], ['.*', '.'], preg_quote($pattern, '/'));
        return preg_match("/^{$regex}$/i", $value) === 1;
    }

    protected function resolveValue(QueryContext $context): mixed
    {
        if (is_string($this->value) && strpos($this->value, '$') === 0) {
            return $context->get(substr($this->value, 1));
        }
        return $this->value;
    }

    public function toString(): string
    {
        return "{$this->field} {$this->operator} {$this->value}";
    }
}
```

```php
<?php
// includes/Interpreter/Expressions/AndExpression.php

namespace CustomModule\Interpreter\Expressions;

use CustomModule\Interpreter\Expression;
use CustomModule\Interpreter\QueryContext;

class AndExpression implements Expression
{
    protected Expression $left;
    protected Expression $right;

    public function __construct(Expression $left, Expression $right)
    {
        $this->left = $left;
        $this->right = $right;
    }

    public function interpret(QueryContext $context): bool
    {
        return $this->left->interpret($context) && $this->right->interpret($context);
    }

    public function toString(): string
    {
        return "({$this->left->toString()} AND {$this->right->toString()})";
    }
}
```

```php
<?php
// includes/Interpreter/Expressions/OrExpression.php

namespace CustomModule\Interpreter\Expressions;

use CustomModule\Interpreter\Expression;
use CustomModule\Interpreter\QueryContext;

class OrExpression implements Expression
{
    protected Expression $left;
    protected Expression $right;

    public function __construct(Expression $left, Expression $right)
    {
        $this->left = $left;
        $this->right = $right;
    }

    public function interpret(QueryContext $context): bool
    {
        return $this->left->interpret($context) || $this->right->interpret($context);
    }

    public function toString(): string
    {
        return "({$this->left->toString()} OR {$this->right->toString()})";
    }
}
```

```php
<?php
// includes/Interpreter/Expressions/NotExpression.php

namespace CustomModule\Interpreter\Expressions;

use CustomModule\Interpreter\Expression;
use CustomModule\Interpreter\QueryContext;

class NotExpression implements Expression
{
    protected Expression $expression;

    public function __construct(Expression $expression)
    {
        $this->expression = $expression;
    }

    public function interpret(QueryContext $context): bool
    {
        return !$this->expression->interpret($context);
    }

    public function toString(): string
    {
        return "NOT ({$this->expression->toString()})";
    }
}
```

### Expression Parser

```php
<?php
// includes/Interpreter/ExpressionParser.php

namespace CustomModule\Interpreter;

class ExpressionParser
{
    protected string $expression;
    protected int $position = 0;

    public function __construct(string $expression)
    {
        $this->expression = trim($expression);
    }

    public function parse(): Expression
    {
        return $this->parseOr();
    }

    protected function parseOr(): Expression
    {
        $left = $this->parseAnd();

        while ($this->skipKeyword('OR')) {
            $right = $this->parseAnd();
            $left = new Expressions\OrExpression($left, $right);
        }

        return $left;
    }

    protected function parseAnd(): Expression
    {
        $left = $this->parseNot();

        while ($this->skipKeyword('AND')) {
            $right = $this->parseNot();
            $left = new Expressions\AndExpression($left, $right);
        }

        return $left;
    }

    protected function parseNot(): Expression
    {
        if ($this->skipKeyword('NOT')) {
            $expression = $this->parsePrimary();
            return new Expressions\NotExpression($expression);
        }

        return $this->parsePrimary();
    }

    protected function parsePrimary(): Expression
    {
        // Skip opening parenthesis
        if ($this->skipChar('(')) {
            $expression = $this->parseOr();
            $this->skipChar(')');
            return $expression;
        }

        // Parse field comparison
        return $this->parseComparison();
    }

    protected function parseComparison(): Expression
    {
        $field = $this->parseIdentifier();
        $this->skipWhitespace();

        // Get operator
        $operator = $this->parseOperator();

        $this->skipWhitespace();

        // Get value
        $value = $this->parseValue();

        return new Expressions\FieldExpression($field, $operator, $value);
    }

    protected function parseIdentifier(): string
    {
        $this->skipWhitespace();

        $start = $this->position;
        while ($this->position < strlen($this->expression) &&
               preg_match('/[a-zA-Z0-9_.]/', $this->expression[$this->position])) {
            $this->position++;
        }

        return substr($this->expression, $start, $this->position - $start);
    }

    protected function parseOperator(): string
    {
        $this->skipWhitespace();

        // Multi-char operators
        if ($this->lookAhead('>=')) {
            $this->position += 2;
            return '>=';
        }
        if ($this->lookAhead('<=')) {
            $this->position += 2;
            return '<=';
        }
        if ($this->lookAhead('!=')) {
            $this->position += 2;
            return '!=';
        }
        if ($this->lookAhead('==')) {
            $this->position += 2;
            return '=';
        }

        // Single char operators
        $op = $this->expression[$this->position] ?? '';
        if (in_array($op, ['=', '>', '<'])) {
            $this->position++;
            return $op;
        }

        return '=';
    }

    protected function parseValue(): mixed
    {
        $this->skipWhitespace();

        // String value
        if ($this->lookAhead('"') || $this->lookAhead("'")) {
            return $this->parseString();
        }

        // Boolean
        if ($this->lookAheadKeyword('true')) {
            $this->position += 4;
            return true;
        }
        if ($this->lookAheadKeyword('false')) {
            $this->position += 5;
            return false;
        }

        // Null
        if ($this->lookAheadKeyword('null')) {
            $this->position += 4;
            return null;
        }

        // Numeric
        if (preg_match('/-?[0-9]+(\.[0-9]+)?/', substr($this->expression, $this->position), $matches)) {
            $this->position += strlen($matches[0]);
            return strpos($matches[0], '.') !== false ? (float)$matches[0] : (int)$matches[0];
        }

        // Variable reference
        if ($this->lookAhead('$')) {
            $this->position++;
            return '$' . $this->parseIdentifier();
        }

        return $this->parseIdentifier();
    }

    protected function parseString(): string
    {
        $quote = $this->expression[$this->position] ?? '"';
        $this->position++;

        $start = $this->position;
        while ($this->position < strlen($this->expression) &&
               $this->expression[$this->position] !== $quote) {
            if ($this->expression[$this->position] === '\\') {
                $this->position++;
            }
            $this->position++;
        }

        $value = substr($this->expression, $start, $this->position - $start);
        $this->position++; // Skip closing quote

        return $value;
    }

    protected function skipWhitespace(): void
    {
        while ($this->position < strlen($this->expression) &&
               ctype_space($this->expression[$this->position])) {
            $this->position++;
        }
    }

    protected function skipChar(string $char): bool
    {
        $this->skipWhitespace();
        if ($this->expression[$this->position] ?? '' === $char) {
            $this->position++;
            return true;
        }
        return false;
    }

    protected function skipKeyword(string $keyword): bool
    {
        $this->skipWhitespace();
        if ($this->lookAhead(strtoupper($keyword))) {
            $this->position += strlen($keyword);
            return true;
        }
        return false;
    }

    protected function lookAhead(string $text): bool
    {
        return substr($this->expression, $this->position, strlen($text)) === $text;
    }

    protected function lookAheadKeyword(string $keyword): bool
    {
        return $this->lookAhead($keyword);
    }
}
```

### Rule Engine

```php
<?php
// includes/Interpreter/RuleEngine.php

namespace CustomModule\Interpreter;

use WHMCS\Database\Capsule;

class RuleEngine
{
    protected array $rules = [];

    public function loadRulesFromDatabase(): void
    {
        $dbRules = Capsule::table('mod_business_rules')
            ->where('is_active', 1)
            ->orderBy('priority')
            ->get();

        foreach ($dbRules as $rule) {
            $this->addRule($rule->name, $rule->expression, json_decode($rule->actions, true));
        }
    }

    public function addRule(string $name, string $expression, array $actions): void
    {
        $this->rules[] = [
            'name' => $name,
            'expression' => $expression,
            'actions' => $actions
        ];
    }

    public function evaluate(array $data): array
    {
        $context = new QueryContext();
        $context->set('record', $data);

        $matchedRules = [];

        foreach ($this->rules as $rule) {
            try {
                $parser = new ExpressionParser($rule['expression']);
                $expression = $parser->parse();

                if ($expression->interpret($context)) {
                    $matchedRules[] = [
                        'rule' => $rule,
                        'context' => $data
                    ];
                }
            } catch (\Exception $e) {
                logActivity("Rule evaluation error: {$e->getMessage()}");
            }
        }

        return $matchedRules;
    }

    public function executeRules(array $data): array
    {
        $matchedRules = $this->evaluate($data);
        $results = [];

        foreach ($matchedRules as $matched) {
            $actions = $matched['rule']['actions'];

            foreach ($actions as $action) {
                $results[] = $this->executeAction($action, $matched['context']);
            }
        }

        return $results;
    }

    protected function executeAction(array $action, array $context): array
    {
        $actionType = $action['type'] ?? '';
        $actionData = $action['data'] ?? [];

        return match($actionType) {
            'send_email' => $this->sendEmail($actionData, $context),
            'update_status' => $this->updateStatus($actionData, $context),
            'add_note' => $this->addNote($actionData, $context),
            'apply_discount' => $this->applyDiscount($actionData, $context),
            'notify_admin' => $this->notifyAdmin($actionData, $context),
            default => ['success' => false, 'error' => 'Unknown action type']
        };
    }

    protected function sendEmail(array $action, array $context): array
    {
        // Send email logic
        return ['success' => true, 'action' => 'email_sent'];
    }

    protected function updateStatus(array $action, array $context): array
    {
        Capsule::table('tblhosting')
            ->where('id', $context['service_id'] ?? 0)
            ->update(['domainstatus' => $action['status'] ?? 'Active']);

        return ['success' => true, 'action' => 'status_updated'];
    }

    protected function addNote(array $action, array $context): array
    {
        Capsule::table('tblnotes')->insert([
            'relid' => $context['client_id'] ?? 0,
            'type' => 'client',
            'message' => $action['message'] ?? '',
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return ['success' => true, 'action' => 'note_added'];
    }

    protected function applyDiscount(array $action, array $context): array
    {
        // Apply discount to order/invoice
        return ['success' => true, 'action' => 'discount_applied'];
    }

    protected function notifyAdmin(array $action, array $context): array
    {
        // Send admin notification
        return ['success' => true, 'action' => 'admin_notified'];
    }
}
```

### Usage Examples

```php
<?php
// Using the query DSL

$parser = new ExpressionParser('status = "Active" AND amount > 100');
$expression = $parser->parse();

$context = new QueryContext();
$context->set('record:status', 'Active');
$context->set('record:amount', 150);

$result = $expression->interpret($context);
echo $result ? 'Match!' : 'No match'; // Outputs: Match!
```

```php
<?php
// Complex expression

$parser = new ExpressionParser(
    '(country = "US" OR country = "CA") AND (amount >= 100 OR quantity > 5) AND NOT status = "Cancelled"'
);
$expression = $parser->parse();

$context = new QueryContext();
$context->set('record:country', 'US');
$context->set('record:amount', 200);
$context->set('record:quantity', 3);
$context->set('record:status', 'Active');

$result = $expression->interpret($context);
```

```php
<?php
// Rule engine with database rules

$engine = new RuleEngine();
$engine->loadRulesFromDatabase();

// Add custom rule
$engine->addRule('high_value_customer', 'total_spent > 1000 AND order_count > 5', [
    ['type' => 'apply_discount', 'data' => ['percentage' => 10]],
    ['type' => 'add_note', 'data' => ['message' => 'High value customer - 10% discount applied']]
]);

// Evaluate data
$matched = $engine->evaluate([
    'total_spent' => 1500,
    'order_count' => 7,
    'client_id' => 123
]);

if (!empty($matched)) {
    $results = $engine->executeRules([
        'total_spent' => 1500,
        'order_count' => 7,
        'client_id' => 123
    ]);
}
```

### Configuration DSL

```php
<?php
// includes/Interpreter/ConfigExpression.php

namespace CustomModule\Interpreter;

class ConfigExpression implements Expression
{
    protected string $key;
    protected $value;
    protected string $condition;

    public function __construct(string $key, $value, string $condition = '=')
    {
        $this->key = $key;
        $this->value = $value;
        $this->condition = $condition;
    }

    public function interpret(Context $context): bool
    {
        $configValue = $context->get($this->key);
        return $this->evaluate($configValue);
    }

    protected function evaluate($configValue): bool
    {
        return match($this->condition) {
            '=' => $configValue == $this->value,
            '!=' => $configValue != $this->value,
            'exists' => $configValue !== null,
            'not_exists' => $configValue === null,
            default => false
        };
    }

    public function toString(): string
    {
        return "config.{$this->key} {$this->condition} {$this->value}";
    }
}
```

```php
<?php
// includes/Interpreter/ConfigInterpreter.php

namespace CustomModule\Interpreter;

class ConfigInterpreter
{
    protected QueryContext $context;

    public function __construct()
    {
        $this->context = new QueryContext();
        $this->loadConfig();
    }

    protected function loadConfig(): void
    {
        $config = Capsule::table('tblconfiguration')
            ->pluck('value', 'setting')
            ->toArray();

        foreach ($config as $key => $value) {
            $this->context->set($key, $value);
        }
    }

    public function evaluate(string $expression): bool
    {
        $parser = new ExpressionParser($expression);
        return $parser->parse()->interpret($this->context);
    }

    public function setValue(string $key, $value): void
    {
        $this->context->set($key, $value);
    }

    public function getValue(string $key, $default = null)
    {
        return $this->context->get($key, $default);
    }
}
```

### SQL-like Query Builder

```php
<?php
// includes/Interpreter/QueryInterpreter.php

namespace CustomModule\Interpreter;

use WHMCS\Database\Capsule;

class QueryInterpreter
{
    protected Expression $whereClause;
    protected array $selectFields = [];
    protected string $tableName;
    protected int $limit = 100;
    protected int $offset = 0;
    protected array $orderBy = [];

    public function parse(string $sql): self
    {
        // Simple SQL parser
        $sql = trim($sql);

        // Parse SELECT
        if (preg_match('/^SELECT\s+(.+?)\s+FROM\s+(\w+)/i', $sql, $matches)) {
            $this->selectFields = array_map('trim', explode(',', $matches[1]));
            $this->tableName = $matches[2];
        }

        // Parse WHERE
        if (preg_match('/WHERE\s+(.+?)(?:\s+LIMIT|$)/i', $sql, $matches)) {
            $parser = new ExpressionParser($matches[1]);
            $this->whereClause = $parser->parse();
        }

        // Parse LIMIT
        if (preg_match('/LIMIT\s+(\d+)/i', $sql, $matches)) {
            $this->limit = (int)$matches[1];
        }

        // Parse ORDER BY
        if (preg_match('/ORDER BY\s+(.+?)(?:\s+LIMIT|$)/i', $sql, $matches)) {
            $fields = array_map('trim', explode(',', $matches[1]));
            foreach ($fields as $field) {
                $direction = 'ASC';
                if (preg_match('/(\w+)\s+(ASC|DESC)/i', $field, $dirMatches)) {
                    $field = $dirMatches[1];
                    $direction = strtoupper($dirMatches[2]);
                }
                $this->orderBy[] = ['field' => $field, 'direction' => $direction];
            }
        }

        return $this;
    }

    public function execute(): array
    {
        $query = Capsule::table($this->tableName);

        // Apply WHERE clause
        $results = $query->get();

        if ($this->whereClause) {
            $results = $results->filter(function($row) {
                $context = new QueryContext();
                foreach ((array)$row as $key => $value) {
                    $context->set("record:{$key}", $value);
                }
                return $this->whereClause->interpret($context);
            })->values();
        }

        // Apply ORDER BY
        if (!empty($this->orderBy)) {
            $results = $results->sortBy(function($row) {
                $values = [];
                foreach ($this->orderBy as $order) {
                    $values[] = $row->{$order['field']} ?? '';
                }
                return implode('|', $values);
            })->values();
        }

        // Apply LIMIT and OFFSET
        $results = $results->slice($this->offset, $this->limit)->values();

        return $results->toArray();
    }
}

// Usage
$interpreter = new QueryInterpreter();
$results = $interpreter->parse("
    SELECT id, firstname, lastname, email
    FROM tblclients
    WHERE status = 'Active' AND country IN ('US', 'CA', 'UK')
    ORDER BY lastname ASC, firstname ASC
    LIMIT 50
")->execute();
```

## Pros

- **Extensibility**: Easy to add new expressions
- **Flexibility**: Build complex queries dynamically
- **Testability**: Each expression can be tested independently
- **Reusability**: Share grammar across different interpreters

## Cons

- **Complexity**: Can become complex for large grammars
- **Performance**: Parsing overhead for simple operations
- **Maintenance**: Grammar changes require parser updates

## Best Practices

1. Use builder pattern for constructing complex expressions
2. Implement caching for frequently used expressions
3. Add error handling for malformed expressions
4. Consider using existing expression parsers for complex DSLs
5. Document grammar syntax clearly
# WHMCS GraphQL API Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing GraphQL APIs in WHMCS modules when needed.

## When to Use

- Building flexible APIs
- Mobile app backends
- Third-party integrations

## GraphQL Patterns

```php
<?php
// modules/addons/{module}/graphql.php

use GraphQL\Type\Schema;
use GraphQL\Type\Definition\Type;

class WHMCSGraphQL {
    private Schema $schema;

    public function __construct() {
        $this->schema = new Schema([
            'query' => $this->buildQueryType(),
            'mutation' => $this->buildMutationType(),
        ]);
    }

    private function buildQueryType(): ObjectType {
        return new ObjectType([
            'name' => 'Query',
            'fields' => [
                'services' => [
                    'type' => Type::listOf(Type::string()),
                    'resolve' => fn() => $this->getServices(),
                ],
                'service' => [
                    'type' => Type::string(),
                    'args' => [
                        'id' => Type::nonNull(Type::int()),
                    ],
                    'resolve' => fn($_, $args) => $this->getService($args['id']),
                ],
            ],
        ]);
    }

    private function getServices(): array {
        return Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->get()
            ->toArray();
    }
}

// Handle request
$query = $_POST['query'] ?? '';
$result = $graphql->execute($query);

header('Content-Type: application/json');
echo json_encode($result);
```

---

**Related Skills:**
- whmcs-rest-api-builder
- whmcs-api-documentation

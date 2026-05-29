# WHMCS API Documentation Generator

## Skill Description
Automatically generate API documentation from code annotations for WHMCS modules including endpoint descriptions, parameter documentation, and response schemas.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with Reflection API
- Basic understanding of API documentation patterns
- DocBlock annotation knowledge

## Step-by-Step Implementation

### 1. Documentation Generator
```php
<?php
// includes/documentation/DocGenerator.php

namespace WHMCS\Module\YourModule\Documentation;

class DocGenerator
{
    private string $basePath;
    private string $outputPath;
    private array $endpoints = [];
    private array $schemas = [];

    public function __construct(string $basePath, string $outputPath)
    {
        $this->basePath = $basePath;
        $this->outputPath = $outputPath;
    }

    public function scanControllers(string $controllersDir): self
    {
        $files = glob($controllersDir . '/*Controller.php');

        foreach ($files as $file) {
            $this->parseController($file);
        }

        return $this;
    }

    private function parseController(string $filePath): void
    {
        $content = file_get_contents($filePath);
        $className = $this->getClassName($filePath);

        // Extract class docblock
        $classDocBlock = $this->extractDocBlock($content, 'class');

        // Find all methods
        if (preg_match_all('/public function (\w+)\(\)(?:\s*:\s*void)?\s*\{/i', $content, $matches)) {
            foreach ($matches[1] as $methodName) {
                $this->parseMethod($className, $methodName, $content);
            }
        }
    }

    private function parseMethod(string $className, string $methodName, string $content): void
    {
        $methodDocBlock = $this->extractMethodDocBlock($content, $methodName);

        if (!$methodDocBlock) {
            return;
        }

        // Determine HTTP method from method name
        $httpMethod = $this->getHttpMethod($methodName);
        $endpoint = $this->getEndpoint($className, $methodName, $methodDocBlock);

        if (!$endpoint) {
            return;
        }

        $this->endpoints[] = [
            'method' => $httpMethod,
            'endpoint' => $endpoint,
            'summary' => $methodDocBlock['summary'] ?? '',
            'description' => $methodDocBlock['description'] ?? '',
            'parameters' => $methodDocBlock['params'] ?? [],
            'responses' => $methodDocBlock['responses'] ?? [],
            'security' => $methodDocBlock['security'] ?? ['api_key'],
            'tags' => $methodDocBlock['tags'] ?? []
        ];
    }

    private function extractDocBlock(string $content, string $type): ?array
    {
        $pattern = '/\/\\*\\*[\s\S]*?\\*\//';
        preg_match_all($pattern, $content, $matches);

        foreach ($matches[0] as $docBlock) {
            $parsed = $this->parseDocBlock($docBlock);
            if (!empty($parsed)) {
                return $parsed;
            }
        }

        return null;
    }

    private function extractMethodDocBlock(string $content, string $methodName): ?array
    {
        // Find the method and its docblock
        $pattern = '/\/\*\\*[\s\S]*?\*\/\s*public function ' . $methodName . '/';

        if (preg_match($pattern, $content, $match)) {
            $docBlock = preg_match('/\/\*\\*[\s\S]*?\*\//', $match[0], $docMatch);
            if ($docMatch) {
                return $this->parseDocBlock($docMatch[0]);
            }
        }

        return null;
    }

    private function parseDocBlock(string $docBlock): array
    {
        $result = [
            'summary' => '',
            'description' => '',
            'params' => [],
            'responses' => [],
            'tags' => [],
            'security' => []
        ];

        // Remove docblock markers
        $content = preg_replace('/^\s*\*\s{0,1}/m', '', $docBlock);
        $content = trim(preg_replace('/^\s*\/\\*\\*|^\s*\\*\//', '', $content));

        $lines = explode("\n", $content);
        $currentTag = null;
        $currentTagContent = [];

        foreach ($lines as $line) {
            $line = trim($line);

            if (preg_match('/^@(\w+)(?:\s+(.*))?$/', $line, $matches)) {
                // Save previous tag content
                if ($currentTag) {
                    $result = $this->processTag($result, $currentTag, $currentTagContent);
                }

                $currentTag = $matches[1];
                $currentTagContent = isset($matches[2]) ? [$matches[2]] : [];
            } elseif ($currentTag && !empty($line)) {
                $currentTagContent[] = $line;
            } elseif (empty($currentTag) && !empty($line) && strpos($line, '@') !== 0) {
                $result['summary'] .= (empty($result['summary']) ? '' : ' ') . $line;
            }
        }

        // Process last tag
        if ($currentTag) {
            $result = $this->processTag($result, $currentTag, $currentTagContent);
        }

        return $result;
    }

    private function processTag(array $result, string $tag, array $content): array
    {
        $contentStr = implode(' ', $content);

        switch ($tag) {
            case 'param':
                $parts = preg_split('/\s+/', $contentStr, 4);
                if (count($parts) >= 3) {
                    $result['params'][] = [
                        'name' => ltrim($parts[1], '$'),
                        'type' => $parts[0],
                        'description' => $parts[2] ?? ''
                    ];
                }
                break;

            case 'response':
                $parts = preg_split('/\s+/', $contentStr, 3);
                if (count($parts) >= 2) {
                    $result['responses'][] = [
                        'code' => $parts[0],
                        'description' => $parts[1] ?? ''
                    ];
                }
                break;

            case 'security':
                $result['security'][] = $contentStr;
                break;

            case 'tag':
                $result['tags'][] = $contentStr;
                break;

            case 'endpoint':
                $result['endpoint'] = $contentStr;
                break;

            case 'method':
                $result['method'] = strtoupper($contentStr);
                break;

            case 'description':
                $result['description'] = $contentStr;
                break;
        }

        return $result;
    }

    private function getHttpMethod(string $methodName): string
    {
        $methodMap = [
            'index' => 'GET',
            'show' => 'GET',
            'store' => 'POST',
            'create' => 'POST',
            'update' => 'PUT',
            'edit' => 'PUT',
            'destroy' => 'DELETE',
            'delete' => 'DELETE'
        ];

        $lowerName = strtolower($methodName);
        return $methodMap[$lowerName] ?? 'GET';
    }

    private function getEndpoint(string $className, string $methodName, array $docBlock): ?string
    {
        if (isset($docBlock['endpoint'])) {
            return $docBlock['endpoint'];
        }

        // Generate from class name and method
        $resource = preg_replace('/Controller$/', '', $className);
        $resource = strtolower(preg_replace('/([a-z])([A-Z])/', '$1-$2', $resource));

        switch (strtolower($methodName)) {
            case 'index':
                return "/api/{$resource}";
            case 'show':
                return "/api/{$resource}/{id}";
            case 'store':
            case 'create':
                return "/api/{$resource}";
            case 'update':
                return "/api/{$resource}/{id}";
            case 'destroy':
            case 'delete':
                return "/api/{$resource}/{id}";
            default:
                return "/api/{$resource}/" . strtolower($methodName);
        }
    }

    private function getClassName(string $filePath): string
    {
        $content = file_get_contents($filePath);
        if (preg_match('/namespace\s+([^;]+);/', $content, $matches)) {
            $namespace = $matches[1];
            $className = basename($filePath, '.php');
            return $namespace . '\\' . $className;
        }
        return basename($filePath, '.php');
    }

    public function generateOpenApi(): array
    {
        return [
            'openapi' => '3.0.0',
            'info' => [
                'title' => 'WHMCS Module API',
                'version' => '1.0.0',
                'description' => 'API documentation for WHMCS Module'
            ],
            'servers' => [
                ['url' => '/api', 'description' => 'API Base']
            ],
            'paths' => $this->generatePaths(),
            'components' => [
                'securitySchemes' => [
                    'api_key' => [
                        'type' => 'apiKey',
                        'name' => 'X-API-Key',
                        'in' => 'header'
                    ]
                ],
                'schemas' => $this->schemas
            ]
        ];
    }

    private function generatePaths(): array
    {
        $paths = [];

        foreach ($this->endpoints as $endpoint) {
            $path = $endpoint['endpoint'];

            if (!isset($paths[$path])) {
                $paths[$path] = [];
            }

            $pathItem = [
                'summary' => $endpoint['summary'],
                'description' => $endpoint['description'],
                'tags' => $endpoint['tags'],
                'security' => array_map(fn($s) => [$s => []], $endpoint['security']),
                'responses' => $this->generateResponses($endpoint['responses'])
            ];

            if (!empty($endpoint['parameters'])) {
                $pathItem['parameters'] = $this->generateParameters($endpoint['parameters']);
            }

            $paths[$path][strtolower($endpoint['method'])] = $pathItem;
        }

        return $paths;
    }

    private function generateParameters(array $params): array
    {
        $parameters = [];

        foreach ($params as $param) {
            $parameters[] = [
                'name' => $param['name'],
                'in' => 'query',
                'schema' => ['type' => $this->mapPhpType($param['type'])],
                'description' => $param['description'],
                'required' => false
            ];
        }

        return $parameters;
    }

    private function generateResponses(array $responses): array
    {
        $result = [];

        foreach ($responses as $response) {
            $result[$response['code']] = [
                'description' => $response['description']
            ];
        }

        // Add default responses
        if (!isset($result['200'])) {
            $result['200'] = ['description' => 'Successful response'];
        }

        if (!isset($result['401'])) {
            $result['401'] = ['description' => 'Unauthorized'];
        }

        if (!isset($result['404'])) {
            $result['404'] = ['description' => 'Not found'];
        }

        return $result;
    }

    private function mapPhpType(string $type): string
    {
        $typeMap = [
            'int' => 'integer',
            'integer' => 'integer',
            'string' => 'string',
            'bool' => 'boolean',
            'boolean' => 'boolean',
            'array' => 'array',
            'object' => 'object',
            'float' => 'number',
            'mixed' => 'string'
        ];

        return $typeMap[$type] ?? 'string';
    }

    public function generateMarkdown(): string
    {
        $md = "# API Documentation\n\n";
        $md .= "Generated: " . date('Y-m-d H:i:s') . "\n\n";

        // Group by tags
        $grouped = [];
        foreach ($this->endpoints as $endpoint) {
            $tag = $endpoint['tags'][0] ?? 'General';
            if (!isset($grouped[$tag])) {
                $grouped[$tag] = [];
            }
            $grouped[$tag][] = $endpoint;
        }

        foreach ($grouped as $tag => $endpoints) {
            $md .= "## {$tag}\n\n";

            foreach ($endpoints as $endpoint) {
                $md .= "### {$endpoint['method']} {$endpoint['endpoint']}\n\n";
                $md .= "{$endpoint['summary']}\n\n";

                if ($endpoint['description']) {
                    $md .= "{$endpoint['description']}\n\n";
                }

                if (!empty($endpoint['parameters'])) {
                    $md .= "#### Parameters\n\n";
                    $md .= "| Name | Type | Description |\n";
                    $md .= "|------|------|-------------|\n";

                    foreach ($endpoint['parameters'] as $param) {
                        $md .= "| {$param['name']} | {$param['type']} | {$param['description']} |\n";
                    }

                    $md .= "\n";
                }

                $md .= "#### Responses\n\n";
                foreach ($endpoint['responses'] as $response) {
                    $md .= "- `{$response['code']}`: {$response['description']}\n";
                }

                $md .= "\n";
            }
        }

        return $md;
    }

    public function save(string $format = 'json'): void
    {
        $output = match ($format) {
            'json' => json_encode($this->generateOpenApi(), JSON_PRETTY_PRINT),
            'markdown', 'md' => $this->generateMarkdown(),
            default => json_encode($this->generateOpenApi(), JSON_PRETTY_PRINT)
        };

        $extension = $format === 'markdown' || $format === 'md' ? 'md' : 'json';
        $filePath = rtrim($this->outputPath, '/') . "/api-docs.{$extension}";

        file_put_contents($filePath, $output);
    }
}
```

### 2. Example Annotated Controller
```php
<?php
// includes/api/Controllers/UserController.php

namespace WHMCS\Module\YourModule\Api\Controllers;

/**
 * User management endpoints
 *
 * @tag Users
 */
class UserController extends ApiController
{
    /**
     * List all users
     *
     * @endpoint /api/users
     * @method GET
     * @security api_key
     *
     * @param page Page number (default: 1)
     * @param per_page Items per page (default: 20)
     * @param status Filter by status
     *
     * @response 200 Returns list of users
     * @response 401 Unauthorized
     * @response 500 Server error
     */
    public function index(): void
    {
        // Implementation
    }

    /**
     * Get a specific user
     *
     * @endpoint /api/users/{id}
     * @method GET
     * @security api_key
     *
     * @param id User ID (required)
     *
     * @response 200 Returns user details
     * @response 404 User not found
     */
    public function show(): void
    {
        // Implementation
    }

    /**
     * Create a new user
     *
     * @endpoint /api/users
     * @method POST
     * @security api_key
     * @tag Users
     *
     * @param firstname First name (required)
     * @param lastname Last name (required)
     * @param email Email address (required)
     * @param password User password (required)
     *
     * @response 201 User created successfully
     * @response 400 Validation error
     * @response 401 Unauthorized
     */
    public function store(): void
    {
        // Implementation
    }

    /**
     * Update an existing user
     *
     * @endpoint /api/users/{id}
     * @method PUT
     * @security api_key
     *
     * @param id User ID (required)
     * @param firstname First name
     * @param lastname Last name
     * @param email Email address
     *
     * @response 200 User updated successfully
     * @response 404 User not found
     */
    public function update(): void
    {
        // Implementation
    }

    /**
     * Delete a user
     *
     * @endpoint /api/users/{id}
     * @method DELETE
     * @security api_key
     *
     * @param id User ID (required)
     *
     * @response 204 User deleted successfully
     * @response 404 User not found
     */
    public function destroy(): void
    {
        // Implementation
    }
}
```

### 3. Documentation Generator Script
```php
<?php
// generate-docs.php

require_once __DIR__ . '/init.php';

use WHMCS\Module\YourModule\Documentation\DocGenerator;

$basePath = __DIR__;
$controllersDir = __DIR__ . '/includes/api/Controllers';
$outputDir = __DIR__ . '/docs';

if (!is_dir($outputDir)) {
    mkdir($outputDir, 0755, true);
}

$generator = new DocGenerator($basePath, $outputDir);
$generator->scanControllers($controllersDir);
$generator->save('json');
$generator->save('markdown');

echo "Documentation generated successfully!\n";
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Incomplete annotations | Use strict parsing that requires annotations |
| Inconsistent endpoint naming | Enforce naming conventions in documentation |
| Missing parameter types | Default to string type when not specified |
| Outdated documentation | Generate docs in CI/CD pipeline |
| Complex nested parameters | Use JSON schema for complex objects |

## Security Considerations

1. **Don't expose sensitive endpoints** - Filter out admin-only endpoints from public docs
2. **Document authentication requirements** - Always specify security requirements
3. **Hide internal implementation details** - Don't include internal class names
4. **Validate documentation output** - Check generated docs for sensitive data
5. **Access control for docs** - Consider protecting documentation endpoints

## Testing Checklist

- [ ] Test doc generation from annotated controllers
- [ ] Test missing annotations are handled
- [ ] Test endpoint extraction from class names
- [ ] Test parameter type mapping
- [ ] Test response code extraction
- [ ] Test security annotation parsing
- [ ] Test tag grouping
- [ ] Test OpenAPI JSON output format
- [ ] Test Markdown output format
- [ ] Test documentation coverage report

## Reference Links

- [OpenAPI Specification 3.0](https://spec.openapis.org/oas/v3.0.3)
- [API Documentation Best Practices](https://swagger.io/specification/)
- [DocBlock Standard](https://docs.phpdoc.org/)

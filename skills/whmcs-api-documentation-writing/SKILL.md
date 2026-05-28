# WHMCS API Documentation Writing Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for writing comprehensive API documentation for WHMCS custom modules and integrations.

## When to Use

- Documenting custom API endpoints
- Writing developer documentation
- Creating integration guides
- Generating API reference docs

## Documentation Patterns

### 1. Documentation Generator

```php
<?php
namespace WHMCS\Docs;

class ApiDocGenerator {
    private string $outputPath;
    private array $modules = [];

    public function generateFromModules(string $modulePath): array {
        $files = glob($modulePath . '/modules/servers/*/*.php');

        $docs = [];
        foreach ($files as $file) {
            $module = basename(dirname($file));
            $content = file_get_contents($file);

            $docs[$module] = $this->parseModuleApi($content, $module);
        }

        return $docs;
    }

    private function parseModuleApi(string $content, string $moduleName): array {
        $functions = $this->extractFunctions($content);
        $api = [
            'name' => $moduleName,
            'type' => 'server_module',
            'endpoints' => [],
            'parameters' => [],
            'errors' => [],
        ];

        foreach ($functions as $function) {
            if ($this->isApiFunction($function['name'], $moduleName)) {
                $api['endpoints'][] = $this->parseEndpoint($function);
            }
        }

        return $api;
    }

    private function extractFunctions(string $content): array {
        $functions = [];
        preg_match_all('/function\s+' . preg_quote($GLOBALS['moduleName'] ?? '', '/') . '_(\w+)\s*\((.*?)\)/s', $content, $matches, PREG_SET_ORDER);

        foreach ($matches as $match) {
            $functions[] = [
                'name' => $match[1],
                'params_str' => $match[2],
            ];
        }

        return $functions;
    }

    private function isApiFunction(string $name, string $module): bool {
        // Known API functions
        $apiFunctions = [
            'MetaData', 'ConfigOptions', 'CreateAccount', 'SuspendAccount',
            'UnsuspendAccount', 'TerminateAccount', 'ChangePassword',
            'ChangePackage', 'TestConnection', 'ClientArea',
        ];

        return !in_array($name, $apiFunctions);
    }

    public function generateMarkdown(array $apiDoc): string {
        $md = "# {$apiDoc['name']} API\n\n";
        $md .= "> Generated: " . date('Y-m-d H:i:s') . "\n\n";

        foreach ($apiDoc['endpoints'] as $endpoint) {
            $md .= "### `{$endpoint['function']}()`\n\n";
            $md .= "{$endpoint['description']}\n\n";
            $md .= "**Parameters:**\n\n";

            if (!empty($endpoint['params'])) {
                $md .= "| Name | Type | Required | Description |\n";
                $md .= "|------|------|----------|-------------|\n";

                foreach ($endpoint['params'] as $param) {
                    $required = $param['required'] ? 'Yes' : 'No';
                    $md .= "| {$param['name']} | {$param['type']} | {$required} | {$param['description']} |\n";
                }
            } else {
                $md .= "_No parameters_\n";
            }

            $md .= "\n**Returns:** `{$endpoint['returns']}`\n\n";
            $md .= "**Example:**\n\n```php\n{$endpoint['example']}\n```\n\n---\n\n";
        }

        return $md;
    }

    public function generateOpenApi(array $apiDoc): array {
        $openapi = [
            'openapi' => '3.0.0',
            'info' => [
                'title' => "{$apiDoc['name']} API",
                'version' => '1.0',
            ],
            'paths' => [],
        ];

        foreach ($apiDoc['endpoints'] as $endpoint) {
            $pathName = '/' . strtolower(str_replace('_', '-', $endpoint['function']));

            $openapi['paths'][$pathName] = [
                'post' => [
                    'summary' => $endpoint['description'],
                    'requestBody' => [
                        'content' => [
                            'application/json' => [
                                'schema' => [
                                    'type' => 'object',
                                    'properties' => $this->generateSchema($endpoint['params']),
                                ],
                            ],
                        ],
                    ],
                    'responses' => [
                        '200' => [
                            'description' => 'Success',
                            'content' => [
                                'application/json' => [
                                    'schema' => [
                                        'type' => 'object',
                                        'properties' => [
                                            'status' => ['type' => 'string', 'example' => 'success'],
                                            'data' => ['type' => 'object'],
                                        ],
                                    ],
                                ],
                            ],
                        ],
                    ],
                ],
            ];
        }

        return $openapi;
    }

    private function generateSchema(array $params): array Properties {
        $properties = [];
        foreach ($params as $param) {
            $properties[$param['name']] = [
                'type' => $this->mapPhpToJsonType($param['type']),
                'description' => $param['description'],
            ];
        }
        return $properties;
    }

    private function mapPhpToJsonType(string $phpType): string {
        return match(true) {
            str_contains($phpType, 'int') || str_contains($phpType, 'float') => 'number',
            str_contains($phpType, 'bool') => 'boolean',
            str_contains($phpType, 'array') => 'array',
            default => 'string',
        };
    }
}
```

### 2. Integration Documentation

```php
<?php
namespace WHMCS\Docs;

class IntegrationDocBuilder {
    public function buildIntegrationGuide(
        string $moduleName,
        string $integrationType,
        array $options = []
    ): string {
        $sections = [
            $this->buildOverview($moduleName, $integrationType),
            $this->buildPrerequisites(),
            $this->buildInstallation(),
            $this->buildConfiguration($options),
            $this->buildUsage(),
            $this->buildApiReference(),
            $this->buildTroubleshooting(),
            $this->buildChangelog(),
        ];

        return implode("\n\n---\n\n", $sections);
    }

    private function buildOverview(string $module, string $type): string {
        return <<<MD
# {$module} Integration Guide

## Overview

This integration connects WHMCS with [External Service] for {$type}.

**Module Version:** 1.0.0
**WHMCS Compatibility:** 8.0+
**Last Updated:** {$this->formatDate()}

## Features

- Automatic provisioning
- Real-time status sync
- Automated billing
- Webhook notifications
MD;
    }

    private function buildPrerequisites(): string {
        return <<<MD
## Prerequisites

- WHMCS 8.0 or higher
- PHP 8.0 or higher
- External service account
- API credentials

## Installation

1. Download the module from GitHub
2. Upload to `/modules/{servers|gateways|registrars}/{module}/`
3. Navigate to **System Settings > Modules**
4. Activate the module
5. Configure API credentials
MD;
    }

    private function buildConfiguration(array $options): string {
        $fields = array_map(function($field) {
            return "| `{$field['name']}` | {$field['description']} |";
        }, $options['config_fields'] ?? []);

        return <<<MD
## Configuration

| Setting | Description |
|---------|------------|
{$fields}

### Environment Variables

\`\`\`bash
EXTERNAL_API_URL=https://api.example.com
EXTERNAL_API_KEY=your_api_key
EXTERNAL_API_SECRET=your_api_secret
\`\`\`
MD;
    }

    private function buildApiReference(): string {
        return <<<MD
## API Reference

### Endpoints

#### `CreateAccount`

Creates a new service account.

**Parameters:**
- `user_id` (int): WHMCS client ID
- `service_id` (int): WHMCS service ID
- `params` (array): Module configuration options

**Returns:** `success` on success, error message on failure

**Example:**
```php
\$result = localAPI('CreateAccount', [
    'serviceid' => 123,
    'pid' => 1,
]);
```

#### `SuspendAccount`

Suspends an active service.

**Parameters:**
- `service_id` (int): WHMCS service ID
- `params` (array): Suspend reason and options

**Returns:** `success` on success, error message on failure
MD;
    }

    private function buildTroubleshooting(): string {
        return <<<MD
## Troubleshooting

### Error: "API authentication failed"

1. Verify API credentials in module settings
2. Check if API key has required permissions
3. Ensure IP whitelist includes your WHMCS server

### Error: "Service not found"

1. Verify the external service exists
2. Check sync status in admin area
3. Review webhook delivery logs

### Error: "Rate limit exceeded"

Implement exponential backoff or contact support for higher limits.

## Support

For issues, contact: support@example.com
GitHub Issues: https://github.com/example/whmcs-module/issues
MD;
    }

    private function buildChangelog(): string {
        return <<<MD
## Changelog

### Version 1.0.0 (2026-05-29)
- Initial release
- Support for WHMCS 8.x
- Basic provisioning features

### Version 1.1.0 (2026-06-15)
- Added webhook support
- Improved error handling
- Performance optimizations
MD;
    }

    private function formatDate(): string {
        return date('Y-m-d');
    }
}
```

### 3. Markdown Documentation Template

```php
<?php
// Generates complete API documentation markdown
function generateApiDocumentation(string $moduleName, array $moduleInfo): string {
    $md = <<<HEADER
# {$moduleName} API Documentation
**Version:** {$moduleInfo['version']}
**Last Updated:** {$moduleInfo['updated']}
**WHMCS:** {$moduleInfo['whmcs_version']}+

## Base URL

```
{$moduleInfo['base_url']}
```

## Authentication

All API requests require authentication using a Bearer token:

\`\`\`bash
curl -H "Authorization: Bearer YOUR_TOKEN" \\
     -H "Content-Type: application/json" \\
     {$moduleInfo['base_url']}/endpoint
\`\`\`

## Request Format

\`\`\`json
{
    "action": "action_name",
    "params": {
        "key": "value"
    }
}
\`\`\`

## Response Format

\`\`\`json
{
    "status": "success",
    "data": { },
    "message": ""
}
\`\`\`

---

## Actions
HEADER;

    foreach ($moduleInfo['actions'] as $action) {
        $md .= generateActionDocumentation($action);
    }

    $md .= <<<ERRORS
---

## Error Codes

| Code | Error | Description |
|------|-------|-------------|
| 1000 | AUTH_FAILED | Invalid or expired authentication token |
| 1001 | RATE_LIMIT | Too many requests |
| 2000 | INVALID_PARAMS | Missing or invalid parameters |
| 3000 | NOT_FOUND | Resource not found |
| 4000 | SERVER_ERROR | External service error |

---

## Rate Limits

- **Default:** 100 requests/minute
- **Enterprise:** 1000 requests/minute

---

## Examples
ERRORS;

    $md .= generateExamples($moduleInfo['examples']);

    return $md;
}

function generateActionDocumentation(array $action): string {
    $md = "\n## `{$action['name']}`\n\n";
    $md .= "{$action['description']}\n\n";
    $md .= "**Endpoint:** `{$action['endpoint']}`\n\n";

    if (!empty($action['parameters'])) {
        $md .= "### Parameters\n\n";
        $md .= "| Name | Type | Required | Description |\n";
        $md .= "|------|------|----------|-------------|\n";

        foreach ($action['parameters'] as $param) {
            $required = $param['required'] ? 'Yes' : 'No';
            $md .= "| {$param['name']} | {$param['type']} | {$required} | {$param['description']} |\n";
        }
        $md .= "\n";
    }

    $md .= "### Example Request\n\n";
    $md .= "```bash\n{$action['example_request']}\n```\n\n";
    $md .= "### Example Response\n\n";
    $md .= "```json\n{$action['example_response']}\n```\n";

    return $md;
}
```

### 4. Interactive API Console

```php
<?php
// modules/addons/{module}/admin/api_console.php
if (!defined("WHMCS")) { die("Direct access denied"); }

use WHMCS\Docs\ApiDocGenerator;

$generator = new ApiDocGenerator();
$apiDocs = $generator->generateFromModules(ROOTDIR);

// Display API console
echo <<<HTML
<div class="api-console">
    <h2>API Console</h2>

    <form id="api-form">
        <select name="action" id="action-select">
            <option value="">Select Action</option>
            <?php foreach ($apiDocs as $doc): ?>
                <?php foreach ($doc['endpoints'] as $endpoint): ?>
                    <option value="{$endpoint['function']}">{$doc['name']}::{$endpoint['function']}</option>
                <?php endforeach; ?>
            <?php endforeach; ?>
        </select>

        <textarea name="params" id="params-input" placeholder='{"param": "value"}'></textarea>

        <button type="submit">Execute</button>
    </form>

    <div id="response-output">
        <h3>Response</h3>
        <pre id="response-content"></pre>
    </div>
</div>

<script>
document.getElementById('api-form').addEventListener('submit', async (e) => {
    e.preventDefault();
    const response = await fetch('api.php', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            action: document.getElementById('action-select').value,
            params: JSON.parse(document.getElementById('params-input').value || '{}')
        })
    });
    document.getElementById('response-content').textContent =
        JSON.stringify(await response.json(), null, 2);
});
</script>
HTML;
```

### 5. Documentation Configuration

```php
<?php
// Automatic documentation generation on module activation
add_hook('{Module}_activate', 1, function($vars) {
    $docConfig = [
        'output_dir' => ROOTDIR . '/docs/api/',
        'formats' => ['markdown', 'html', 'openapi'],
        'include_examples' => true,
        'include_errors' => true,
    ];

    Capsule::table('tblconfiguration')->insert([
        'setting' => 'ModuleDocConfig',
        'value' => json_encode($docConfig),
    ]);
});

// Generate documentation on version update
add_hook('{Module}_upgrade', 1, function($vars) {
    $generator = new ApiDocGenerator();
    $apiDocs = $generator->generateFromModules(ROOTDIR . '/modules/servers/my_module/');

    foreach ($apiDocs as $module => $doc) {
        $markdown = $generator->generateMarkdown($doc);
        $openapi = $generator->generateOpenApi($doc);

        file_put_contents(ROOTDIR . "/docs/api/{$module}.md", $markdown);
        file_put_contents(ROOTDIR . "/docs/api/{$module}.json", json_encode($openapi, JSON_PRETTY_PRINT));
    }
});
```

## Checklist

- [ ] API doc generator class
- [ ] Module function documentation extraction
- [ ] Markdown format output
- [ ] OpenAPI/Swagger format
- [ ] Integration guide template
- [ ] Common errors documentation
- [ ] Code examples
- [ ] Interactive API console
- [ ] Auto-generation on module update

---

**Related Skills:**
- whmcs-module-documentation
- whmcs-api-integration
- whmcs-rest-api-builder
- whmcs-module-packaging

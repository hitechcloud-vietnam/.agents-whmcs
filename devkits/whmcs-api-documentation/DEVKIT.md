# WHMCS API Documentation Module

API documentation generator with Swagger/OpenAPI support.

## Features

- Auto-generate API documentation from code
- Swagger/OpenAPI 3.0 support
- Interactive API explorer
- Request/response examples
- Authentication documentation
- Rate limiting documentation
- Code samples (PHP, JavaScript, Python)
- Export to HTML/PDF
- Version control
- Changelog tracking
- Custom documentation pages
- API endpoint grouping
- Search functionality
- Try-it-out sandbox

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/apidocumentation/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Access documentation at /admin/apidocumentation/

## Usage

```php
// Register API endpoint for documentation
$result = apidocumentation_RegisterEndpoint(array(
    'group' => 'clients',
    'path' => '/api/clients/{id}',
    'method' => 'GET',
    'summary' => 'Get client by ID',
    'description' => 'Retrieves client information',
    'parameters' => array(
        array('name' => 'id', 'in' => 'path', 'required' => true, 'schema' => array('type' => 'integer'))
    ),
    'responses' => array(
        '200' => array('description' => 'Success', 'schema' => array('$ref' => '#/components/schemas/Client'))
    ),
    'tags' => array('clients', 'read')
));

// Generate full documentation
$result = apidocumentation_GenerateDocumentation(array(
    'title' => 'WHMCS API Documentation',
    'version' => '1.0.0',
    'include_internal' => true
));

// Export documentation
$export = apidocumentation_ExportDocumentation('swagger');
// Returns: json, yaml, or html

// Add endpoint example
apidocumentation_AddExample($endpointId, array(
    'request' => array('method' => 'GET', 'url' => '/api/clients/123'),
    'response' => array('status' => 200, 'body' => array('id' => 123, 'name' => 'John Doe'))
));

// Update changelog
apidocumentation_AddChangelog(array(
    'version' => '1.0.1',
    'changes' => array('Added new endpoint', 'Fixed authentication bug'),
    'date' => '2026-05-28'
));

// Get all endpoints
$endpoints = apidocumentation_GetEndpoints(array(
    'group' => 'clients',
    'tag' => 'read'
));

// Get endpoint details
$details = apidocumentation_GetEndpoint($endpointId);

// Update endpoint
apidocumentation_UpdateEndpoint($endpointId, array(
    'deprecated' => true,
    'description' => 'Use /api/v2/clients instead'
));

// Generate code samples
$samples = apidocumentation_GenerateCodeSamples($endpointId);
// Returns: php, javascript, python examples
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| DocTitle | text | WHMCS API Documentation | Documentation title |
| DocVersion | text | 1.0.0 | API version |
| IncludeInternal | yesno | no | Include internal endpoints |
| EnableTryIt | yesno | yes | Enable sandbox testing |
| DefaultLanguage | dropdown | en | Default language |
| Theme | dropdown | default | Documentation theme |
| ShowExamples | yesno | yes | Show request examples |
| Authentication | dropdown | bearer | Auth type |

## OpenAPI Components

| Component | Description |
|-----------|-------------|
| paths | API endpoints |
| components/schemas | Data models |
| components/security | Security schemes |
| tags | Endpoint grouping |
| servers | Server configurations |

## Database Tables

- `mod_apidocumentation_endpoints` - API endpoints
- `mod_apidocumentation_groups` - Endpoint groups
- `mod_apidocumentation_examples` - Request/response examples
- `mod_apidocumentation_changelog` - Version changelog
- `mod_apidocumentation_versions` - Doc versions

## API Functions

| Function | Description |
|----------|-------------|
| `apidocumentation_RegisterEndpoint()` | Register endpoint |
| `apidocumentation_GetEndpoints()` | List endpoints |
| `apidocumentation_GetEndpoint()` | Get endpoint details |
| `apidocumentation_UpdateEndpoint()` | Update endpoint |
| `apidocumentation_DeleteEndpoint()` | Remove endpoint |
| `apidocumentation_AddExample()` | Add example |
| `apidocumentation_GenerateDocumentation()` | Generate OpenAPI spec |
| `apidocumentation_ExportDocumentation()` | Export docs |
| `apidocumentation_GenerateCodeSamples()` | Generate code samples |
| `apidocumentation_AddChangelog()` | Add changelog entry |
| `apidocumentation_SearchEndpoints()` | Search endpoints |

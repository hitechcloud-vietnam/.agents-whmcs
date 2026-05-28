# WHMCS Custom API DevKit

A comprehensive custom REST API server for WHMCS with full CRUD operations, authentication, and rate limiting.

## Features

- RESTful API design with standard HTTP methods
- API key authentication with scopes
- Rate limiting per API key
- IP whitelist support
- Comprehensive request logging
- Pagination support
- Built-in API documentation
- CRUD operations for clients, services, invoices, and domains

## Installation

1. Copy module files to:
   ```
   modules/addons/whmcs_custom_api/
   ```

2. Activate the module in WHMCS Admin > Addon Modules

3. Access the API at:
   ```
   https://your-whmcs.com/modules/addons/whmcs_custom_api/api.php
   ```

## Configuration

### Admin Settings

1. Go to Admin > Addon Modules > Custom API
2. Manage API keys (create, revoke, view)
3. Configure endpoints
4. View request logs
5. Access API documentation

## Usage

### Authentication

Include your API key in the request header:

```bash
curl -X GET https://your-whmcs.com/modules/addons/whmcs_custom_api/api.php/clients \
  -H "X-API-Key: your-api-key-here"
```

Or use Bearer token:

```bash
curl -X GET https://your-whmcs.com/modules/addons/whmcs_custom_api/api.php/clients \
  -H "Authorization: Bearer your-api-key-here"
```

### Endpoints

#### Clients

```bash
# List clients
GET /clients?page=1&per_page=20&search=john&status=Active

# Get single client
GET /clients/{id}

# Create client
POST /clients
{
    "firstname": "John",
    "lastname": "Doe",
    "email": "john@example.com",
    "companyname": "Example Inc",
    "password": "secure_password"
}

# Update client
PUT /clients/{id}
{
    "firstname": "John",
    "lastname": "Updated"
}

# Delete client
DELETE /clients/{id}
```

#### Services

```bash
# List services
GET /services?client_id=123&status=Active

# Get single service
GET /services/{id}

# Create service
POST /services
{
    "client_id": 123,
    "product_id": 1,
    "domain": "example.com"
}

# Update service
PUT /services/{id}
{
    "domain": "new-domain.com"
}
```

#### Invoices

```bash
# List invoices
GET /invoices?client_id=123&status=Paid

# Get single invoice
GET /invoices/{id}

# Create invoice
POST /invoices
{
    "client_id": 123,
    "items": [
        {"description": "Product", "amount": 10.00}
    ]
}
```

#### Domains

```bash
# List domains
GET /domains?client_id=123

# Get single domain
GET /domains/{id}

# Register domain
POST /domains
{
    "client_id": 123,
    "domain": "example.com"
}
```

## Response Format

### Success Response

```json
{
    "success": true,
    "status_code": 200,
    "data": { ... }
}
```

### Paginated Response

```json
{
    "success": true,
    "status_code": 200,
    "data": [ ... ],
    "meta": {
        "current_page": 1,
        "per_page": 20,
        "total": 100,
        "total_pages": 5
    }
}
```

### Error Response

```json
{
    "error": true,
    "message": "Client not found",
    "code": 404
}
```

## API Key Management

### Create API Key

1. Go to Admin > Custom API > API Keys
2. Click "Add New Key"
3. Enter name and select scopes
4. Copy the generated API key

### Scopes

Available scopes for API keys:
- `clients:read` - View clients
- `clients:write` - Create/update clients
- `clients:delete` - Delete clients
- `services:read` - View services
- `services:write` - Create/update services
- `invoices:read` - View invoices
- `invoices:write` - Create/update invoices
- `domains:read` - View domains
- `domains:write` - Register/manage domains
- `*` - Full access (admin only)

## Rate Limiting

Default rate limit is 100 requests per minute per API key. Configure custom limits per key in admin panel.

Rate limit headers are included in responses:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1622820060
```

## Logging

All API requests are logged with:
- API key ID
- HTTP method
- Endpoint path
- Request body
- Response code
- Execution time
- Client IP

Access logs in Admin > Custom API > Logs

## File Structure

```
whmcs-custom-api/
├── api-server.php          # Main API server
├── lib/
│   ├── Router.php          # Route handling
│   ├── Middleware.php      # Authentication & rate limiting
│   └── Controllers/
│       ├── BaseController.php
│       ├── ClientsController.php
│       ├── ServicesController.php
│       ├── InvoicesController.php
│       └── DomainsController.php
├── routes/
│   └── api-routes.php     # Route definitions
└── templates/
    └── api-docs.tpl       # API documentation
```

## Security

- Always use HTTPS for API requests
- Store API secrets securely
- Rotate API keys regularly
- Use IP whitelisting for sensitive operations
- Implement proper scopes for each key

## Requirements

- WHMCS 7.0+
- PHP 7.4+
- cURL extension

## Support

For issues and feature requests, please contact the developer.
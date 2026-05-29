# WHMCS API Contract Testing Workflow

## Overview
This workflow guides you through comprehensive API contract testing for WHMCS, ensuring your integrations conform to expected API specifications.

## Prerequisites
- WHMCS installation (v8.0+)
- API access credentials
- Contract testing framework (Pact, Dredd)
- API documentation

## Step-by-Step Guide

### Step 1: Define API Contracts

#### OpenAPI Specification
```yaml
# openapi/whmcs-api.yaml
openapi: 3.0.3
info:
  title: WHMCS REST API
  version: 8.0.0
  description: WHMCS API Contract

servers:
  - url: http://localhost/includes/api.php
    description: Local development

paths:
  /api/v1/clients:
    get:
      summary: List clients
      operationId: getClients
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 25
            maximum: 100
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ClientList'
        '401':
          $ref: '#/components/responses/Unauthorized'

    post:
      summary: Create client
      operationId: createClient
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateClientRequest'
      responses:
        '201':
          description: Client created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ClientResponse'
        '400':
          $ref: '#/components/responses/BadRequest'

  /api/v1/clients/{id}:
    get:
      summary: Get client
      operationId: getClient
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ClientResponse'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  schemas:
    ClientList:
      type: object
      properties:
        result:
          type: string
          enum: [success]
        totalresults:
          type: integer
        clients:
          type: array
          items:
            $ref: '#/components/schemas/Client'

    Client:
      type: object
      properties:
        id:
          type: integer
        firstname:
          type: string
        lastname:
          type: string
        email:
          type: string
          format: email
        companyname:
          type: string
        datecreated:
          type: string
          format: date

    CreateClientRequest:
      type: object
      required:
        - firstname
        - lastname
        - email
        - password2
      properties:
        firstname:
          type: string
          minLength: 2
        lastname:
          type: string
          minLength: 2
        email:
          type: string
          format: email
        password2:
          type: string
          minLength: 8

  responses:
    Unauthorized:
      description: Unauthorized
      content:
        application/json:
          schema:
            type: object
            properties:
              result:
                type: string
                example: error
              message:
                type: string
                example: Authentication Failed
```

### Step 2: Set Up Pact for Contract Testing

#### Install Dependencies
```bash
npm init -y
npm install -D @pact-foundation/pact jest @pact-foundation/pact-js
```

#### Configure Pact
```javascript
// pact/config.js
const { PactV3 } = require('@pact-foundation/pact');

const provider = new PactV3({
  consumer: 'YourIntegration',
  provider: 'WHMCS',
  dir: './pact/contracts',
  logLevel: 'warn',
  host: 'localhost',
});
```

### Step 3: Write Contract Tests

#### Client API Contract Tests
```javascript
// pact/client.contract.spec.js
const { PactV3, like, eachLike } = require('@pact-foundation/pact');

const provider = new PactV3({
  consumer: 'YourIntegration',
  provider: 'WHMCS',
  dir: './pact/contracts',
});

describe('WHMCS Client API Contract', () => {
  describe('GET /api/v1/clients', () => {
    test('returns list of clients', async () => {
      await provider.addInteraction({
        states: [{ description: 'clients exist' }],
        uponReceiving: 'a request for clients',
        withRequest: {
          method: 'GET',
          path: '/includes/api.php',
          query: {
            action: 'GetClients',
            username: 'test_user',
            password: 'test_pass',
            responsetype: 'json',
          },
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': like('application/json') },
          body: {
            result: like('success'),
            totalresults: like(1),
            clients: eachLike({
              id: like(1),
              firstname: like('John'),
              lastname: like('Doe'),
              email: like('john@example.com'),
            }),
          },
        },
      });

      const response = await fetch(
        'http://localhost/includes/api.php?action=GetClients&username=test_user&password=test_pass&responsetype=json'
      );
      const body = await response.json();

      expect(response.status).toBe(200);
      expect(body.result).toBe('success');
      expect(body.totalresults).toBeGreaterThan(0);
    });

    test('returns empty list when no clients', async () => {
      await provider.addInteraction({
        states: [{ description: 'no clients exist' }],
        uponReceiving: 'a request for clients with no data',
        withRequest: {
          method: 'GET',
          path: '/includes/api.php',
          query: {
            action: 'GetClients',
            username: 'test_user',
            password: 'test_pass',
          },
        },
        willRespondWith: {
          status: 200,
          body: {
            result: like('success'),
            totalresults: 0,
            clients: [],
          },
        },
      });

      const response = await fetch(
        'http://localhost/includes/api.php?action=GetClients'
      );
      const body = await response.json();

      expect(body.totalresults).toBe(0);
      expect(body.clients).toEqual([]);
    });
  });

  describe('POST /api/v1/clients', () => {
    test('creates a new client', async () => {
      await provider.addInteraction({
        states: [{ description: 'client can be created' }],
        uponReceiving: 'a request to create a client',
        withRequest: {
          method: 'POST',
          path: '/includes/api.php',
          body: {
            action: 'AddClient',
            username: 'test_user',
            password: 'test_pass',
            firstname: 'Jane',
            lastname: 'Smith',
            email: 'jane@example.com',
            password2: 'SecurePass123',
          },
        },
        willRespondWith: {
          status: 200,
          body: {
            result: like('success'),
            clientid: like(2),
          },
        },
      });

      const response = await fetch('http://localhost/includes/api.php', {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: new URLSearchParams({
          action: 'AddClient',
          firstname: 'Jane',
          lastname: 'Smith',
          email: 'jane@example.com',
          password2: 'SecurePass123',
        }),
      });
      const body = await response.json();

      expect(body.result).toBe('success');
      expect(body.clientid).toBeDefined();
    });

    test('returns error for invalid email', async () => {
      await provider.addInteraction({
        uponReceiving: 'a request to create client with invalid email',
        withRequest: {
          method: 'POST',
          path: '/includes/api.php',
          body: {
            action: 'AddClient',
            email: 'invalid-email',
          },
        },
        willRespondWith: {
          status: 400,
          body: {
            result: like('error'),
            message: like('Email Address is invalid'),
          },
        },
      });

      const response = await fetch('http://localhost/includes/api.php', {
        method: 'POST',
        body: new URLSearchParams({
          action: 'AddClient',
          email: 'invalid-email',
        }),
      });
      const body = await response.json();

      expect(body.result).toBe('error');
    });
  });
});
```

#### Module API Contract Tests
```javascript
// pact/module.contract.spec.js
const { PactV3, like, term } = require('@pact-foundation/pact');

const provider = new PactV3({
  consumer: 'YourModule',
  provider: 'WHMCS',
  dir: './pact/contracts',
});

describe('WHMCS Module API Contract', () => {
  describe('Module Webhook', () => {
    test('receives and processes webhook events', async () => {
      await provider.addInteraction({
        uponReceiving: 'a webhook event for client creation',
        withRequest: {
          method: 'POST',
          path: '/modules/addons/yourmodule/webhook.php',
          headers: {
            'Content-Type': 'application/json',
            'X-Webhook-Signature': like('sha256=abc123'),
          },
          body: {
            event: term({
              generate: 'ClientCreate',
              matcher: '^Client(Create|Update|Delete)$',
            }),
            client_id: like(1),
            timestamp: /\d{10}/,
          },
        },
        willRespondWith: {
          status: 200,
          body: {
            success: true,
            message: like('Event processed'),
          },
        },
      });

      const response = await fetch(
        'http://localhost/modules/addons/yourmodule/webhook.php',
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Webhook-Signature': 'sha256=abc123',
          },
          body: JSON.stringify({
            event: 'ClientCreate',
            client_id: 1,
            timestamp: Date.now(),
          }),
        }
      );
      const body = await response.json();

      expect(response.status).toBe(200);
      expect(body.success).toBe(true);
    });

    test('rejects invalid webhook signature', async () => {
      await provider.addInteraction({
        uponReceiving: 'a webhook with invalid signature',
        withRequest: {
          method: 'POST',
          path: '/modules/addons/yourmodule/webhook.php',
          headers: {
            'X-Webhook-Signature': 'invalid-signature',
          },
        },
        willRespondWith: {
          status: 403,
          body: {
            success: false,
            error: like('Invalid signature'),
          },
        },
      });

      const response = await fetch(
        'http://localhost/modules/addons/yourmodule/webhook.php',
        {
          method: 'POST',
          headers: {
            'X-Webhook-Signature': 'invalid-signature',
          },
        }
      );
      const body = await response.json();

      expect(response.status).toBe(403);
      expect(body.success).toBe(false);
    });
  });
});
```

### Step 4: Verify Contracts with Dredd

#### Dredd Configuration
```yaml
# dredd.yml
reporter: api-blueprint,html
dry-run: null
hookfiles: './tests/dredd-hooks.js'
language: nodejs
sandbox: false
server: 'http://localhost'
server-cleanup: true
init: false
names: false
only: []
output: []
header: []
sorted: true
user: null
inline-errors: false
details: false
method: []
color: true
level: info
timestamp: true
broadhead: false

endpoints:
  - http://localhost/includes/api.php
```

#### Dredd Hooks
```javascript
// tests/dredd-hooks.js
const hooks = require('hooks');

hooks.beforeEach((transaction) => {
  // Add authentication
  if (transaction.request.method !== 'GET') {
    transaction.request.body += '&username=test_user&password=test_pass';
  } else {
    transaction.request.uri += '&username=test_user&password=test_pass';
  }
  transaction.request.headers['Content-Type'] = 'application/x-www-form-urlencoded';
});

hooks.afterEach((transaction) => {
  // Log results
  console.log(`${transaction.name}: ${transaction.status}`);
});

hooks.beforeValidation((transaction) => {
  // Normalize response
  if (transaction.real.body) {
    try {
      JSON.parse(transaction.real.body);
    } catch (e) {
      // Body is not JSON, skip validation
    }
  }
});
```

### Step 5: Run Contract Tests

```bash
# Run Pact tests
npx jest pact/

# Generate contracts
npx pact-broker publish ./pact/contracts \
  --broker-base-url=https://pact-broker.example.com \
  --consumer-version=1.0.0

# Verify with Dredd
npx dredd

# Run all contract tests
npm test:contracts
```

### Step 6: CI/CD Integration
```yaml
# .github/workflows/contract-tests.yml
name: API Contract Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  contract-tests:
    runs-on: ubuntu-latest
    services:
      whmcs:
        image: whmcs:latest
        ports:
          - 80:80
        env:
          DB_HOST: mysql
          DB_NAME: whmcs
          DB_USER: whmcs
          DB_PASS: whmcs

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Run contract tests
        run: npm test:contracts

      - name: Publish contracts
        if: github.ref == 'refs/heads/main'
        run: npx pact-broker publish ./pact/contracts
        env:
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
```

## API Contract Testing Checklist

### Request Validation
- [ ] All required parameters present
- [ ] Parameter types correct
- [ ] Parameter constraints validated
- [ ] Authentication headers present

### Response Validation
- [ ] Status codes correct
- [ ] Response headers valid
- [ ] Response body matches schema
- [ ] Error responses conform to spec

### Security
- [ ] Authentication required
- [ ] Rate limiting enforced
- [ ] Input sanitized
- [ ] Sensitive data redacted

### Documentation
- [ ] API documented
- [ ] Examples provided
- [ ] Error codes documented
- [ ] Changelog maintained

# WHMCS API Documentation Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for documenting WHMCS module APIs and creating developer guides.

## When to Use

- Creating API documentation
- Writing developer guides
- Building integration references

## Documentation Patterns

### API Reference Format

```markdown
# {Module} API Reference

## Endpoints

### Create Server
```
POST /api/servers
```

**Request:**
```json
{
    "hostname": "server.example.com",
    "plan": "starter",
    "region": "us-east"
}
```

**Response:**
```json
{
    "id": "srv_123",
    "ip": "192.168.1.1",
    "status": "creating"
}
```

### Get Server
```
GET /api/servers/{id}
```

## Authentication

All requests require Bearer token authentication:
```
Authorization: Bearer {api_key}
```

## Error Codes

| Code | Description |
|------|-------------|
| 400 | Invalid request |
| 401 | Authentication failed |
| 404 | Resource not found |
| 429 | Rate limit exceeded |
| 500 | Server error |
```

## Checklist

- [ ] Endpoint documentation
- [ ] Request/response examples
- [ ] Authentication guide
- [ ] Error codes reference
- [ ] Rate limits

---

**Related Skills:**
- whmcs-api-integration
- whmcs-testing-qa
- whmcs-deployment
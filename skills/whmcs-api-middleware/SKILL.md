# WHMCS API Middleware Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing middleware patterns in WHMCS API modules.

## When to Use

- API request processing
- Authentication middleware
- Rate limiting middleware

## Middleware Patterns

```php
<?php
abstract class Middleware {
    protected ?Middleware $next = null;

    public function setNext(Middleware $middleware): Middleware {
        $this->next = $middleware;
        return $middleware;
    }

    abstract public function handle(array $request): array;

    protected function passToNext(array $request): array {
        if ($this->next) {
            return $this->next->handle($request);
        }
        return $this->process($request);
    }

    abstract protected function process(array $request): array;
}

class AuthMiddleware extends Middleware {
    public function handle(array $request): array {
        if (empty($request['token'])) {
            return ['error' => 'Unauthorized', 'code' => 401];
        }

        $token = validateToken($request['token']);
        if (!$token) {
            return ['error' => 'Invalid token', 'code' => 401];
        }

        $request['user_id'] = $token['user_id'];
        return $this->passToNext($request);
    }

    protected function process(array $request): array {
        return $request;
    }
}

class RateLimitMiddleware extends Middleware {
    private int $maxRequests = 60;

    public function handle(array $request): array {
        $identifier = $request['user_id'] ?? $_SERVER['REMOTE_ADDR'];

        if (!$this->checkRateLimit($identifier)) {
            return ['error' => 'Rate limit exceeded', 'code' => 429];
        }

        return $this->passToNext($request);
    }

    protected function process(array $request): array {
        return $request;
    }
}
```

## Middleware Chain Usage

```php
function handleApiRequest(array $request): array {
    $chain = new AuthMiddleware();
    $chain->setNext(new RateLimitMiddleware())
          ->setNext(new ValidationMiddleware());

    return $chain->handle($request);
}
```

---

**Related Skills:**
- whmcs-security-hardening
- whmcs-rest-api-builder
- whmcs-rate-limiting

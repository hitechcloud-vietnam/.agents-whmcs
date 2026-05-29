# WHMCS User Authorization

## Overview
Guide for implementing authorization flows in WHMCS. Covers OAuth2, API authorization, and token management.

## Authorization System

### OAuth2 Implementation

```php
<?php
// /includes/hooks/authorization.php

/**
 * OAuth2 Authorization Server Implementation
 */

// Authorization Endpoint
add_hook("OAuthAuthorize", 1, function(array $params) {
    $clientId = $params["client_id"];
    $redirectUri = $params["redirect_uri"];
    $scope = $params["scope"];
    $state = $params["state"];
    $userId = $_SESSION["uid"];
    
    // Validate client
    $client = Capsule::table("mod_oauth_clients")
        ->where("client_id", $clientId)
        ->first();
    
    if (!$client) {
        return ["error" => "Invalid client application"];
    }
    
    if ($client->redirect_uri !== $redirectUri) {
        return ["error" => "Invalid redirect URI"];
    }
    
    // Check if user has authorized this app before
    $existingAuth = Capsule::table("mod_oauth_authorizations")
        ->where("user_id", $userId)
        ->where("client_id", $clientId)
        ->where("scope", $scope)
        ->first();
    
    if ($existingAuth && isset($_GET["approve"])) {
        // Re-authorize
        $authorizationCode = generateAuthorizationCode();
        
        Capsule::table("mod_oauth_codes")->insert([
            "code" => $authorizationCode,
            "client_id" => $clientId,
            "user_id" => $userId,
            "redirect_uri" => $redirectUri,
            "scope" => $scope,
            "expires_at" => date("Y-m-d H:i:s", strtotime("+10 minutes")),
            "created_at" => date("Y-m-d H:i:s")
        ]);
        
        return [
            "redirect" => "{$redirectUri}?code={$authorizationCode}&state={$state}"
        ];
    }
    
    if (isset($_GET["deny"])) {
        return ["redirect" => "{$redirectUri}?error=access_denied&state={$state}"];
    }
    
    // Show authorization prompt (handled by template)
    return [
        "show_prompt" => true,
        "client_name" => $client->name,
        "client_logo" => $client->logo_url,
        "scope" => $scope,
        "requested_permissions" => parseScope($scope)
    ];
});

function parseScope(string $scope): array
{
    $permissions = [];
    $scopes = explode(" ", $scope);
    
    foreach ($scopes as $s) {
        $permissions[] = [
            "name" => $s,
            "description" => getScopeDescription($s)
        ];
    }
    
    return $permissions;
}
```

### Token Management

```php
// Token Endpoint
add_hook("OAuthToken", 1, function(array $params) {
    $grantType = $params["grant_type"];
    
    switch ($grantType) {
        case "authorization_code":
            return handleAuthorizationCodeGrant($params);
            
        case "refresh_token":
            return handleRefreshTokenGrant($params);
            
        case "client_credentials":
            return handleClientCredentialsGrant($params);
            
        case "password":
            return handlePasswordGrant($params);
    }
    
    return ["error" => "unsupported_grant_type"];
});

function handleAuthorizationCodeGrant(array $params): array
{
    $code = $params["code"];
    $redirectUri = $params["redirect_uri"];
    
    // Find and validate code
    $authCode = Capsule::table("mod_oauth_codes")
        ->where("code", $code)
        ->where("expires_at", ">", date("Y-m-d H:i:s"))
        ->where("used", 0)
        ->first();
    
    if (!$authCode || $authCode->redirect_uri !== $redirectUri) {
        return ["error" => "invalid_grant", "error_description" => "Invalid authorization code"];
    }
    
    // Mark code as used
    Capsule::table("mod_oauth_codes")
        ->where("id", $authCode->id)
        ->update(["used" => 1, "used_at" => date("Y-m-d H:i:s")]);
    
    // Generate tokens
    return generateTokens($authCode->client_id, $authCode->user_id, $authCode->scope);
}

function handleRefreshTokenGrant(array $params): array
{
    $refreshToken = $params["refresh_token"];
    
    $token = Capsule::table("mod_oauth_refresh_tokens")
        ->where("token", hash("sha256", $refreshToken))
        ->where("expires_at", ">", date("Y-m-d H:i:s"))
        ->where("revoked", 0)
        ->first();
    
    if (!$token) {
        return ["error" => "invalid_grant"];
    }
    
    // Revoke old refresh token
    Capsule::table("mod_oauth_refresh_tokens")
        ->where("id", $token->id)
        ->update(["revoked" => 1, "revoked_at" => date("Y-m-d H:i:s")]);
    
    // Generate new tokens
    return generateTokens($token->client_id, $token->user_id, $token->scope);
}

function generateTokens(string $clientId, int $userId, string $scope): array
{
    $accessToken = bin2hex(random_bytes(32));
    $refreshToken = bin2hex(random_bytes(32));
    $accessTokenExpiry = date("Y-m-d H:i:s", strtotime("+1 hour"));
    $refreshTokenExpiry = date("Y-m-d H:i:s", strtotime("+30 days"));
    
    // Store access token
    Capsule::table("mod_oauth_access_tokens")->insert([
        "token" => hash("sha256", $accessToken),
        "client_id" => $clientId,
        "user_id" => $userId,
        "scope" => $scope,
        "expires_at" => $accessTokenExpiry,
        "created_at" => date("Y-m-d H:i:s")
    ]);
    
    // Store refresh token
    Capsule::table("mod_oauth_refresh_tokens")->insert([
        "token" => hash("sha256", $refreshToken),
        "client_id" => $clientId,
        "user_id" => $userId,
        "scope" => $scope,
        "expires_at" => $refreshTokenExpiry,
        "created_at" => date("Y-m-d H:i:s")
    ]);
    
    return [
        "access_token" => $accessToken,
        "token_type" => "Bearer",
        "expires_in" => 3600,
        "refresh_token" => $refreshToken,
        "scope" => $scope
    ];
}
```

### API Authorization

```php
function authorizeAPIRequest(string $token): ?array
{
    $tokenHash = hash("sha256", $token);
    
    $accessToken = Capsule::table("mod_oauth_access_tokens")
        ->where("token", $tokenHash)
        ->where("expires_at", ">", date("Y-m-d H:i:s"))
        ->first();
    
    if (!$accessToken) {
        return null;
    }
    
    return [
        "client_id" => $accessToken->client_id,
        "user_id" => $accessToken->user_id,
        "scope" => $accessToken->scope
    ];
}

function requireScope(string $requiredScope): void
{
    $auth = $_SERVER["HTTP_AUTHORIZATION"] ?? "";
    
    if (!preg_match("/Bearer\s+(.+)/i", $auth, $matches)) {
        header("HTTP/1.1 401 Unauthorized");
        header("WWW-Authenticate: Bearer");
        echo json_encode(["error" => "missing_token"]);
        exit;
    }
    
    $token = $matches[1];
    $authData = authorizeAPIRequest($token);
    
    if (!$authData) {
        header("HTTP/1.1 401 Unauthorized");
        echo json_encode(["error" => "invalid_token"]);
        exit;
    }
    
    $scopes = explode(" ", $authData["scope"]);
    if (!in_array($requiredScope, $scopes)) {
        header("HTTP/1.1 403 Forbidden");
        echo json_encode(["error" => "insufficient_scope"]);
        exit;
    }
    
    $_SESSION["api_auth"] = $authData;
}
```

### Token Revocation

```php
add_hook("OAuthRevoke", 1, function(array $params) {
    $token = $params["token"];
    $tokenTypeHint = $params["token_type_hint"] ?? "access_token";
    
    $tokenHash = hash("sha256", $token);
    
    if ($tokenTypeHint === "access_token") {
        Capsule::table("mod_oauth_access_tokens")
            ->where("token", $tokenHash)
            ->update(["revoked" => 1, "revoked_at" => date("Y-m-d H:i:s")]);
    } else {
        Capsule::table("mod_oauth_refresh_tokens")
            ->where("token", $tokenHash)
            ->update(["revoked" => 1, "revoked_at" => date("Y-m-d H:i:s")]);
    }
    
    return ["success" => true];
});
```

## Best Practices

1. **Secure Token Generation**: Use cryptographically secure random bytes
2. **Short-lived Tokens**: Access tokens should expire quickly
3. **Refresh Tokens**: Provide secure refresh mechanisms
4. **Token Storage**: Hash tokens before database storage
5. **Scope Enforcement**: Validate requested scopes
6. **Token Revocation**: Support token revocation
7. **Client Validation**: Validate OAuth clients
8. **PKCE for Public Clients**: Use PKCE for mobile/SPA apps

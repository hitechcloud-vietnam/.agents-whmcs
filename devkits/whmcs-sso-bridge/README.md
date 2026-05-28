# WHMCS SSO Bridge DevKit

A comprehensive SSO bridge module for integrating WHMCS with external authentication providers like OAuth2, SAML, and custom SSO systems.

## Features

- Multiple SSO provider support (OAuth2, SAML, JWT)
- Automatic user synchronization
- Session bridging to WHMCS
- Secure token management
- Configurable user attribute mapping
- Comprehensive logging
- Admin interface for provider management

## Installation

1. Copy module files to:
   ```
   modules/addons/whmcs_sso_bridge/
   ```

2. Activate the module in WHMCS Admin > Addon Modules

3. Configure SSO providers in the admin interface

## Configuration

### OAuth2 Provider Setup

1. Create an OAuth application in your identity provider
2. Set callback URL to:
   ```
   https://your-whmcs.com/modules/addons/whmcs_sso_bridge/callback.php
   ```
3. Configure the provider in admin:
   - Client ID
   - Client Secret
   - Authorization URL
   - Token URL
   - User Info URL
   - Scopes

### SAML Provider Setup

1. Configure your IdP (Identity Provider)
2. Set ACS URL to:
   ```
   https://your-whmcs.com/modules/addons/whmcs_sso_bridge/callback.php
   ```
3. Configure SP metadata in admin panel

## Usage

### Initiate SSO Login

Redirect users to the SSO login page:
```
https://your-whmcs.com/modules/addons/whmcs_sso_bridge/initiate.php?provider=google
```

### Callback Handling

The callback handler automatically:
1. Validates the authentication response
2. Retrieves user information
3. Creates/updates WHMCS client
4. Creates SSO session
5. Bridges to WHMCS session
6. Redirects to client area

### Programmatic SSO

```php
// In your custom code
use SsoBridge\TokenManager;
use SsoBridge\SessionBridge;

// Validate token
$session = TokenManager::validate($token);

if ($session) {
    // Bridge to WHMCS
    SessionBridge::bridgeToWhmcs($token);
}
```

## Token Management

```php
// Create session token
$token = TokenManager::createSession($userId, 'oauth2', 3600);

// Validate token
$session = TokenManager::validate($token);

// Refresh token
TokenManager::refresh($token, 3600);

// Invalidate (logout)
TokenManager::invalidate($token);

// Invalidate all user sessions
TokenManager::invalidateUser($userId);

// Cleanup expired sessions
TokenManager::cleanup();
```

## Session Bridge

```php
// Bridge SSO session to WHMCS
SessionBridge::bridgeToWhmcs($token);

// Create WHMCS session directly
SessionBridge::createWhmcsSession($userId);

// Clear session (logout)
SessionBridge::clearSession($token);
```

## User Attribute Mapping

Configure how external user attributes map to WHMCS fields:

| External Field | WHMCS Field |
|---------------|-------------|
| id / sub | external_id |
| email | email |
| given_name / first_name | firstname |
| family_name / last_name | lastname |
| name | firstname + lastname |
| picture | (stored in custom field) |

## Supported Providers

### OAuth2
Works with any OAuth2-compatible provider:
- Google
- Microsoft Azure AD
- Facebook
- GitHub
- Custom OAuth servers

### SAML
Works with any SAML2-compatible IdP:
- Okta
- Azure AD
- OneLogin
- Custom SAML servers

## Security

- Tokens are stored hashed (SHA-256)
- Session tokens are cryptographically secure (64 bytes)
- Automatic session expiration
- IP-based session validation (optional)
- Comprehensive audit logging

## Hooks

The SSO Bridge integrates with WHMCS hooks:

- `ClientAreaPageLogin` - Adds SSO login buttons
- `ClientLogin` - Tracks SSO logins
- `AfterClientLogin` - Post-login actions

## File Structure

```
whmcs-sso-bridge/
├── sso-bridge.php          # Main module
├── callback.php            # OAuth/SAML callback handler
├── initiate.php           # SSO initiation endpoint
├── lib/
│   ├── SsoProvider.php     # Provider interface & implementations
│   ├── TokenManager.php    # Token generation/validation
│   └── SessionBridge.php  # Session bridging
└── templates/
    └── sso-config.tpl      # Configuration template
```

## Requirements

- WHMCS 7.0+
- PHP 7.4+
- cURL extension
- OpenSSL extension (for token generation)

## Support

For issues and feature requests, please contact the developer.
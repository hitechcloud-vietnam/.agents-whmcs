# WHMCS Configuration Management Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for managing module configuration and settings.

## When to Use

- Building configuration systems
- Managing multiple environments
- Storing encrypted credentials

## Config Patterns

### Settings Manager

```php
class ConfigManager {
    private string $module;

    public function __construct(string $module) {
        $this->module = $module;
    }

    public function get(string $key, $default = null) {
        $value = Capsule::table('mod_' . $this->module . '_settings')
            ->where('setting_key', $key)
            ->value('setting_value');

        return $value ?? $default;
    }

    public function set(string $key, $value): void {
        Capsule::table('mod_' . $this->module . '_settings')
            ->updateOrInsert(
                ['setting_key' => $key],
                ['setting_value' => $value, 'updated_at' => date('Y-m-d H:i:s')]
            );
    }

    public function getAll(): array {
        return Capsule::table('mod_' . $this->module . '_settings')
            ->pluck('setting_value', 'setting_key')
            ->toArray();
    }

    public function delete(string $key): void {
        Capsule::table('mod_' . $this->module . '_settings')
            ->where('setting_key', $key)
            ->delete();
    }

    // Environment-based config
    public function getApiCredentials(): array {
        $env = $this->get('environment', 'production');

        if ($env === 'sandbox') {
            return [
                'api_key' => $this->getDecrypted('sandbox_api_key'),
                'api_secret' => $this->getDecrypted('sandbox_api_secret'),
            ];
        }

        return [
            'api_key' => $this->getDecrypted('production_api_key'),
            'api_secret' => $this->getDecrypted('production_api_secret'),
        ];
    }
}
```

### Encrypted Storage

```php
public function setEncrypted(string $key, string $value): void {
    $encrypted = \Illuminate\Support\Facades\Crypt::encrypt($value);
    $this->set($key, $encrypted);
}

public function getDecrypted(string $key): string {
    $encrypted = $this->get($key);
    if (!$encrypted) return '';

    try {
        return \Illuminate\Support\Facades\Crypt::decrypt($encrypted);
    } catch (\Exception $e) {
        return '';
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-security-hardening
- whmcs-database-design
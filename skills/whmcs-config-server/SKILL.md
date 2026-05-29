# WHMCS Configuration Server Skill

## Purpose
Provides patterns for implementing a configuration server in WHMCS, centralizing configuration management, enabling feature flags, and managing application settings.

## Implementation Patterns

### Config Server
```php
<?php
class ConfigServer {
    private $db;
    private $cache;
    
    public function __construct() {
        $this->cache = \WHMCS\Application\Services\CacheService::getInstance();
    }
    
    public function get($key, $default = null) {
        $cacheKey = "config_{$key}";
        $cached = $this->cache->get($cacheKey);
        
        if ($cached !== null) {
            return $cached;
        }
        
        $config = $this->db->select(
            "SELECT value FROM mod_config WHERE config_key = ?",
            [$key]
        );
        
        $value = $config['value'] ?? $default;
        
        $this->cache->set($cacheKey, $value, 300);
        
        return $value;
    }
    
    public function set($key, $value, $options = []) {
        $existing = $this->db->select(
            "SELECT id FROM mod_config WHERE config_key = ?",
            [$key]
        );
        
        $data = [
            'config_key' => $key,
            'value' => $value,
            'type' => $options['type'] ?? 'string',
            'environment' => $options['environment'] ?? 'all',
            'updated_at' => date('Y-m-d H:i:s')
        ];
        
        if ($existing) {
            $this->db->where('config_key', $key)->update('mod_config', $data);
        } else {
            $this->db->insert('mod_config', $data);
        }
        
        $this->cache->delete("config_{$key}");
        
        return true;
    }
    
    public function getNamespace($namespace) {
        $configs = $this->db->select(
            "SELECT config_key, value FROM mod_config WHERE config_key LIKE ?",
            ["{$namespace}.%"]
        );
        
        $result = [];
        foreach ($configs as $config) {
            $key = str_replace("{$namespace}.", '', $config['config_key']);
            $result[$key] = $config['value'];
        }
        
        return $result;
    }
    
    public function delete($key) {
        $this->db->where('config_key', $key)->delete('mod_config');
        $this->cache->delete("config_{$key}");
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_config (
    id INT AUTO_INCREMENT PRIMARY KEY,
    config_key VARCHAR(255) UNIQUE,
    value TEXT,
    type VARCHAR(50),
    environment VARCHAR(50),
    updated_at DATETIME
);
```

## Usage Examples
```php
$config = new ConfigServer();

// Get value
$apiUrl = $config->get('api.base_url', 'https://api.example.com');

// Set value
$config->set('feature.new_dashboard', true);

// Get namespace
$apiConfig = $config->getNamespace('api');
```

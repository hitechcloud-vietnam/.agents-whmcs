# WHMCS Serialization

Complete guide to data serialization patterns.

## Overview

Handle data serialization for storage and transmission.

## Serializer

```php
<?php
/**
 * Data serializer
 */
class Serializer
{
    /**
     * Serialize to JSON
     */
    public function toJson(array $data): string
    {
        return json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
    }
    
    /**
     * Deserialize from JSON
     */
    public function fromJson(string $json): array
    {
        return json_decode($json, true) ?? [];
    }
    
    /**
     * Serialize to XML
     */
    public function toXml(array $data, string $rootElement = 'root'): string
    {
        $xml = new SimpleXMLElement("<{$rootElement}/>");
        
        $this->arrayToXml($data, $xml);
        
        return $xml->asXML();
    }
    
    /**
     * Array to XML helper
     */
    private function arrayToXml(array $data, SimpleXMLElement $xml): void
    {
        foreach ($data as $key => $value) {
            if (is_numeric($key)) {
                $key = 'item';
            }
            
            if (is_array($value)) {
                $child = $xml->addChild($key);
                $this->arrayToXml($value, $child);
            } else {
                $xml->addChild($key, htmlspecialchars($value));
            }
        }
    }
}
```

## Best Practices

1. **JSON for APIs** - Use JSON for web services
2. **Versioning** - Include version in serialized data
3. **Compression** - Compress large data
4. **Validation** - Validate before serialization
5. **Encoding** - Handle Unicode properly
6. **Security** - Sanitize untrusted data

## Related Documentation

- [whmcs-advanced-api.md](whmcs-advanced-api.md)
- [whmcs-advanced-database.md](whmcs-advanced-database.md)

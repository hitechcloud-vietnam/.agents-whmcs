# WHMCS IDN Domains

## Concept
Internationalized Domain Name support.

## Code
```php
<?php
class IDNDomains {
    public static function convertToPunycode($domain) {
        return idn_to_ascii($domain);
    }
    
    public static function convertToUnicode($domain) {
        return idn_to_utf8($domain);
    }
}
```

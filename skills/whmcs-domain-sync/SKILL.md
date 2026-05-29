# WHMCS Domain Sync

## Concept
Synchronization with domain registrars.

## Code
```php
<?php
class DomainSync {
    public static function syncWithRegistrar($registrarId) {
        return localAPI("SyncRegistrarDomains", ["registrar" => $registrarId]);
    }
}
add_hook("DailyCronJob", 1, function() {
    $registrars = getActiveRegistrars();
    foreach ($registrars as $registrar) {
        DomainSync::syncWithRegistrar($registrar);
    }
});
```

# WHMCS Privacy Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build domain privacy/proxy services modules.

## Privacy Module Structure

```php
<?php
/**
 * Privacy Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Domain Privacy',
        'description' => 'WHOIS privacy protection service',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_privacy_services', function($t) {
        $t->increments('id');
        $t->integer('service_id')->unsigned();
        $t->string('domain');
        $t->string('privacy_contact_id');
        $t->boolean('auto_renew')->default(true);
        $t->date('expires_at');
        $t->timestamps();
    });

    Capsule::schema()->create('mod_privacy_log', function($t) {
        $t->increments('id');
        $t->integer('privacy_id')->unsigned();
        $t->string('action', 50);
        $t->text('details');
        $t->timestamps();
    });

    return ['status' => 'success', 'description' => 'Privacy module activated'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_privacy_services');
    Capsule::schema()->dropIfExists('mod_privacy_log');
    return ['status' => 'success'];
}
```

## Privacy Service Management

```php
public function enablePrivacy(int $serviceId, string $domain): bool {
    $privacyContact = $this->registrarApi->createPrivacyContact([
        'name' => 'Privacy Service',
        'email' => 'privacy@' . $this->getTld($domain),
        'phone' => '+1.5550000000',
        'address' => [
            'street1' => '123 Privacy St',
            'city' => 'Anytown',
            'state' => 'CA',
            'postal_code' => '12345',
            'country' => 'US',
        ],
    ]);

    $expiresAt = date('Y-m-d', strtotime('+1 year'));

    $id = Capsule::table('mod_privacy_services')->insertGetId([
        'service_id' => $serviceId,
        'domain' => $domain,
        'privacy_contact_id' => $privacyContact['id'],
        'auto_renew' => true,
        'expires_at' => $expiresAt,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    $this->logAction($id, 'enabled', "Privacy enabled for {$domain}");

    return true;
}

public function disablePrivacy(int $serviceId): bool {
    $privacy = Capsule::table('mod_privacy_services')
        ->where('service_id', $serviceId)
        ->first();

    if (!$privacy) {
        return false;
    }

    $this->registrarApi->deletePrivacyContact($privacy->privacy_contact_id);

    Capsule::table('mod_privacy_services')
        ->where('id', $privacy->id)
        ->delete();

    $this->logAction($privacy->id, 'disabled', 'Privacy service removed');

    return true;
}

public function renewPrivacy(int $serviceId): bool {
    $privacy = Capsule::table('mod_privacy_services')
        ->where('service_id', $serviceId)
        ->first();

    if (!$privacy) {
        return false;
    }

    $newExpiry = date('Y-m-d', strtotime('+1 year'));

    Capsule::table('mod_privacy_services')
        ->where('id', $privacy->id)
        ->update(['expires_at' => $newExpiry]);

    $this->logAction($privacy->id, 'renewed', "Privacy renewed until {$newExpiry}");

    return true;
}
```

## WHOIS Data Retrieval

```php
public function getForwardingEmail(int $serviceId): ?string {
    $privacy = Capsule::table('mod_privacy_services')
        ->where('service_id', $serviceId)
        ->first();

    return $privacy ? $privacy->forwarding_email : null;
}

public function setForwardingEmail(int $serviceId, string $email): bool {
    $privacy = Capsule::table('mod_privacy_services')
        ->where('service_id', $serviceId)
        ->first();

    if (!$privacy) {
        return false;
    }

    $this->logAction($privacy->id, 'forwarding_updated', "Forwarding set to {$email}");

    return true;
}

public function getLeakAlerts(int $serviceId): array {
    return Capsule::table('mod_privacy_leaks')
        ->where('service_id', $serviceId)
        ->orderBy('detected_at', 'desc')
        ->limit(20)
        ->get();
}
```

## Automated Renewal Hook

```php
add_hook('DailyCronJob', 1, function() {
    $expiringPrivacy = Capsule::table('mod_privacy_services')
        ->where('auto_renew', true)
        ->whereRaw('DATEDIFF(expires_at, CURDATE()) <= 30')
        ->get();

    foreach ($expiringPrivacy as $privacy) {
        $service = Capsule::table('tblhosting')
            ->where('id', $privacy->service_id)
            ->first();

        if ($service && $service->autorenew) {
            $module = new PrivacyModule();
            $module->renewPrivacy($privacy->service_id);

            send_email([
                'id' => $service->userid,
                'type' => 'admin',
                'subject' => 'Privacy Service Renewed',
                'message' => "Privacy service for {$privacy->domain} has been renewed.",
            ]);
        }
    }
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-registrar-builder
- whmcs-cron-automation
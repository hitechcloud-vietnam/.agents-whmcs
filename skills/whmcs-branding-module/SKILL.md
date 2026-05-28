# WHMCS Branding Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build white-label/branding modules for reseller environments.

## Branding Module Structure

```php
<?php
/**
 * Branding Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Custom Branding',
        'description' => 'White-label branding management',
        'version' => '1.0',
        'author' => 'Developer',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_branding_profiles', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('name');
        $t->string('logo_url');
        $t->string('favicon_url');
        $t->string('primary_color', 7)->default('#0068ff');
        $t->string('secondary_color', 7)->default('#ffffff');
        $t->text('custom_css');
        $t->text('custom_js');
        $t->string('footer_text');
        $t->boolean('enabled')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_branding_domains', function($t) {
        $t->increments('id');
        $t->integer('profile_id')->unsigned();
        $t->string('domain');
        $t->boolean('is_default')->default(false);
        $t->timestamps();
    });

    return ['status' => 'success', 'description' => 'Branding module activated'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_branding_profiles');
    Capsule::schema()->dropIfExists('mod_branding_domains');
    return ['status' => 'success'];
}
```

## Branding Profile Management

```php
public function createProfile(int $userId, array $data): int {
    $id = Capsule::table('mod_branding_profiles')->insertGetId([
        'user_id' => $userId,
        'name' => $data['name'],
        'logo_url' => $data['logo_url'] ?? '',
        'favicon_url' => $data['favicon_url'] ?? '',
        'primary_color' => $data['primary_color'] ?? '#0068ff',
        'secondary_color' => $data['secondary_color'] ?? '#ffffff',
        'custom_css' => $data['custom_css'] ?? '',
        'custom_js' => $data['custom_js'] ?? '',
        'footer_text' => $data['footer_text'] ?? '',
        'enabled' => true,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return $id;
}

public function updateProfile(int $profileId, array $data): bool {
    $update = array_filter([
        'name' => $data['name'] ?? null,
        'logo_url' => $data['logo_url'] ?? null,
        'favicon_url' => $data['favicon_url'] ?? null,
        'primary_color' => $data['primary_color'] ?? null,
        'secondary_color' => $data['secondary_color'] ?? null,
        'custom_css' => $data['custom_css'] ?? null,
        'custom_js' => $data['custom_js'] ?? null,
        'footer_text' => $data['footer_text'] ?? null,
        'enabled' => $data['enabled'] ?? null,
    ], fn($v) => $v !== null);

    return Capsule::table('mod_branding_profiles')
        ->where('id', $profileId)
        ->update($update) > 0;
}
```

## Domain Mapping

```php
public function addDomain(int $profileId, string $domain, bool $isDefault = false): int {
    if ($isDefault) {
        Capsule::table('mod_branding_domains')
            ->where('profile_id', $profileId)
            ->update(['is_default' => false]);
    }

    return Capsule::table('mod_branding_domains')->insertGetId([
        'profile_id' => $profileId,
        'domain' => $domain,
        'is_default' => $isDefault,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

public function getProfileByDomain(string $domain): ?array {
    $mapping = Capsule::table('mod_branding_domains')
        ->where('domain', $domain)
        ->first();

    if (!$mapping) {
        return null;
    }

    return Capsule::table('mod_branding_profiles')
        ->where('id', $mapping->profile_id)
        ->where('enabled', true)
        ->first();
}
```

## CSS/JS Generation

```php
public function generateBrandingCSS(int $profileId): string {
    $profile = Capsule::table('mod_branding_profiles')
        ->where('id', $profileId)
        ->first();

    $css = ":root {";
    $css .= "--brand-primary: {$profile->primary_color};";
    $css .= "--brand-secondary: {$profile->secondary_color};";
    $css .= "}";

    $css .= $profile->custom_css;

    return $css;
}

public function generateBrandingJS(int $profileId): string {
    $profile = Capsule::table('mod_branding_profiles')
        ->where('id', $profileId)
        ->first();

    $js = "const BRAND_CONFIG = " . json_encode([
        'logo_url' => $profile->logo_url,
        'favicon_url' => $profile->favicon_url,
        'primary_color' => $profile->primary_color,
        'footer_text' => $profile->footer_text,
    ]) . ";";

    $js .= $profile->custom_js;

    return $js;
}
```

## Hook Integration

```php
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    $userId = $_SESSION['uid'] ?? null;
    if (!$userId) return '';

    $profile = Capsule::table('mod_branding_profiles')
        ->where('user_id', $userId)
        ->where('enabled', true)
        ->first();

    if (!$profile) return '';

    $module = new BrandingModule();

    $html = '<link rel="stylesheet" href="data:text/css,' .
        urlencode($module->generateBrandingCSS($profile->id)) . '">';

    $html .= '<script>' . $module->generateBrandingJS($profile->id) . '</script>';

    return $html;
});

add_hook('ClientAreaPageOutput', 1, function($vars) {
    $userId = $_SESSION['uid'] ?? null;
    if (!$userId) return;

    $profile = Capsule::table('mod_branding_profiles')
        ->where('user_id', $userId)
        ->where('enabled', true)
        ->first();

    if ($profile && $profile->logo_url) {
        $smarty = $vars['smarty'];
        $smarty->assign('BRAND_LOGO', $profile->logo_url);
    }
});
```

## Admin Output

```php
function {module}_output(array $vars): void {
    $action = $vars['action'] ?? 'profiles';

    echo '<div class="header bg-primary"><h3>Branding Management</h3></div>';

    switch ($action) {
        case 'profiles':
            echo $this->renderProfilesList();
            break;
        case 'edit':
            echo $this->renderProfileForm($vars['id'] ?? null);
            break;
        case 'domains':
            echo $this->renderDomainsList($vars['profile_id'] ?? null);
            break;
    }
}

private function renderProfilesList(): string {
    $profiles = Capsule::table('mod_branding_profiles')
        ->orderBy('created_at', 'desc')
        ->get();

    $html = '<table class="table"><thead><tr>';
    $html .= '<th>Name</th><th>Primary Color</th><th>Status</th><th>Actions</th>';
    $html .= '</tr></thead><tbody>';

    foreach ($profiles as $profile) {
        $html .= '<tr>';
        $html .= "<td>{$profile->name}</td>";
        $html .= "<td><span style='background:{$profile->primary_color};
            padding: 4px 12px; border-radius: 4px;'>&nbsp;</span></td>";
        $html .= "<td>" . ($profile->enabled ? 'Active' : 'Disabled') . "</td>";
        $html .= "<td><a href='?action=edit&id={$profile->id}'>Edit</a></td>";
        $html .= '</tr>';
    }

    $html .= '</tbody></table>';
    return $html;
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-template-styling
- whmcs-multi-tenant-module
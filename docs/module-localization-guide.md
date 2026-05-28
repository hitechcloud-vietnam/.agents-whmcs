# WHMCS Module Localization Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers creating multi-language module support for WHMCS, including language file creation, string translation, RTL support, and locale-specific formatting.

---

## Language File Structure

### Directory Structure

```
modules/addons/your_addon/
├── your_addon.php
└── lang/
    ├── english.php
    ├── spanish.php
    ├── french.php
    └── german.php
```

### Language File Format

```php
<?php
/**
 * English Language Strings
 * 
 * @package WHMCS\Module\YourAddon
 */

$lang = [
    // Module identification
    'your_addon' => [
        'name'       => 'Your Addon Name',
        'title'      => 'Advanced Analytics',
        'description'=> 'Track client behavior and engagement',
    ],
    
    // Menu items
    'nav' => [
        'dashboard'  => 'Dashboard',
        'settings'   => 'Settings',
        'reports'   => 'Reports',
        'export'     => 'Export Data',
    ],
    
    // Dashboard
    'dashboard' => [
        'title'          => 'Analytics Dashboard',
        'total_clients'  => 'Total Clients',
        'active_users'   => 'Active Users',
        'page_views'     => 'Page Views',
        'conversion'     => 'Conversion Rate',
        'period'         => 'Period',
        'last_30_days'   => 'Last 30 Days',
        'last_90_days'   => 'Last 90 Days',
    ],
    
    // Settings
    'settings' => [
        'title'          => 'Configuration Settings',
        'api_key'        => 'API Key',
        'api_key_desc'   => 'Enter your API key from the provider',
        'enable_tracking'=> 'Enable Tracking',
        'enable_tracking_desc' => 'Track client activity',
        'save_success'   => 'Settings saved successfully',
        'save_failed'    => 'Failed to save settings',
    ],
    
    // Reports
    'reports' => [
        'title'       => 'Activity Reports',
        'generate'    => 'Generate Report',
        'download'    => 'Download CSV',
        'no_data'     => 'No data available for the selected period',
    ],
    
    // Errors
    'error' => [
        'api_connection'  => 'Failed to connect to API',
        'invalid_config'  => 'Invalid configuration',
        'permission_denied' => 'Access denied',
        'not_found'       => 'Record not found',
    ],
    
    // Buttons
    'button' => [
        'save'    => 'Save Settings',
        'cancel'  => 'Cancel',
        'delete'  => 'Delete',
        'edit'    => 'Edit',
        'view'    => 'View Details',
        'export'  => 'Export',
    ],
    
    // Confirmation messages
    'confirm' => [
        'delete'     => 'Are you sure you want to delete this record?',
        'clear_data' => 'This will clear all tracking data. Continue?',
    ],
];
```

### Spanish Translation Example

```php
<?php
/**
 * Spanish Language Strings
 */

$lang = [
    'your_addon' => [
        'name'       => 'Tu Nombre de Complemento',
        'title'      => 'Analisis Avanzados',
        'description'=> 'Rastrea el comportamiento y compromiso del cliente',
    ],
    
    'nav' => [
        'dashboard'  => 'Panel de Control',
        'settings'   => 'Configuracion',
        'reports'    => 'Informes',
        'export'     => 'Exportar Datos',
    ],
    
    'dashboard' => [
        'title'          => 'Panel de Analisis',
        'total_clients'  => 'Total de Clientes',
        'active_users'   => 'Usuarios Activos',
        'page_views'     => 'Vistas de Pagina',
        'conversion'     => 'Tasa de Conversion',
        'period'         => 'Periodo',
        'last_30_days'   => 'Ultimos 30 Dias',
        'last_90_days'   => 'Ultimos 90 Dias',
    ],
    
    'settings' => [
        'title'          => 'Configuracion',
        'api_key'        => 'Clave API',
        'api_key_desc'   => 'Ingresa tu clave API del proveedor',
        'enable_tracking'=> 'Habilitar Rastreo',
        'enable_tracking_desc' => 'Rastrear actividad del cliente',
        'save_success'   => 'Configuracion guardada con exito',
        'save_failed'    => 'Error al guardar la configuracion',
    ],
    
    'error' => [
        'api_connection'  => 'Error al conectar con la API',
        'invalid_config'  => 'Configuracion invalida',
        'permission_denied' => 'Acceso denegado',
        'not_found'       => 'Registro no encontrado',
    ],
    
    'button' => [
        'save'    => 'Guardar Configuracion',
        'cancel'  => 'Cancelar',
        'delete'  => 'Eliminar',
        'edit'    => 'Editar',
        'view'    => 'Ver Detalles',
        'export'  => 'Exportar',
    ],
    
    'confirm' => [
        'delete'     => 'Estas seguro de que deseas eliminar este registro?',
        'clear_data' => 'Esto borrara todos los datos de rastreo. Continuar?',
    ],
];
```

## Loading Language Strings

```php
/**
 * Load module language strings
 * 
 * @param string $language Language code (optional)
 * @return array Language strings
 */
function your_addon_loadLanguage($language = null)
{
    if ($language === null) {
        $language = $_SESSION['Language'] ?? 'english';
    }
    
    // Load language file
    $langFile = __DIR__ . '/lang/' . $language . '.php';
    
    if (!file_exists($langFile)) {
        $langFile = __DIR__ . '/lang/english.php';
    }
    
    require_once $langFile;
    
    return $lang ?? [];
}

/**
 * Get lang string with fallback
 * 
 * @param string $key Dot-notation key
 * @param array $vars Substitute variables
 * @param string $language Language code
 * @return string
 */
function your_addon_lang($key, $vars = [], $language = null)
{
    static $lang = null;
    
    if ($lang === null) {
        $lang = your_addon_loadLanguage($language);
    }
    
    // Parse dot notation key
    $keys = explode('.', $key);
    $value = $lang;
    
    foreach ($keys as $k) {
        if (is_array($value) && isset($value[$k])) {
            $value = $value[$k];
        } else {
            return $key; // Return key if not found
        }
    }
    
    // Substitute variables
    if (!empty($vars)) {
        foreach ($vars as $placeholder => $replacement) {
            $value = str_replace('{' . $placeholder . '}', $replacement, $value);
        }
    }
    
    return $value;
}
```

## Using Language Strings in Code

```php
/**
 * Render with language strings
 */
function your_addon_renderDashboard($vars)
{
    // Load language
    $lang = your_addon_loadLanguage();
    
    $html = '<div class="contentbox">';
    $html .= '<h2>' . $lang['dashboard']['title'] . '</h2>';
    
    $html .= '<div class="stats">';
    $html .= '<div class="stat">';
    $html .= '<div class="label">' . $lang['dashboard']['total_clients'] . '</div>';
    $html .= '<div class="value">' . $stats['total_clients'] . '</div>';
    $html .= '</div>';
    
    $html .= '<div class="stat">';
    $html .= '<div class="label">' . $lang['dashboard']['active_users'] . '</div>';
    $html .= '<div class="value">' . $stats['active_users'] . '</div>';
    $html .= '</div>';
    $html .= '</div>';
    
    $html .= '</div>';
    
    return $html;
}

/**
 * Using with variable substitution
 */
function your_addon_formatMessage($messageKey, $data = [])
{
    $message = your_addon_lang($messageKey);
    return your_addon_lang($messageKey, $data);
}

// Usage
$message = your_addon_formatMessage('error.api_connection');
$message = your_addon_lang('settings.saved', [
    'name' => 'John Doe',
    'date' => date('Y-m-d'),
]);
```

## RTL Support

### RTL Language File

```php
<?php
/**
 * Arabic Language Strings (RTL)
 */

$lang = [
    'your_addon' => [
        'name'       => 'اسم الاضافة',
        'title'      => 'التحليلات المتقدمة',
        'description'=> 'تتبع سلوك العملاء والمشاركة',
    ],
    
    'nav' => [
        'dashboard'  => 'لوحة القيادة',
        'settings'   => 'الاعدادات',
        'reports'    => 'التقارير',
        'export'     => 'تصدير البيانات',
    ],
    
    'direction' => 'rtl',
    'align'      => 'right',
    'float'      => 'left',
];
```

### RTL Template Support

```php
/**
 * Get RTL-aware CSS classes
 * 
 * @param string $direction LTR or RTL
 * @param string $ltrClass Class for LTR languages
 * @param string $rtlClass Class for RTL languages
 * @return string Appropriate class
 */
function getRtlClass($direction, $ltrClass, $rtlClass)
{
    return ($direction === 'rtl') ? $rtlClass : $ltrClass;
}

/**
 * Determine text direction
 * 
 * @param string $language Language code
 * @return string 'ltr' or 'rtl'
 */
function getTextDirection($language)
{
    $rtlLanguages = ['arabic', 'hebrew', 'persian', 'urdu'];
    
    return in_array(strtolower($language), $rtlLanguages) ? 'rtl' : 'ltr';
}
```

```smarty
{assign var="direction" value=$language_info.text_direction}
{assign var="text_align" value={if $direction == 'rtl'}right{else}left{/if}}
{assign var="margin_side" value={if $direction == 'rtl'}right{else}left{/if}}

<div class="contentbox" style="text-align: {$text_align}">
    <div class="stats">
        <div class="stat" style="margin-{$margin_side}: 10px;">
            {$lang.your_addon.dashboard.total_clients}
        </div>
    </div>
</div>
```

## Locale Formatting

```php
/**
 * Format currency with locale
 * 
 * @param float $amount Amount
 * @param string $currency Currency code
 * @param string $language Language code
 * @return string Formatted amount
 */
function your_addon_formatCurrency($amount, $currency = 'USD', $language = null)
{
    if ($language === null) {
        $language = $_SESSION['Language'] ?? 'english';
    }
    
    $localeMap = [
        'english'  => 'en_US',
        'spanish'  => 'es_ES',
        'french'   => 'fr_FR',
        'german'   => 'de_DE',
        'arabic'   => 'ar_SA',
    ];
    
    $locale = $localeMap[$language] ?? 'en_US';
    
    setlocale(LC_MONETARY, $locale);
    
    return money_format('%i', $amount);
}

/**
 * Format date with locale
 * 
 * @param string $date Date string
 * @param string $language Language code
 * @param string $format Date format
 * @return string Formatted date
 */
function your_addon_formatDate($date, $language = null, $format = 'long')
{
    if ($language === null) {
        $language = $_SESSION['Language'] ?? 'english';
    }
    
    $timestamp = strtotime($date);
    
    $formats = [
        'long'  => '%B %d, %Y',
        'short' => '%m/%d/%Y',
        'month' => '%B %Y',
    ];
    
    $formatStr = $formats[$format] ?? $formats['long'];
    
    $localeMap = [
        'english'  => 'en_US.UTF-8',
        'spanish'  => 'es_ES.UTF-8',
        'french'   => 'fr_FR.UTF-8',
        'german'   => 'de_DE.UTF-8',
    ];
    
    setlocale(LC_TIME, $localeMap[$language] ?? 'en_US.UTF-8');
    
    return strftime($formatStr, $timestamp);
}

/**
 * Format numbers with locale
 * 
 * @param float $number Number
 * @param string $language Language code
 * @return string Formatted number
 */
function your_addon_formatNumber($number, $language = null)
{
    if ($language === null) {
        $language = $_SESSION['Language'] ?? 'english';
    }
    
    $localeMap = [
        'english'  => 'en_US',
        'spanish'  => 'es_ES',
        'french'   => 'fr_FR',
        'german'   => 'de_DE',
    ];
    
    $locale = $localeMap[$language] ?? 'en_US';
    
    setlocale(LC_NUMERIC, $locale);
    
    return number_format($number, 2, '.', ',');
}
```

## Testing Localized Strings

```php
<?php
/**
 * Localization Tests
 */

namespace Tests\Localization;

class LanguageTest extends \PHPUnit\Framework\TestCase
{
    private $language = 'spanish';
    
    /**
     * Test language file loads
     */
    public function testLanguageFileLoads()
    {
        $lang = your_addon_loadLanguage($this->language);
        
        $this->assertIsArray($lang);
        $this->assertArrayHasKey('your_addon', $lang);
    }
    
    /**
     * Test required keys exist
     */
    public function testRequiredKeysExist()
    {
        $lang = your_addon_loadLanguage($this->language);
        
        $requiredKeys = [
            'your_addon.name',
            'dashboard.title',
            'settings.title',
            'error.api_connection',
            'button.save',
        ];
        
        foreach ($requiredKeys as $key) {
            $keys = explode('.', $key);
            $value = $lang;
            
            foreach ($keys as $k) {
                $value = $value[$k] ?? null;
                if ($value === null) {
                    break;
                }
            }
            
            $this->assertNotNull($value, "Missing key: $key");
        }
    }
    
    /**
     * Test variable substitution
     */
    public function testVariableSubstitution()
    {
        $template = your_addon_lang('settings.saved');
        
        $result = your_addon_lang('settings.saved', [
            'name' => 'Test User',
            'date' => '2026-05-28',
        ]);
        
        $this->assertStringContainsString('Test User', $result);
        $this->assertStringContainsString('2026-05-28', $result);
    }
    
    /**
     * Test RTL detection
     */
    public function testRtlDetection()
    {
        $this->assertEquals('rtl', getTextDirection('arabic'));
        $this->assertEquals('ltr', getTextDirection('english'));
    }
}
```

---

## Related Skills and Workflows

- `internationalization-guide` - WHMCS i18n guide
- `client-area-theming` - Template localization
- `smarty-template-reference` - Template handling

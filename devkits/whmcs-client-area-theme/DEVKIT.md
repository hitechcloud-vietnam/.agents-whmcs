# WHMCS Client Area Theme DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-client-area-theme/
├── theme.php                 # Theme configuration
├── lib/
│   ├── ThemeConfig.php       # Theme configuration handler
│   ├── TemplateCompiler.php   # Template compilation
│   └── AssetManager.php      # Asset management
├── templates/
│   ├── client/
│   │   └── layout.tpl        # Main client area layout
│   └── overrides/
│       ├── header.tpl         # Header override
│       ├── footer.tpl         # Footer override
│       └── sidebar.tpl        # Sidebar override
├── assets/
│   ├── css/
│   │   └── custom.css         # Custom styles
│   ├── js/
│   │   └── custom.js          # Custom JavaScript
│   └── images/
└── config.json               # Theme manifest
```

## Main Theme Configuration

```php
<?php
/**
 * WHMCS Client Area Theme
 * DevKit Template
 * 
 * Custom client area theming module
 * Installation: Copy to templates/{theme_name}/
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {theme}_config(): array {
    return [
        'name' => '{Theme Name}',
        'author' => '{Author}',
        'description' => 'Custom client area theme',
        'version' => '1.0',
        'preview' => 'preview.png',
    ];
}

/**
 * Activate
 */
function {theme}_activate(): array {
    // Store theme settings
    Capsule::table('mod_{theme}_settings')->insert([
        'setting' => 'theme_config',
        'value' => json_encode([
            'primary_color' => '#007bff',
            'secondary_color' => '#6c757d',
            'accent_color' => '#28a745',
            'font_family' => 'Poppins',
            'logo_url' => '',
            'favicon_url' => '',
        ]),
    ]);
    
    // Copy custom assets
    {theme}_copyAssets();
    
    return ['status' => 'success', 'description' => 'Theme activated'];
}

/**
 * Deactivate
 */
function {theme}_deactivate(): array {
    Capsule::table('mod_{theme}_settings')->where('setting', 'LIKE', 'theme_%')->delete();
    
    return ['status' => 'success'];
}

/**
 * Copy theme assets
 */
function {theme}_copyAssets(): void {
    $sourceDir = dirname(__DIR__) . '/{theme}/assets';
    $destDir = ROOTDIR . '/assets/{theme}';
    
    if (!is_dir($destDir)) {
        mkdir($destDir, 0755, true);
    }
    
    // Copy CSS
    if (is_dir($sourceDir . '/css')) {
        $files = glob($sourceDir . '/css/*.css');
        foreach ($files as $file) {
            copy($file, $destDir . '/css/' . basename($file));
        }
    }
    
    // Copy JS
    if (is_dir($sourceDir . '/js')) {
        $files = glob($sourceDir . '/js/*.js');
        foreach ($files as $file) {
            copy($file, $destDir . '/js/' . basename($file));
        }
    }
    
    // Copy Images
    if (is_dir($sourceDir . '/images')) {
        $files = glob($sourceDir . '/images/*');
        foreach ($files as $file) {
            copy($file, $destDir . '/images/' . basename($file));
        }
    }
}

/**
 * Output function (Admin Configuration)
 */
function {theme}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'settings';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'colors':
            {theme}_showColorSettings();
            break;
        case 'fonts':
            {theme}_showFontSettings();
            break;
        case 'assets':
            {theme}_showAssetSettings();
            break;
        case 'preview':
            {theme}_showPreview();
            break;
        default:
            {theme}_showSettings();
    }
}
```

## Theme Config Handler

```php
<?php
/**
 * Theme Configuration Handler
 */

namespace ClientTheme;

use WHMCS\Database\Capsule;

class ThemeConfig {
    
    private static string $themeName;
    private static array $config = [];
    
    /**
     * Initialize theme configuration
     */
    public static function init(string $themeName): void {
        self::$themeName = $themeName;
        self::loadConfig();
    }
    
    /**
     * Load configuration from database
     */
    private static function loadConfig(): void {
        $rows = Capsule::table('mod_' . self::$themeName . '_settings')
            ->get();
        
        foreach ($rows as $row) {
            self::$config[$row->setting] = $row->value;
        }
    }
    
    /**
     * Get configuration value
     */
    public static function get(string $key, $default = null) {
        if (strpos($key, '.') !== false) {
            $keys = explode('.', $key);
            $value = self::$config;
            
            foreach ($keys as $k) {
                if (isset($value[$k])) {
                    $value = $value[$k];
                } else {
                    return $default;
                }
            }
            
            return $value;
        }
        
        $value = self::$config[$key] ?? $default;
        
        if (is_string($value) && strpos($value, '{') !== false) {
            return json_decode($value, true);
        }
        
        return $value;
    }
    
    /**
     * Set configuration value
     */
    public static function set(string $key, $value): void {
        if (is_array($value)) {
            $value = json_encode($value);
        }
        
        Capsule::table('mod_' . self::$themeName . '_settings')->updateOrInsert(
            ['setting' => $key],
            ['value' => $value]
        );
        
        self::$config[$key] = $value;
    }
    
    /**
     * Get all configuration
     */
    public static function all(): array {
        $config = [];
        
        foreach (self::$config as $key => $value) {
            $decoded = json_decode($value, true);
            $config[$key] = $decoded ?? $value;
        }
        
        return $config;
    }
    
    /**
     * Get CSS variables for theming
     */
    public static function getCssVariables(): string {
        $config = self::all();
        
        $vars = [];
        
        // Colors
        $vars[] = '--primary-color: ' . ($config['primary_color'] ?? '#007bff');
        $vars[] = '--secondary-color: ' . ($config['secondary_color'] ?? '#6c757d');
        $vars[] = '--accent-color: ' . ($config['accent_color'] ?? '#28a745');
        $vars[] = '--success-color: ' . ($config['success_color'] ?? '#28a745');
        $vars[] = '--warning-color: ' . ($config['warning_color'] ?? '#ffc107');
        $vars[] = '--danger-color: ' . ($config['danger_color'] ?? '#dc3545');
        $vars[] = '--info-color: ' . ($config['info_color'] ?? '#17a2b8');
        
        // Typography
        $vars[] = '--font-family: ' . ($config['font_family'] ?? 'Poppins, sans-serif');
        $vars[] = '--font-size-base: ' . ($config['font_size'] ?? '14px');
        $vars[] = '--font-weight-normal: 400';
        $vars[] = '--font-weight-bold: 600';
        
        // Spacing
        $vars[] = '--spacing-xs: ' . ($config['spacing_xs'] ?? '0.25rem');
        $vars[] = '--spacing-sm: ' . ($config['spacing_sm'] ?? '0.5rem');
        $vars[] = '--spacing-md: ' . ($config['spacing_md'] ?? '1rem');
        $vars[] = '--spacing-lg: ' . ($config['spacing_lg'] ?? '1.5rem');
        $vars[] = '--spacing-xl: ' . ($config['spacing_xl'] ?? '2rem');
        
        // Border radius
        $vars[] = '--border-radius: ' . ($config['border_radius'] ?? '0.25rem');
        $vars[] = '--border-radius-lg: ' . ($config['border_radius_lg'] ?? '0.5rem');
        
        // Shadows
        $vars[] = '--box-shadow: ' . ($config['box_shadow'] ?? '0 0.125rem 0.25rem rgba(0,0,0,0.075)');
        $vars[] = '--box-shadow-lg: ' . ($config['box_shadow_lg'] ?? '0 0.5rem 1rem rgba(0,0,0,0.15)');
        
        return ':root { ' . implode('; ', $vars) . ' }';
    }
    
    /**
     * Generate custom CSS
     */
    public static function generateCustomCss(): string {
        $config = self::all();
        
        $css = self::getCssVariables();
        
        // Additional custom CSS
        if (!empty($config['custom_css'])) {
            $css .= "\n" . $config['custom_css'];
        }
        
        return $css;
    }
}

/**
 * Theme Hook Registration
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $themeName = basename(__DIR__);
    ThemeConfig::init($themeName);
    
    $css = ThemeConfig::generateCustomCss();
    
    return '<style>' . $css . '</style>';
});

add_hook('ClientAreaFooterOutput', 1, function($vars) {
    // Add custom JavaScript
    $jsPath = ROOTDIR . '/assets/' . basename(__DIR__) . '/js/custom.js';
    
    if (file_exists($jsPath)) {
        return '<script src="' . $jsPath . '"></script>';
    }
    
    return '';
});
```

## Template Compiler

```php
<?php
/**
 * Template Compiler
 * Compiles and caches theme templates
 */

namespace ClientTheme;

class TemplateCompiler {
    
    private string $templateDir;
    private string $cacheDir;
    private array $variables = [];
    
    public function __construct(string $templateDir = '') {
        $this->templateDir = $templateDir ?: __DIR__ . '/../templates';
        $this->cacheDir = CACHEDIR . '/templates/' . basename($this->templateDir);
        
        if (!is_dir($this->cacheDir)) {
            mkdir($this->cacheDir, 0755, true);
        }
    }
    
    /**
     * Set template variable
     */
    public function set(string $key, $value): self {
        $this->variables[$key] = $value;
        return $this;
    }
    
    /**
     * Set multiple variables
     */
    public function setMany(array $variables): self {
        $this->variables = array_merge($this->variables, $variables);
        return $this;
    }
    
    /**
     * Render template
     */
    public function render(string $template, array $data = []): string {
        $this->setMany($data);
        
        $templatePath = $this->templateDir . '/' . $template;
        
        if (!file_exists($templatePath)) {
            throw new \Exception("Template not found: {$template}");
        }
        
        // Check cache
        $cacheKey = md5($template . filemtime($templatePath));
        $cacheFile = $this->cacheDir . '/' . $cacheKey . '.php';
        
        if (file_exists($cacheFile) && filemtime($cacheFile) > filemtime($templatePath)) {
            return $this->executeCompiled($cacheFile);
        }
        
        // Compile template
        $compiled = $this->compile(file_get_contents($templatePath));
        
        // Save to cache
        file_put_contents($cacheFile, $compiled);
        
        return $this->executeCompiled($cacheFile);
    }
    
    /**
     * Compile template to PHP
     */
    private function compile(string $template): string {
        // Replace template variables
        $template = preg_replace('/\{\{(\$?\w+(?:\.\w+)*)\}\}/', '<?php echo htmlspecialchars($this->get(\'$1\'), ENT_QUOTES); ?>', $template);
        
        // Replace foreach loops
        $template = preg_replace('/\{foreach\s+\$(\w+)\s+as\s+\$(\w+)\}/', '<?php foreach ($this->get(\'$1\') as $$2): ?>', $template);
        $template = str_replace('/{foreach}', '<?php endforeach; ?>', $template);
        
        // Replace if statements
        $template = preg_replace('/\{if\s+(\$?\w+(?:\.\w+)*)\}/', '<?php if ($this->get(\'$1\')): ?>', $template);
        $template = str_replace('/{if}', '<?php endif; ?>', $template);
        
        // Replace includes
        $template = preg_replace('/\{include\s+[\'"]([^\'"]+)[\'"]\}/', '<?php echo $this->render(\'$1\'); ?>', $template);
        
        return $template;
    }
    
    /**
     * Get variable value
     */
    public function get(string $key, $default = null) {
        $keys = explode('.', $key);
        $value = $this->variables;
        
        foreach ($keys as $k) {
            if (is_array($value) && isset($value[$k])) {
                $value = $value[$k];
            } else {
                return $default;
            }
        }
        
        return $value;
    }
    
    /**
     * Execute compiled template
     */
    private function executeCompiled(string $cacheFile): string {
        ob_start();
        
        extract($this->variables);
        include $cacheFile;
        
        return ob_get_clean();
    }
    
    /**
     * Clear template cache
     */
    public function clearCache(): void {
        $files = glob($this->cacheDir . '/*.php');
        
        foreach ($files as $file) {
            unlink($file);
        }
    }
}
```

## Asset Manager

```php
<?php
/**
 * Asset Manager
 * Manages theme CSS/JS assets with versioning
 */

namespace ClientTheme;

class AssetManager {
    
    private string $themeName;
    private string $assetsDir;
    private array $manifest = [];
    
    public function __construct(string $themeName) {
        $this->themeName = $themeName;
        $this->assetsDir = ROOTDIR . '/assets/' . $themeName;
        
        $this->loadManifest();
    }
    
    /**
     * Load asset manifest
     */
    private function loadManifest(): void {
        $manifestFile = $this->assetsDir . '/manifest.json';
        
        if (file_exists($manifestFile)) {
            $this->manifest = json_decode(file_get_contents($manifestFile), true) ?? [];
        }
    }
    
    /**
     * Get CSS file URL with version
     */
    public function css(string $file): string {
        $path = $this->assetsDir . '/css/' . $file;
        $version = $this->getVersion($path);
        
        return 'assets/' . $this->themeName . '/css/' . $file . '?v=' . $version;
    }
    
    /**
     * Get JS file URL with version
     */
    public function js(string $file): string {
        $path = $this->assetsDir . '/js/' . $file;
        $version = $this->getVersion($path);
        
        return 'assets/' . $this->themeName . '/js/' . $file . '?v=' . $version;
    }
    
    /**
     * Get file version (modification time)
     */
    private function getVersion(string $path): string {
        if (file_exists($path)) {
            return filemtime($path);
        }
        
        return '1.0';
    }
    
    /**
     * Enqueue CSS
     */
    public static function enqueueCss(string $themeName, string $file): void {
        $instance = new self($themeName);
        
        return '<link rel="stylesheet" href="' . $instance->css($file) . '">';
    }
    
    /**
     * Enqueue JS
     */
    public static function enqueueJs(string $themeName, string $file): void {
        $instance = new self($themeName);
        
        return '<script src="' . $instance->js($file) . '"></script>';
    }
    
    /**
     * Get all CSS files
     */
    public function getAllCss(): array {
        $css = [];
        $dir = $this->assetsDir . '/css';
        
        if (is_dir($dir)) {
            $files = glob($dir . '/*.css');
            
            foreach ($files as $file) {
                $css[] = basename($file);
            }
        }
        
        return $css;
    }
    
    /**
     * Get all JS files
     */
    public function getAllJs(): array {
        $js = [];
        $dir = $this->assetsDir . '/js';
        
        if (is_dir($dir)) {
            $files = glob($dir . '/*.js');
            
            foreach ($files as $file) {
                $js[] = basename($file);
            }
        }
        
        return $js;
    }
    
    /**
     * Minify CSS
     */
    public static function minifyCss(string $css): string {
        // Remove comments
        $css = preg_replace('/\/\*[\s\S]*?\*\//', '', $css);
        
        // Remove whitespace
        $css = preg_replace('/\s+/', ' ', $css);
        $css = preg_replace('/\s*([{}:;,])\s*/', '$1', $css);
        
        // Remove trailing semicolons before closing braces
        $css = str_replace(';}', '}', $css);
        
        return trim($css);
    }
    
    /**
     * Minify JS
     */
    public static function minifyJs(string $js): string {
        // In production, use a proper JS minifier library
        // This is a simplified example
        return $js;
    }
}
```

## Theme Manifest (config.json)

```json
{
    "name": "{Theme Name}",
    "version": "1.0.0",
    "author": "{Author}",
    "description": "Custom client area theme",
    "min_whmcs_version": "7.0",
    "assets": {
        "css": [
            "custom.css",
            "components.css",
            "pages.css"
        ],
        "js": [
            "custom.js",
            "animations.js"
        ],
        "images": [
            "logo.svg",
            "favicon.ico"
        ]
    },
    "settings": {
        "colors": {
            "primary": "#007bff",
            "secondary": "#6c757d",
            "accent": "#28a745"
        },
        "typography": {
            "font_family": "Poppins",
            "font_size_base": "14px"
        },
        "layout": {
            "container_max_width": "1200px",
            "sidebar_width": "250px"
        }
    }
}
```

## Main Layout Template

```smarty
<!DOCTYPE html>
<html lang="{$lang}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    
    <title>{$pagetitle} - {$companyname}</title>
    
    <!-- Favicon -->
    {if $theme_favicon}
        <link rel="icon" type="image/x-icon" href="{$theme_favicon}">
    {/if}
    
    <!-- Theme CSS Variables -->
    <style>
        :root {
            --primary-color: {$theme_primary_color};
            --secondary-color: {$theme_secondary_color};
            --accent-color: {$theme_accent_color};
            --font-family: {$theme_font_family};
            --font-size-base: {$theme_font_size};
            --border-radius: {$theme_border_radius};
            --box-shadow: {$theme_box_shadow};
        }
    </style>
    
    <!-- Theme Styles -->
    <link rel="stylesheet" href="{$base_url}assets/{$theme_name}/css/custom.css">
    
    {* Include page-specific head content *}
    {$headoutput}
</head>
<body class="client-area theme-{$theme_name}">
    
    <!-- Navigation Header -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand" href="{$WEB_ROOT}/">
                {if $theme_logo}
                    <img src="{$theme_logo}" alt="{$companyname}" height="40">
                {else}
                    {$companyname}
                {/if}
            </a>
            
            <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav ml-auto">
                    <li class="nav-item {if $current_page eq 'home'}active{/if}">
                        <a class="nav-link" href="clientarea.php">{lang key='home'}</a>
                    </li>
                    <li class="nav-item {if $current_page eq 'services'}active{/if}">
                        <a class="nav-link" href="clientarea.php?action=services">{lang key='services'}</a>
                    </li>
                    <li class="nav-item {if $current_page eq 'domains'}active{/if}">
                        <a class="nav-link" href="clientarea.php?action=domains">{lang key='domains'}</a>
                    </li>
                    <li class="nav-item {if $current_page eq 'billing'}active{/if}">
                        <a class="nav-link" href="clientarea.php?action=billing">{lang key='billing'}</a>
                    </li>
                    <li class="nav-item {if $current_page eq 'support'}active{/if}">
                        <a class="nav-link" href="clientarea.php?action=support">{lang key='support'}</a>
                    </li>
                    
                    {* User menu *}
                    <li class="nav-item dropdown">
                        <a class="nav-link dropdown-toggle" href="#" data-toggle="dropdown">
                            <i class="fa fa-user"></i> {$client.firstname} {$client.lastname}
                        </a>
                        <div class="dropdown-menu">
                            <a class="dropdown-item" href="clientarea.php?action=account">
                                <i class="fa fa-cog"></i> {lang key='accountdetails'}
                            </a>
                            <a class="dropdown-item" href="clientarea.php?action=security">
                                <i class="fa fa-shield-alt"></i> {lang key='security'}
                            </a>
                            <div class="dropdown-divider"></div>
                            <a class="dropdown-item" href="logout.php">
                                <i class="fa fa-sign-out-alt"></i> {lang key='logout'}
                            </a>
                        </div>
                    </li>
                </ul>
            </div>
        </div>
    </nav>
    
    <!-- Main Content -->
    <main class="main-content">
        <div class="container">
            {* Breadcrumbs *}
            {if $breadcrumb}
                <nav aria-label="breadcrumb" class="breadcrumb-nav">
                    <ol class="breadcrumb">
                        {foreach $breadcrumb as $crumb}
                            <li class="breadcrumb-item {if $crumb.active}active{/if}">
                                {if $crumb.active}
                                    {$crumb.label}
                                {else}
                                    <a href="{$crumb.url}">{$crumb.label}</a>
                                {/if}
                            </li>
                        {/foreach}
                    </ol>
                </nav>
            {/if}
            
            {* Page content *}
            {$content}
        </div>
    </main>
    
    <!-- Footer -->
    <footer class="footer">
        <div class="container">
            <div class="row">
                <div class="col-md-6">
                    <p>&copy; {date('Y')} {$companyname}. All rights reserved.</p>
                </div>
                <div class="col-md-6 text-md-right">
                    <p>
                        <a href="privacy.php">{lang key='privacy'}</a> |
                        <a href="terms.php">{lang key='terms'}</a> |
                        <a href="contact.php">{lang key='contact'}</a>
                    </p>
                </div>
            </div>
        </div>
    </footer>
    
    <!-- Theme Scripts -->
    <script src="{$base_url}assets/{$theme_name}/js/custom.js"></script>
    
    {* Include page-specific footer content *}
    {$footeroutput}
</body>
</html>
```

## Custom CSS Template

```css
/* {Theme Name} Custom Styles */

/* Typography */
:root {
    --primary-color: #007bff;
    --secondary-color: #6c757d;
    --accent-color: #28a745;
    --font-family: 'Poppins', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    --font-size-base: 14px;
}

body {
    font-family: var(--font-family);
    font-size: var(--font-size-base);
    background-color: #f8f9fa;
}

/* Navbar customization */
.navbar {
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.navbar-brand img {
    max-height: 40px;
}

/* Cards */
.card {
    border: none;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
    transition: transform 0.2s, box-shadow 0.2s;
}

.card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.12);
}

/* Buttons */
.btn {
    border-radius: 6px;
    font-weight: 500;
    transition: all 0.2s;
}

.btn-primary {
    background-color: var(--primary-color);
    border-color: var(--primary-color);
}

.btn-primary:hover {
    background-color: #0056b3;
    border-color: #0056b3;
}

/* Forms */
.form-control {
    border-radius: 6px;
    border: 1px solid #ddd;
    padding: 10px 15px;
}

.form-control:focus {
    border-color: var(--primary-color);
    box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.15);
}

/* Tables */
.table {
    border-radius: 8px;
    overflow: hidden;
}

.table thead th {
    background-color: var(--primary-color);
    color: white;
    border: none;
    font-weight: 600;
}

/* Alerts */
.alert {
    border-radius: 8px;
    border: none;
}

/* Footer */
.footer {
    background-color: #343a40;
    color: white;
    padding: 30px 0;
    margin-top: 60px;
}

.footer a {
    color: rgba(255, 255, 255, 0.8);
}

.footer a:hover {
    color: white;
}

/* Animations */
@keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
}

.fade-in {
    animation: fadeIn 0.3s ease-in;
}

/* Responsive */
@media (max-width: 768px) {
    .navbar-nav {
        padding-top: 10px;
    }
    
    .card {
        margin-bottom: 15px;
    }
}
```

## Admin Configuration Template

```smarty
<div class="client-theme-config">
    <h2>Client Area Theme Configuration</h2>
    
    <div class="alert alert-info">
        <i class="fa fa-palette"></i>
        Customize the appearance of your client area using the options below.
    </div>
    
    <form method="post" action="{$smarty.server.PHP_SELF}">
        <input type="hidden" name="module" value="themes">
        <input type="hidden" name="action" value="save">
        <input type="hidden" name="csrf_token" value="{$csrf_token}">
        
        <!-- Colors -->
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Colors</h3>
            </div>
            <div class="panel-body">
                <div class="row">
                    <div class="col-md-4">
                        <div class="form-group">
                            <label>Primary Color</label>
                            <div class="input-group color-picker">
                                <input type="color" name="primary_color" class="form-control" 
                                       value="{$config.primary_color|default:'#007bff'}">
                                <span class="input-group-addon">{$config.primary_color}</span>
                            </div>
                        </div>
                    </div>
                    <div class="col-md-4">
                        <div class="form-group">
                            <label>Secondary Color</label>
                            <input type="color" name="secondary_color" class="form-control"
                                   value="{$config.secondary_color|default:'#6c757d'}">
                        </div>
                    </div>
                    <div class="col-md-4">
                        <div class="form-group">
                            <label>Accent Color</label>
                            <input type="color" name="accent_color" class="form-control"
                                   value="{$config.accent_color|default:'#28a745'}">
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Typography -->
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Typography</h3>
            </div>
            <div class="panel-body">
                <div class="row">
                    <div class="col-md-6">
                        <div class="form-group">
                            <label>Font Family</label>
                            <select name="font_family" class="form-control">
                                <option value="Poppins" {if $config.font_family eq 'Poppins'}selected{/if}>Poppins</option>
                                <option value="Roboto" {if $config.font_family eq 'Roboto'}selected{/if}>Roboto</option>
                                <option value="Open Sans" {if $config.font_family eq 'Open Sans'}selected{/if}>Open Sans</option>
                                <option value="Lato" {if $config.font_family eq 'Lato'}selected{/if}>Lato</option>
                                <option value="Montserrat" {if $config.font_family eq 'Montserrat'}selected{/if}>Montserrat</option>
                            </select>
                        </div>
                    </div>
                    <div class="col-md-6">
                        <div class="form-group">
                            <label>Base Font Size</label>
                            <select name="font_size" class="form-control">
                                <option value="12px" {if $config.font_size eq '12px'}selected{/if}>12px</option>
                                <option value="14px" {if $config.font_size eq '14px'}selected{/if}>14px</option>
                                <option value="16px" {if $config.font_size eq '16px'}selected{/if}>16px</option>
                            </select>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Layout -->
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Layout</h3>
            </div>
            <div class="panel-body">
                <div class="row">
                    <div class="col-md-6">
                        <div class="form-group">
                            <label>Logo Upload</label>
                            <input type="file" name="logo" class="form-control">
                            {if $config.logo_url}
                                <img src="{$config.logo_url}" alt="Logo" style="max-height: 50px; margin-top: 10px;">
                            {/if}
                        </div>
                    </div>
                    <div class="col-md-6">
                        <div class="form-group">
                            <label>Favicon Upload</label>
                            <input type="file" name="favicon" class="form-control">
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Custom CSS -->
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Custom CSS</h3>
            </div>
            <div class="panel-body">
                <textarea name="custom_css" class="form-control" rows="10">{$config.custom_css}</textarea>
                <span class="help-block">Add custom CSS rules to further customize the theme.</span>
            </div>
        </div>
        
        <button type="submit" class="btn btn-primary">
            <i class="fa fa-save"></i> Save Theme Settings
        </button>
        
        <a href="?module={module}&action=preview" class="btn btn-info">
            <i class="fa fa-eye"></i> Preview Theme
        </a>
    </form>
</div>
```

## Checklist

```
Pre-Dev:
□ Define theme design system
□ Plan color palette
□ Choose typography
□ Design layout structure

Development:
□ Create theme configuration
□ Implement ThemeConfig class
□ Implement TemplateCompiler class
□ Implement AssetManager class
□ Create layout templates
□ Add CSS variables system
□ Build admin configuration UI
□ Add theme preview
□ Create custom CSS framework
□ Add responsive breakpoints

Testing:
□ Test all pages
□ Verify responsiveness
□ Test color customization
□ Test font changes
□ Verify CSS compilation
□ Test asset loading
□ Check browser compatibility
```
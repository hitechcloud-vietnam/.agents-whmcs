# WHMCS Template Compiler

## Overview

The Smarty compiler in WHMCS converts template files into PHP code. Understanding compiler hooks allows you to modify this process for custom functionality.

## Compiler Hooks

### Registering Compiler Hooks

```php
<?php
$smarty = \WHMCS\View\Template::getSmarty();

$smarty->registerPlugin('compiler', 'customtag', 'myCustomTag');
```

### Compiler Function

```php
<?php
function smarty_compiler_customtag($tagArg, $smarty)
{
    // Return PHP code to insert
    return '<?php echo "Custom content"; ?>';
}
```

## Custom Compiler Tags

### Simple Tag

```php
<?php
// Register
$smarty->registerPlugin('compiler', 'mydate', 'smarty_compiler_mydate');

// Usage in template: {mydate format="Y-m-d"}
function smarty_compiler_mydate($tagArg, $smarty)
{
    // Parse tag attributes
    preg_match('/format=["\']([^"\']+)["\']/', $tagArg, $matches);
    $format = $matches[1] ?? 'Y-m-d H:i:s';
    
    return '<?php echo date("' . $format . '"); ?>';
}
```

### Tag with Content

```php
<?php
// Usage: {mywrapper}content{/mywrapper}
$smarty->registerPlugin('compiler', 'mywrapper', 'smarty_compiler_mywrapper');

function smarty_compiler_mywrapper($tagArg, $tagContent, $smarty)
{
    return '<?php ob_start(); ?>' 
        . $tagContent 
        . '<?php echo "<div class=\\"wrapper\\">" . ob_get_clean() . "</div>"; ?>';
}
```

## Custom Block Tags

### Block Function

```php
<?php
$smarty->registerPlugin('function', 'infobox', 'smarty_function_infobox');

function smarty_function_infobox($params, $smarty)
{
    $title = $params['title'] ?? '';
    $type = $params['type'] ?? 'info';
    
    $html = '<div class="alert alert-' . $type . '">';
    $html .= '<strong>' . htmlspecialchars($title) . '</strong>';
    $html .= '</div>';
    
    return $html;
}
```

### Block with Content

```php
<?php
$smarty->registerPlugin('block', 'card', 'smarty_block_card');

function smarty_block_card($params, $content, $smarty, &$repeat)
{
    if (!$repeat) {
        $title = $params['title'] ?? '';
        return '<div class="card">
            <div class="card-header">' . $title . '</div>
            <div class="card-body">' . $content . '</div>
        </div>';
    }
}
```

### Template Usage

```smarty
{card title="My Card"}
    <p>Card content here</p>
{/card}
```

## Variable Modifiers

### Register Custom Modifier

```php
<?php
$smarty->registerPlugin('modifier', 'slugify', 'smarty_modifier_slugify');

function smarty_modifier_slugify($string)
{
    $string = strtolower($string);
    $string = preg_replace('/[^a-z0-9-]/', '-', $string);
    $string = preg_replace('/-+/', '-', $string);
    return trim($string, '-');
}
```

### Template Usage

```smarty
{$productName|slugify}  {* converts "My Product!" to "my-product" *}
```

## Security Considerations

### Compile Check

```php
<?php
$smarty->setCompileCheck(true);  // Default
$smarty->setCompileCheck(false); // Disable for production
```

### Trusted Methods

```php
<?php
// Allow specific static methods
$smarty->registerTrustedStaticMethods(['MyClass', 'methodName']);
```

## Compile Directory

### Setting Directory

```php
<?php
$smarty->setCompileDir('/path/to/compile/directory');

// Or via configuration
$compilerDir = dirname(__FILE__) . '/../templates_c';
if (!is_dir($compilerDir)) {
    mkdir($compilerDir, 0755);
}
$smarty->setCompileDir($compilerDir);
```

### Compile ID

```php
<?php
// Different compile directories for different templates
$smarty->setCompileId('template_name');
```

## Force Recompilation

### During Development

```php
<?php
// Always recompile (disable caching)
$smarty->setForceCompile(true);
```

### Per-Request

```php
<?php
if (isset($_GET['recompile'])) {
    $smarty->setForceCompile(true);
}
```

## Compiling Custom Tags

### Async Loading

```php
<?php
$smarty->registerPlugin('compiler', 'async', 'smarty_compiler_async');

function smarty_compiler_async($tagArg, $smarty)
{
    preg_match('/src=["\']([^"\']+)["\']/', $tagArg, $matches);
    $src = $matches[1];
    
    return '<div class="async-placeholder" data-src="' . $src . '"></div>';
}
```

### Conditional Compilation

```php
<?php
$smarty->registerPlugin('compiler', 'env', 'smarty_compiler_env');

function smarty_compiler_env($tagArg, $smarty)
{
    preg_match('/name=["\']([^"\']+)["\']/', $tagArg, $matches);
    $name = $matches[1];
    
    $value = getenv($name) ?: 'default';
    
    return '<?php echo "' . addslashes($value) . '"; ?>';
}
```

## Template Security

### Sandboxing

```php
<?php
// Allow only specific PHP functions
$smarty->setSecurity('WHMCSSecurityPolicy');

// Or create custom policy
$policy = new \Smarty\Security\Policy($smarty);
$policy->php_functions = ['date', 'count', 'in_array'];
$policy->modifiers = ['escape', 'count'];
$smarty->setSecurity($policy);
```

## Debugging Compilation

### View Compiled Output

```php
<?php
// Add ?debug to URL shows compiled template
$smarty->debugging = isset($_GET['debug']);
```

### Log Compilation

```php
<?php
$smarty->setCompileCheck(true);
$smarty->registerCallback('postCompile', function($compiled, $resource) {
    logActivity("Compiled: " . $resource);
});
```

## Performance Tips

1. **Disable `setForceCompile`** in production
2. **Use `setCompileCheck(SMARTY_COMPILECHECK_OFF)** once stable
3. **Separate compile directories** for different template sets
4. **Clear compile directory** after template updates

## See Also

- [Template Cache](../whmcs-template-cache.md)
- [Template Filters](../whmcs-template-filters.md)
- [Smarty Documentation](https://www.smarty.net/docs/en/)
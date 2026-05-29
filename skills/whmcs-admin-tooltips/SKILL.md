# WHMCS Admin Tooltips

## Overview
Guide for implementing tooltips in WHMCS admin area. Covers tooltip types, positioning, and dynamic tooltips.

## Tooltip Implementation

### Tooltip Helper

```php
<?php
// /includes/hooks/admin_tooltips.php

add_hook("AdminTooltipAssets", 1, function(array $params) {
    return [
        "js" => ["/assets/js/tooltip.js"]
    ];
});

function tooltip(
    string $content,
    string $title = "",
    string $placement = "top",
    array $options = []
): string {
    $id = "tooltip-" . uniqid();
    
    $html = '<span class="tooltip-wrapper" data-toggle="tooltip" ' .
            'data-placement="' . $placement . '" ' .
            'title="' . htmlspecialchars($content) . '"';
    
    if ($title) {
        $html .= ' data-original-title="' . htmlspecialchars($title) . '"';
    }
    
    if (isset($options["class"])) {
        $html .= ' class="' . $options["class"] . '"';
    }
    
    if (isset($options["html"]) && $options["html"]) {
        $html .= ' data-html="true"';
    }
    
    $html .= '>';
    $html .= $options["content"] ?? "";
    $html .= '</span>';
    
    return $html;
}

function tooltipIcon(
    string $content,
    string $icon = "question-circle",
    string $placement = "top"
): string {
    return tooltip($content, "", $placement, [
        "html" => true,
        "content" => '<i class="fa fa-' . $icon . '"></i>'
    ]);
}
```

### Dynamic Tooltip

```php
function dynamicTooltip(
    string $url,
    string $title = "",
    string $placement = "right"
): string {
    $id = "dynamic-tooltip-" . uniqid();
    
    return '<span class="dynamic-tooltip" ' .
           'data-url="' . $url . '" ' .
           'data-toggle="tooltip" ' .
           'data-placement="' . $placement . '" ' .
           'data-ajax="true">' .
           $title .
           '</span>';
}
```

## Tooltip Template

```smarty
<!-- /admin/templates/admin_tooltips.tpl -->

<!-- Simple Text Tooltip -->
<span class="has-tooltip" data-toggle="tooltip" 
      title="This is helpful information">
    <i class="fa fa-info-circle"></i>
</span>

<!-- Complex Tooltip with HTML -->
<span class="has-tooltip" data-toggle="tooltip" data-html="true"
      title="<strong>Bold</strong><br>
             <em>Italic</em><br>
             <a href='#'>Link</a>">
    <i class="fa fa-question-circle"></i>
</span>

<!-- Help Text Tooltip -->
<div class="form-group">
    <label for="field">Field Label</label>
    <input type="text" name="field" id="field" class="form-control">
    <span class="help-block">
        Enter the value here.
        <a href="#" class="has-tooltip" data-toggle="tooltip" 
           title="Get more help here">
            <i class="fa fa-question-circle"></i>
        </a>
    </span>
</div>

<!-- Status Tooltip -->
<span class="has-tooltip status-badge" 
      data-status="active"
      title="Active since: January 1, 2024">
    Active
</span>
```

## Tooltip JavaScript

```javascript
// /assets/js/tooltip.js

(function() {
    // Initialize tooltips
    function initTooltips() {
        $('[data-toggle="tooltip"]').tooltip({
            trigger: 'hover',
            container: 'body',
            boundary: 'window'
        });
        
        // Dynamic tooltip with AJAX content
        $('.dynamic-tooltip').each(function() {
            var $el = $(this);
            var url = $el.data('url');
            var cache = {};
            
            $el.tooltip({
                trigger: 'hover',
                html: true,
                title: 'Loading...',
                sanitize: false,
                container: 'body'
            });
            
            $el.on('inserted.bs.tooltip', function() {
                var tip = $(this).next('.tooltip');
                if (tip.find('.tooltip-inner').text() === 'Loading...') {
                    loadTooltipContent($el, url);
                }
            });
        });
    }
    
    function loadTooltipContent($el, url) {
        var tip = $el.next('.tooltip');
        
        $.get(url, function(response) {
            tip.find('.tooltip-inner').html(response);
        }).fail(function() {
            tip.find('.tooltip-inner').html('Failed to load');
        });
    }
    
    // Initialize on document ready
    $(document).ready(initTooltips);
    
    // Reinitialize after AJAX content loads
    $(document).ajaxComplete(function() {
        initTooltips();
    });
})();
```

## Best Practices

1. **Accessibility**: Add aria-describedby for screen readers
2. **Positioning**: Smart positioning based on viewport
3. **Delay**: Add delay to prevent flickering
4. **Sanitization**: Sanitize HTML content
5. **Timing**: Show after short delay
6. **Size Limits**: Limit tooltip content size
7. **Mobile**: Support touch on mobile
8. **Performance**: Lazy load heavy content

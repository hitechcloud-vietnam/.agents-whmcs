# WHMCS Admin Modals

## Overview
Guide for implementing modal dialogs in WHMCS admin area. Covers confirmation modals, form modals, and dynamic content modals.

## Modal Implementation

### Modal Templates

```php
<?php
// /includes/hooks/admin_modals.php

add_hook("AdminModalAssets", 1, function(array $params) {
    return [
        "css" => ["/assets/css/admin-modals.css"],
        "js" => ["/assets/js/admin-modals.js"]
    ];
});
```

### Confirmation Modal

```php
function showConfirmModal(
    string $title,
    string $message,
    string $confirmText = "Confirm",
    string $cancelText = "Cancel",
    string $confirmClass = "btn-primary"
): string {
    $id = "confirm-modal-" . uniqid();
    
    return '
    <div class="modal fade" id="' . $id . '">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <button type="button" class="close" data-dismiss="modal">
                        <span>&times;</span>
                    </button>
                    <h4 class="modal-title">' . htmlspecialchars($title) . '</h4>
                </div>
                <div class="modal-body">
                    <p>' . htmlspecialchars($message) . '</p>
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-default" data-dismiss="modal">
                        ' . htmlspecialchars($cancelText) . '
                    </button>
                    <button type="button" class="btn ' . $confirmClass . ' btn-confirm">
                        ' . htmlspecialchars($confirmText) . '
                    </button>
                </div>
            </div>
        </div>
    </div>';
}
```

### AJAX Modal

```php
function showAjaxModal(string $url, string $title, array $options = []): string
{
    $id = "ajax-modal-" . uniqid();
    $size = $options["size"] ?? "";
    
    $sizes = [
        "small" => "modal-sm",
        "large" => "modal-lg",
        "xlarge" => "modal-xl"
    ];
    
    $sizeClass = $sizes[$size] ?? "";
    
    return '
    <div class="modal fade" id="' . $id . '" tabindex="-1">
        <div class="modal-dialog ' . $sizeClass . '">
            <div class="modal-content">
                <div class="modal-header">
                    <button type="button" class="close" data-dismiss="modal">
                        <span>&times;</span>
                    </button>
                    <h4 class="modal-title">' . htmlspecialchars($title) . '</h4>
                </div>
                <div class="modal-body">
                    <div class="modal-loading">
                        <i class="fa fa-spinner fa-spin"></i>
                    </div>
                </div>
                <div class="modal-footer" style="display:none;">
                </div>
            </div>
        </div>
    </div>
    <script>
    $("#' . $id . '").on("show.bs.modal", function() {
        var $modal = $(this);
        $.get("' . $url . '", function(html) {
            $modal.find(".modal-body").html(html);
        });
    });
    </script>';
}
```

### Form Modal

```php
function showFormModal(
    string $title,
    string $action,
    array $fields,
    array $options = []
): string {
    $id = "form-modal-" . uniqid();
    $method = $options["method"] ?? "post";
    
    $body = "<form method=\"{$method}\" action=\"{$action}\" class=\"modal-form\">";
    
    foreach ($fields as $field) {
        $body .= '<div class="form-group">';
        $body .= '<label>' . ($field["label"] ?? $field["name"]) . '</label>';
        
        switch ($field["type"]) {
            case "text":
            case "email":
            case "password":
            case "number":
                $body .= '<input type="' . $field["type"] . '" ' .
                        'name="' . $field["name"] . '" ' .
                        'class="form-control"';
                if (isset($field["required"])) $body .= ' required';
                if (isset($field["value"])) $body .= ' value="' . $field["value"] . '"';
                $body .= '>';
                break;
                
            case "textarea":
                $body .= '<textarea name="' . $field["name"] . '" ' .
                        'class="form-control" rows="4"';
                if (isset($field["required"])) $body .= ' required';
                $body .= '>';
                if (isset($field["value"])) $body .= $field["value"];
                $body .= '</textarea>';
                break;
                
            case "select":
                $body .= '<select name="' . $field["name"] . '" class="form-control"';
                if (isset($field["required"])) $body .= ' required';
                $body .= '>';
                $body .= '<option value="">Select...</option>';
                foreach ($field["options"] as $value => $label) {
                    $body .= '<option value="' . $value . '">' . $label . '</option>';
                }
                $body .= '</select>';
                break;
        }
        
        $body .= '</div>';
    }
    
    $body .= '</form>';
    
    return '
    <div class="modal fade" id="' . $id . '">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <button type="button" class="close" data-dismiss="modal">
                        <span>&times;</span>
                    </button>
                    <h4 class="modal-title">' . htmlspecialchars($title) . '</h4>
                </div>
                <div class="modal-body">' . $body . '</div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-default" data-dismiss="modal">
                        Cancel
                    </button>
                    <button type="button" class="btn btn-primary btn-submit">
                        Submit
                    </button>
                </div>
            </div>
        </div>
    </div>';
}
```

## Modal Templates

```smarty
<!-- /admin/templates/modal_base.tpl -->
<div class="modal fade" id="{$modal.id}" tabindex="-1" role="dialog">
    <div class="modal-dialog {if $modal.size}modal-{$modal.size}{/if}" role="document">
        <div class="modal-content">
            <div class="modal-header">
                <button type="button" class="close" data-dismiss="modal" 
                        aria-label="Close">
                    <span aria-hidden="true">&times;</span>
                </button>
                <h4 class="modal-title">
                    {if $modal.icon}
                        <i class="fa fa-{$modal.icon}"></i>
                    {/if}
                    {$modal.title}
                </h4>
            </div>
            
            <div class="modal-body">
                {$modal.body}
            </div>
            
            {if $modal.buttons}
                <div class="modal-footer">
                    {foreach $modal.buttons as $button}
                        <button type="button" 
                                class="btn {$button.class}"
                                {if $button.id}id="{$button.id}"{/if}
                                {if $button.dismiss}data-dismiss="modal"{/if}
                                {if $button.onclick}onclick="{$button.onclick}"{/if}>
                            {if $button.icon}
                                <i class="fa fa-{$button.icon}"></i>
                            {/if}
                            {$button.label}
                        </button>
                    {/foreach}
                </div>
            {/if}
        </div>
    </div>
</div>
```

### Modal JavaScript

```javascript
// /assets/js/admin-modals.js

(function() {
    // Global modal handler
    window.showModal = function(options) {
        var defaults = {
            title: '',
            content: '',
            buttons: [
                {
                    label: 'Close',
                    class: 'btn-default',
                    dismiss: true
                }
            ],
            size: '', // '', 'sm', 'lg', 'xl'
            onShow: null,
            onHide: null
        };
        
        var settings = Object.assign({}, defaults, options);
        var modalId = 'modal-' + Date.now();
        
        var html = '<div class="modal fade" id="' + modalId + '">' +
            '<div class="modal-dialog' + (settings.size ? ' modal-' + settings.size : '') + '">' +
            '<div class="modal-content">' +
            '<div class="modal-header">' +
            '<button type="button" class="close" data-dismiss="modal"><span>&times;</span></button>' +
            '<h4 class="modal-title">' + settings.title + '</h4>' +
            '</div>' +
            '<div class="modal-body">' + settings.content + '</div>';
        
        if (settings.buttons.length > 0) {
            html += '<div class="modal-footer">';
            settings.buttons.forEach(function(btn) {
                html += '<button type="button" class="btn ' + btn.class + '"';
                if (btn.dismiss) html += ' data-dismiss="modal"';
                if (btn.id) html += ' id="' + btn.id + '"';
                html += '>' + btn.label + '</button>';
            });
            html += '</div>';
        }
        
        html += '</div></div></div>';
        
        var $modal = $(html);
        $('body').append($modal);
        
        if (settings.onShow) {
            $modal.on('show.bs.modal', settings.onShow);
        }
        if (settings.onHide) {
            $modal.on('hide.bs.modal', settings.onHide);
        }
        
        $modal.on('hidden.bs.modal', function() {
            $(this).remove();
        });
        
        $modal.modal('show');
        return modalId;
    };
    
    // Confirmation modal helper
    window.confirmModal = function(options) {
        return showModal({
            title: options.title || 'Confirm',
            content: '<p>' + (options.message || 'Are you sure?') + '</p>',
            buttons: [
                {
                    label: options.cancelText || 'Cancel',
                    class: 'btn-default'
                },
                {
                    label: options.confirmText || 'Confirm',
                    class: options.confirmClass || 'btn-primary',
                    id: 'confirm-btn'
                }
            ],
            onShow: function() {
                $('#confirm-btn').on('click', function() {
                    if (options.onConfirm) {
                        options.onConfirm();
                    }
                    $('#' + $(this).closest('.modal').attr('id')).modal('hide');
                });
            }
        });
    };
})();
```

## Best Practices

1. **Accessibility**: Support keyboard navigation (Escape to close)
2. **Focus Management**: Trap focus within modal
3. **Animation**: Smooth open/close animations
4. **Size Options**: Support multiple modal sizes
5. **Loading State**: Show loading for async content
6. **Confirmation**: Confirm destructive actions
7. **Cleanup**: Remove modal from DOM after close
8. **Scrollable**: Support scrollable content

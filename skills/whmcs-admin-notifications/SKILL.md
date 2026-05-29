# WHMCS Admin Notifications

## Overview
Guide for implementing admin notifications in WHMCS. Covers toast notifications, alert banners, and notification centers.

## Notification System

### Toast Notifications

```php
<?php
// /includes/hooks/admin_notifications.php

function addNotification(
    string $type,
    string $message,
    array $options = []
): string {
    $id = "notification-" . uniqid();
    
    $icons = [
        "success" => "check-circle",
        "error" => "exclamation-circle",
        "warning" => "exclamation-triangle",
        "info" => "info-circle"
    ];
    
    $notification = [
        "id" => $id,
        "type" => $type,
        "message" => $message,
        "icon" => $icons[$type] ?? "info-circle",
        "title" => $options["title"] ?? "",
        "dismissible" => $options["dismissible"] ?? true,
        "duration" => $options["duration"] ?? 5000,
        "url" => $options["url"] ?? "",
        "action_text" => $options["action_text"] ?? ""
    ];
    
    if (!isset($_SESSION["admin_notifications"])) {
        $_SESSION["admin_notifications"] = [];
    }
    
    $_SESSION["admin_notifications"][] = $notification;
    
    return $id;
}

function addSuccess(string $message, array $options = []): string {
    return addNotification("success", $message, $options);
}

function addError(string $message, array $options = []): string {
    return addNotification("error", $message, array_merge($options, ["duration" => 0]));
}

function addWarning(string $message, array $options = []): string {
    return addNotification("warning", $message, $options);
}

function addInfo(string $message, array $options = []): string {
    return addNotification("info", $message, $options);
}
```

### Alert Banners

```php
function showAlertBanner(
    string $type,
    string $message,
    array $options = []
): string {
    $classes = ["alert", "alert-" . $type];
    
    if (isset($options["dismissible"]) && $options["dismissible"]) {
        $classes[] = "alert-dismissible";
    }
    
    if (isset($options["class"])) {
        $classes[] = $options["class"];
    }
    
    $html = '<div class="' . implode(" ", $classes) . '" role="alert">';
    
    if (isset($options["dismissible"]) && $options["dismissible"]) {
        $html .= '<button type="button" class="close" data-dismiss="alert" ' .
                'aria-label="Close">' .
                '<span aria-hidden="true">&times;</span></button>';
    }
    
    if (isset($options["title"])) {
        $html .= '<strong>' . htmlspecialchars($options["title"]) . '</strong> ';
    }
    
    $html .= htmlspecialchars($message);
    
    if (isset($options["actions"])) {
        $html .= '<div class="alert-actions">';
        foreach ($options["actions"] as $action) {
            $html .= '<a href="' . $action["url"] . '" ' .
                    'class="btn btn-' . ($action["type"] ?? "default") . ' btn-sm"' .
                    (isset($action["target"]) ? ' target="' . $action["target"] . '"' : '') .
                    '>' . $action["label"] . '</a>';
        }
        $html .= '</div>';
    }
    
    $html .= '</div>';
    
    return $html;
}
```

## Notification Templates

```smarty
<!-- /admin/templates/toast_notifications.tpl -->
<div class="toast-container" id="toast-container">
    {foreach $_SESSION["admin_notifications"] as $notification}
        <div class="toast toast-{$notification.type}" 
             id="{$notification.id}"
             role="alert"
             data-duration="{$notification.duration}">
            <div class="toast-icon">
                <i class="fa fa-{$notification.icon}"></i>
            </div>
            <div class="toast-content">
                {if $notification.title}
                    <div class="toast-title">{$notification.title}</div>
                {/if}
                <div class="toast-message">{$notification.message}</div>
                {if $notification.url && $notification.action_text}
                    <a href="{$notification.url}" class="toast-action">
                        {$notification.action_text}
                    </a>
                {/if}
            </div>
            {if $notification.dismissible}
                <button type="button" class="toast-close" data-dismiss="toast">
                    <i class="fa fa-times"></i>
                </button>
            {/if}
        </div>
    {/foreach}
</div>

<script>
(function() {
    function showToasts() {
        $('.toast[data-duration]').each(function() {
            var $toast = $(this);
            var duration = parseInt($toast.data('duration'));
            
            setTimeout(function() {
                $toast.addClass('show');
                
                if (duration > 0) {
                    setTimeout(function() {
                        $toast.removeClass('show');
                        setTimeout(function() {
                            $toast.remove();
                        }, 300);
                    }, duration);
                }
            }, 100);
        });
    }
    
    $(document).ready(showToasts);
    
    $(document).on('click', '.toast-close', function() {
        var $toast = $(this).closest('.toast');
        $toast.removeClass('show');
        setTimeout(function() {
            $toast.remove();
        }, 300);
    });
})();
</script>

<style>
.toast-container {
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 9999;
    max-width: 400px;
}
.toast {
    display: flex;
    align-items: flex-start;
    padding: 15px;
    margin-bottom: 10px;
    background: #fff;
    border-radius: 4px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    opacity: 0;
    transform: translateX(100%);
    transition: all 0.3s ease;
}
.toast.show {
    opacity: 1;
    transform: translateX(0);
}
.toast-success { border-left: 4px solid #28a745; }
.toast-error { border-left: 4px solid #dc3545; }
.toast-warning { border-left: 4px solid #ffc107; }
.toast-info { border-left: 4px solid #17a2b8; }
.toast-icon {
    margin-right: 12px;
    font-size: 20px;
}
.toast-success .toast-icon { color: #28a745; }
.toast-error .toast-icon { color: #dc3545; }
.toast-warning .toast-icon { color: #ffc107; }
.toast-info .toast-icon { color: #17a2b8; }
.toast-content { flex: 1; }
.toast-title { font-weight: bold; }
.toast-message { margin-top: 4px; }
.toast-action {
    display: inline-block;
    margin-top: 8px;
    font-weight: 500;
}
.toast-close {
    background: none;
    border: none;
    padding: 0;
    cursor: pointer;
    opacity: 0.5;
}
.toast-close:hover { opacity: 1; }
</style>
```

## Notification Center

```php
function getNotificationCenter(int $adminId, int $limit = 20): array
{
    return Capsule::table("mod_admin_notifications")
        ->where("admin_id", $adminId)
        ->where("read", 0)
        ->orderBy("created_at", "desc")
        ->limit($limit)
        ->get();
}

function markNotificationRead(int $notificationId): void
{
    Capsule::table("mod_admin_notifications")
        ->where("id", $notificationId)
        ->update(["read" => 1, "read_at" => date("Y-m-d H:i:s")]);
}

function createAdminNotification(
    int $adminId,
    string $type,
    string $title,
    string $message,
    array $options = []
): int {
    return Capsule::table("mod_admin_notifications")->insertGetId([
        "admin_id" => $adminId,
        "type" => $type,
        "title" => $title,
        "message" => $message,
        "url" => $options["url"] ?? "",
        "created_at" => date("Y-m-d H:i:s")
    ]);
}
```

## Best Practices

1. **Non-Intrusive**: Use toasts for non-critical info
2. **Auto-Dismiss**: Auto-dismiss after timeout
3. **Dismissible**: Allow manual dismissal
4. **Queue**: Queue multiple notifications
5. **Sound**: Optional sound for important alerts
6. **Positioning**: Consistent positioning
7. **Mobile**: Responsive on mobile
8. **Accessibility**: Support keyboard navigation

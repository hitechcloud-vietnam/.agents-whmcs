# WHMCS Live Chat

## Overview
Master skill for live chat integration in WHMCS. Covers chat widget implementation, chat routing, and operator management.

## Live Chat Integration

```php
<?php
// /includes/hooks/live_chat_hooks.php

add_hook("ClientAreaFooterOutput", 1, function(array $params) {
    $userId = $params["userid"] ?? 0;
    $userName = $params["clientname"] ?? "Guest";
    
    $chatCode = '<script>
        window.whmcsChatConfig = {
            appId: "' . get_config("livechat_app_id") . '",
            userId: "' . $userId . '",
            userName: "' . htmlspecialchars($userName) . '",
            userEmail: "' . ($params["email"] ?? "") . '"
        };
    </script>';
    $chatCode .= '<script src="https://cdn.livechat.com/livechat.js"></script>';
    
    return $chatCode;
});

add_hook("LiveChatStarted", 1, function(array $params) {
    $conversationId = $params["conversation_id"];
    $clientId = $params["userid"];
    
    logLiveChatStart($conversationId, $clientId);
    
    return true;
});

add_hook("LiveChatEnded", 1, function(array $params) {
    $conversationId = $params["conversation_id"];
    
    saveChatTranscript($conversationId);
    
    return true;
});
```

## Live Chat Widget

```html
<!-- Add to footer template -->
<div id="whmcs-live-chat"></div>

<script>
document.addEventListener("DOMContentLoaded", function() {
    if (window.whmcsChatConfig) {
        LiveChat.call("init", {
            app_id: window.whmcsChatConfig.appId,
            visitor: {
                id: window.whmcsChatConfig.userId,
                name: window.whmcsChatConfig.userName,
                email: window.whmcsChatConfig.userEmail
            }
        });
        
        // Show chat widget
        LiveChat.call("show");
        
        // Track events
        LiveChat.on("connected", function() {
            console.log("Live chat connected");
        });
        
        LiveChat.on("message", function(data) {
            console.log("Chat message:", data);
        });
    }
});
</script>
```

## Best Practices

1. **Availability**: Show online/offline status
2. **Proactive**: Start proactive chats
3. **Integration**: Integrate with tickets
4. **Transcripts**: Save chat transcripts
5. **Routing**: Route chats to right department
6. **Mobile**: Optimize for mobile
7. **Performance**: Keep widget lightweight
8. **Analytics**: Track chat metrics

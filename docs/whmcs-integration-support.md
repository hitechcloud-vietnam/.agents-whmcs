# WHMCS Support Integration

Complete guide for integrating WHMCS with support systems.

## Overview

Connect WHMCS with external support platforms for unified ticket management.

## Zendesk Integration

### Zendesk API Client

```php
<?php
/**
 * Zendesk support integration
 */
class ZendeskSupportClient
{
    private string $subdomain;
    private string $email;
    private string $apiToken;
    
    public function __construct(array $config)
    {
        $this->subdomain = $config['subdomain'];
        $this->email = $config['email'];
        $this->apiToken = $config['api_token'];
    }
    
    /**
     * Get base URL
     */
    private function getBaseUrl(): string
    {
        return "https://{$this->subdomain}.zendesk.com/api/v2";
    }
    
    /**
     * Make API request
     */
    public function request(string $method, string $endpoint, array $data = []): array
    {
        $ch = curl_init($this->getBaseUrl() . $endpoint);
        
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => "{$this->email}/token:{$this->apiToken}",
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method === 'PUT') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PUT');
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response ?? [];
    }
    
    /**
     * Create ticket
     */
    public function createTicket(array $ticket): array
    {
        return $this->request('POST', '/tickets.json', ['ticket' => $ticket]);
    }
    
    /**
     * Get ticket
     */
    public function getTicket(int $ticketId): array
    {
        return $this->request('GET', "/tickets/{$ticketId}.json");
    }
    
    /**
     * Add comment to ticket
     */
    public function addComment(int $ticketId, string $comment, bool $isPublic = true): array
    {
        return $this->request('PUT', "/tickets/{$ticketId}.json", [
            'ticket' => [
                'comment' => [
                    'body' => $comment,
                    'public' => $isPublic,
                ],
            ],
        ]);
    }
    
    /**
     * Search tickets
     */
    public function searchTickets(string $query): array
    {
        return $this->request('GET', '/search.json?query=' . urlencode($query));
    }
}
```

### Zendesk Sync Hook

```php
<?php
/**
 * Create Zendesk ticket when WHMCS ticket created
 */
add_hook('TicketOpen', 1, function($vars) {
    $zendesk = new ZendeskSupportClient([
        'subdomain' => ZENDESK_SUBDOMAIN,
        'email' => ZENDESK_EMAIL,
        'api_token' => ZENDESK_TOKEN,
    ]);
    
    // Get ticket details
    $ticket = Capsule::table('tbltickets')
        ->where('id', $vars['ticketid'])
        ->first();
    
    $client = Capsule::table('tblclients')
        ->where('id', $ticket->userid)
        ->first();
    
    // Create Zendesk ticket
    $zendeskTicket = [
        'subject' => "[#{$ticket->tid}] {$ticket->subject}",
        'comment' => [
            'body' => $ticket->message,
        ],
        'requester' => [
            'name' => $client ? "{$client->firstname} {$client->lastname}" : 'Unknown',
            'email' => $ticket->email,
        ],
        'priority' => mapPriority($ticket->urgency),
        'tags' => ['whmcs', 'source_api'],
        'external_id' => "whmcs_ticket_{$ticket->id}",
    ];
    
    $result = $zendesk->createTicket($zendeskTicket);
    
    if (isset($result['ticket']['id'])) {
        // Store mapping
        Capsule::table('mod_support_sync')->insert([
            'whmcs_ticket_id' => $ticket->id,
            'zendesk_ticket_id' => $result['ticket']['id'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        logActivity("Created Zendesk ticket #{$result['ticket']['id']} for WHMCS ticket #{$ticket->id}");
    }
});

/**
 * Map WHMCS priority to Zendesk
 */
function mapPriority(int $urgency): string
{
    return match(true) {
        $urgency >= 4 => 'high',
        $urgency >= 2 => 'normal',
        default => 'low',
    };
}
```

## Freshdesk Integration

```php
<?php
/**
 * Freshdesk support integration
 */
class FreshdeskSupportClient
{
    private string $domain;
    private string $apiKey;
    
    public function __construct(string $domain, string $apiKey)
    {
        $this->domain = $domain;
        $this->apiKey = $apiKey;
    }
    
    /**
     * Make API request
     */
    public function request(string $method, string $endpoint, array $data = []): array
    {
        $ch = curl_init("https://{$this->domain}.freshdesk.com/api/v2{$endpoint}");
        
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD => "{$this->apiKey}:X",
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method === 'PUT') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PUT');
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response ?? [];
    }
    
    /**
     * Create ticket
     */
    public function createTicket(array $ticket): array
    {
        return $this->request('POST', '/tickets', [
            'subject' => $ticket['subject'],
            'description' => $ticket['description'],
            'email' => $ticket['email'],
            'priority' => $ticket['priority'] ?? 1,
            'status' => $ticket['status'] ?? 2,
            'external_id' => $ticket['external_id'] ?? null,
        ]);
    }
    
    /**
     * Add reply
     */
    public function addReply(int $ticketId, string $body, bool $incoming = false): array
    {
        return $this->request('POST', "/tickets/{$ticketId}/conversations", [
            'body' => $body,
            'incoming' => $incoming,
        ]);
    }
}
```

## Two-Way Sync

```php
<?php
/**
 * Sync replies from support platform to WHMCS
 */
function syncSupportReplies(): void
{
    $zendesk = new ZendeskSupportClient([
        'subdomain' => ZENDESK_SUBDOMAIN,
        'email' => ZENDESK_EMAIL,
        'api_token' => ZENDESK_TOKEN,
    ]);
    
    // Get last sync time
    $lastSync = Capsule::table('mod_support_sync')
        ->max('last_reply_sync');
    
    // Find tickets updated since last sync
    $updatedTickets = Capsule::table('mod_support_sync')
        ->where('last_reply_sync', '<=', $lastSync ?? 0)
        ->get();
    
    foreach ($updatedTickets as $mapping) {
        $zendeskTicket = $zendesk->getTicket($mapping->zendesk_ticket_id);
        
        if (!isset($zendeskTicket['ticket'])) continue;
        
        $comments = $zendesk->request('GET', 
            "/tickets/{$mapping->zendesk_ticket_id}/comments.json");
        
        // Process new comments
        foreach ($comments['comments'] ?? [] as $comment) {
            $createdAt = strtotime($comment['created_at']);
            $lastSyncTime = strtotime($mapping->last_reply_sync ?? 0);
            
            if ($createdAt > $lastSyncTime && $comment['public']) {
                addWHMCSTicketReply(
                    $mapping->whmcs_ticket_id,
                    $comment['body'],
                    $comment['author_id'] !== $mapping->zendesk_requester_id
                );
            }
        }
        
        // Update last sync time
        Capsule::table('mod_support_sync')
            ->where('id', $mapping->id)
            ->update(['last_reply_sync' => date('Y-m-d H:i:s')]);
    }
}

/**
 * Add reply to WHMCS ticket
 */
function addWHMCSTicketReply(int $ticketId, string $message, bool $isAdmin): bool
{
    $adminId = $isAdmin ? getAdminId() : 0;
    
    $replyId = Capsule::table('tblticketreplies')->insertGetId([
        'tid' => $ticketId,
        'userid' => 0,
        'contactid' => 0,
        'adminid' => $adminId,
        'date' => date('Y-m-d H:i:s'),
        'message' => $message,
        'clientUnread' => $isAdmin ? 1 : 0,
    ]);
    
    // Notify client if admin reply
    if ($isAdmin) {
        sendTickerReplyNotification($ticketId, $message);
    }
    
    return $replyId > 0;
}
```

## Support Dashboard Widget

```php
<?php
/**
 * Add support stats to admin dashboard
 */
add_hook('AdminHomepage', 1, function($vars) {
    $zendesk = new ZendeskSupportClient([
        'subdomain' => ZENDESK_SUBDOMAIN,
        'email' => ZENDESK_EMAIL,
        'api_token' => ZENDESK_TOKEN,
    ]);
    
    // Get open tickets count
    $openTickets = $zendesk->searchTickets('type:ticket status:open OR status:pending');
    
    // Get WHMCS tickets
    $whmcsOpen = Capsule::table('tbltickets')
        ->whereIn('status', ['Open', 'Answered'])
        ->count();
    
    return [
        'name' => 'Support Overview',
        'template' => 'support-overview-widget',
        'vars' => [
            'zendesk_open' => $openTickets['count'] ?? 0,
            'whmcs_open' => $whmcsOpen,
        ],
    ];
});
```

## Best Practices

1. **Map ticket fields** - Ensure consistent data between systems
2. **Sync bidirectionally** - Keep both systems updated
3. **Handle attachments** - Sync files correctly
4. **Preserve history** - Don't lose conversation context
5. **Real-time sync** - Use webhooks for immediate updates
6. **Handle conflicts** - Decide which system wins

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-webhooks.md](whmcs-integration-webhooks.md)

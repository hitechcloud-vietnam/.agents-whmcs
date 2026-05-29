# WHMCS Domain Auction

## Overview

Domain auction in WHMCS allows buying and selling of premium domains through auction platforms integrated with registrars.

## Auction Configuration

### Enable Auctions

**Configuration > Domains > Auctions**

```php
// Auction settings
[
    'enable_auctions' => false,
    'provider' => 'sedo',
    'api_key' => 'xxx',
    'auto_list' => false
]
```

## Auction Types

### Auction Categories

| Type | Description |
|------|-------------|
| Premium | Valuable expired domains |
| Backorder | Pre-release auctions |
| Closeout | Liquidation auctions |

## Buying Domains

### Bid on Auction

```php
// Place bid
[
    'auction_id' => 'xxx',
    'domain' => 'premium.com',
    'bid_amount' => 500.00,
    'max_bid' => 750.00,
    'currency' => 'USD'
]
```

## Listing Domains

### Sell Domain at Auction

```php
// List domain for auction
[
    'domain_id' => 1,
    'auction_type' => 'standard',
    'starting_price' => 100.00,
    'reserve_price' => 500.00,
    'duration_days' => 7
]
```

## Auction Status

### Track Auction

```php
// Auction status
[
    'auction_id' => 'xxx',
    'domain' => 'premium.com',
    'current_bid' => 500.00,
    'bids' => 15,
    'ends' => '2024-05-22',
    'status' => 'active'
]
```

## API Functions

```php
// Search auctions
$result = localAPI('SearchDomainAuctions', [
    'tlds' => ['com', 'net'],
    'min_price' => 100,
    'max_price' => 1000
]);

// Place bid
$result = localAPI('PlaceAuctionBid', [
    'auction_id' => 'xxx',
    'amount' => 500
]);
```

## Best Practices

1. **Research domains**: Evaluate before bidding
2. **Set limits**: Define maximum bid amounts
3. **Monitor auctions**: Track active auctions
4. **Quick action**: Bid promptly on desired domains

## Related Documentation

- [Domain Backorder](./whmcs-domain-backorder.md)
- [Premium Domains](./whmcs-premium-domains.md)
- [Domain Catching](./whmcs-domain-catching.md)
# WHMCS Premium DNS Service Devkit

## Overview
A premium DNS service offering with 100% uptime SLA, anycast network, DDoS protection, and advanced traffic management features.

## Features
- Anycast DNS network worldwide
- 100% uptime SLA guarantee
- DDoS protection included
- Fast propagation (<30 seconds)
- Traffic load balancing
- Request throttling
- Real-time analytics
- API access with rate limiting
- Custom resolver IPs
- SNINg support
- DNS-over-HTTPS/TLS support
- Priority support access

## WHMCS Integration Points
- Custom product: premium_dns
- Module: addon/PremiumDns
- Hook: ServiceCreate
- SLA monitoring integration

## Database Schema
```sql
CREATE TABLE mod_premium_dns_zones (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    domain VARCHAR(255) NOT NULL,
    protection_level ENUM('basic', 'standard', 'advanced', 'enterprise') DEFAULT 'basic',
    sla_tier VARCHAR(50),
    anycast_nodes JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id)
);

CREATE TABLE mod_premium_dns_analytics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    zone_id INT NOT NULL,
    query_count BIGINT,
    blocked_threats INT,
    avg_latency_ms DECIMAL(10,2),
    uptime_percent DECIMAL(5,2),
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_zone_time (zone_id, recorded_at)
);
```

## API Endpoints
- POST /api/premium-dns/zones - Create DNS zone
- GET /api/premium-dns/zones/:id - Get zone details
- POST /api/premium-dns/zones/:id/records - Add records
- GET /api/premium-dns/zones/:id/analytics - Get analytics
- PUT /api/premium-dns/zones/:id/protection - Update protection

## Testing Checklist
- [ ] DNS zone creation
- [ ] Anycast routing
- [ ] DDoS protection activation
- [ ] Analytics recording
- [ ] SLA monitoring
- [ ] API rate limiting
- [ ] Uptime tracking

## Dependencies
- WHMCS 8.x+
- PHP 8.1+

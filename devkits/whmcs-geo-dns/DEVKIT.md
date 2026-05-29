# WHMCS Geographic DNS Routing Devkit

## Overview
A geographic DNS routing system that directs users to the closest server based on their location, with failover capabilities and real-time traffic optimization.

## Features
- Geographic-based DNS routing
- Latency-based routing
- IP-based continent/country routing
- Multiple endpoint support
- Health check failover
- Traffic load distribution
- Real-time routing analytics
- Custom routing rules
- Endpoint weight configuration
- Mobile carrier routing
- ISP-based routing
- Time-based routing rules

## WHMCS Integration Points
- Custom product: geo_dns
- Module: addon/GeoDnsRouter
- Hook: ServiceCreate
- Analytics integration

## Database Schema
```sql
CREATE TABLE mod_geo_dns_zones (
    id INT AUTO_INCREMENT PRIMARY KEY,
    domain VARCHAR(255) NOT NULL,
    service_id INT NOT NULL,
    routing_mode ENUM('geo', 'latency', 'weighted', 'failover') DEFAULT 'geo',
    default_endpoint VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_domain (domain)
);

CREATE TABLE mod_geo_dns_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    zone_id INT NOT NULL,
    rule_type ENUM('country', 'continent', 'asn', 'isp', 'mobile', 'custom_ip') NOT NULL,
    match_value VARCHAR(100) NOT NULL,
    target_endpoint VARCHAR(255),
    weight INT DEFAULT 100,
    is_active TINYINT(1) DEFAULT 1,
    priority INT DEFAULT 0
);

CREATE TABLE mod_geo_dns_endpoints (
    id INT AUTO_INCREMENT PRIMARY KEY,
    zone_id INT NOT NULL,
    endpoint_url VARCHAR(255) NOT NULL,
    region VARCHAR(100),
    weight INT DEFAULT 100,
    health_check_url VARCHAR(500),
    health_status ENUM('healthy', 'degraded', 'down') DEFAULT 'healthy',
    last_check_at TIMESTAMP
);
```

## API Endpoints
- POST /api/geo-dns/zones - Create zone
- GET /api/geo-dns/zones/:id - Get zone
- POST /api/geo-dns/zones/:id/rules - Add rules
- PUT /api/geo-dns/rules/:id - Update rule
- GET /api/geo-dns/analytics/:zone - Get analytics

## Testing Checklist
- [ ] Geographic routing accuracy
- [ ] Latency-based routing
- [ ] Endpoint failover
- [ ] Rule priority evaluation
- [ ] Mobile carrier detection
- [ ] Weight-based distribution
- [ ] Health check execution

## Dependencies
- WHMCS 8.x+
- PHP 8.1+

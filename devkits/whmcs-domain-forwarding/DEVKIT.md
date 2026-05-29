# WHMCS Domain Forwarding Service Devkit

## Overview
A domain forwarding service for WHMCS that enables URL redirection, 301/302 redirects, masked forwarding, and DNS-based forwarding with traffic analytics and control.

## Features
- URL forwarding (301, 302, 307, 308)
- Masked/silent forwarding with iframe
- DNS-based CNAME/MX forwarding
- Path forwarding patterns
- Wildcard forwarding rules
- Traffic analytics and logging
- A/B testing for forwarded URLs
- SSL certificate for forwarded domains
- Deep link preservation
- Query parameter handling

## WHMCS Integration Points
- Module: addon/DomainForwarding
- Hook: DomainDNSUpdate
- Hook: ServiceCreate

## Database Schema
```sql
CREATE TABLE mod_forwarding_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    domain_id INT NOT NULL,
    source_path VARCHAR(500) DEFAULT '/',
    destination_url VARCHAR(1000) NOT NULL,
    redirect_type ENUM('301', '302', '307', '308', 'masked') DEFAULT '301',
    is_active TINYINT(1) DEFAULT 1,
    hits_count INT DEFAULT 0,
    last_hit_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_domain (domain_id)
);

CREATE TABLE mod_forwarding_analytics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    rule_id INT NOT NULL,
    visitor_ip VARCHAR(45),
    user_agent VARCHAR(500),
    referrer VARCHAR(1000),
    source_query_params TEXT,
    forwarded_to VARCHAR(1000),
    redirected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## API Endpoints
- POST /api/forwarding/rules - Create forwarding rule
- GET /api/forwarding/rules/:domain - Get domain rules
- PUT /api/forwarding/rules/:id - Update rule
- DELETE /api/forwarding/rules/:id - Delete rule
- GET /api/forwarding/analytics/:rule_id - Get analytics

## Testing Checklist
- [ ] 301/302 redirects work correctly
- [ ] Masked forwarding displays correctly
- [ ] Analytics tracking accuracy
- [ ] Path pattern matching
- [ ] Wildcard forwarding
- [ ] SSL certificate provisioning

## Dependencies
- WHMCS 8.x+
- PHP 8.1+

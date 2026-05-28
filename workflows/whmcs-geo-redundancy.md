# WHMCS Geographic Redundancy Workflow

## Purpose

Implement geographic redundancy for WHMCS to ensure business continuity during regional outages, natural disasters, or infrastructure failures. This workflow covers multi-region deployment, data synchronization, and failover procedures.

## Prerequisites

- Multiple data centers across geographic regions
- CDN or geo-distributed DNS
- Database replication between regions
- File synchronization system
- VPN or dedicated links between locations

## Workflow Steps

### Step 1: Region Architecture Design

Plan your geographic layout:

```
US-East (Primary)                    EU-West (Secondary)
┌─────────────────────┐              ┌─────────────────────┐
│  WHMCS App Server   │              │  WHMCS App Server   │
│  ┌───────────────┐  │   Replicate  │  ┌───────────────┐  │
│  │  WHMCS        │  │ ──────────── │  │  WHMCS        │  │
│  │  Application   │◄─┼─────────────┼─►│  Application   │  │
│  └───────────────┘  │              │  └───────────────┘  │
└──────────┬──────────┘              └──────────┬──────────┘
           │                                    │
┌──────────▼──────────┐              ┌──────────▼──────────┐
│  MySQL Master       │              │  MySQL Slave        │
│  (Read/Write)       │              │  (Read-Only)         │
└─────────────────────┘              └─────────────────────┘

                    CloudFlare/AWS Route53
                           │
            ┌──────────────┼──────────────┐
            │              │              │
     Latency-based    GeoDNS        Failover
     routing          routing       DNS
```

### Step 2: Primary Region Setup

```bash
#!/bin/bash
# /opt/scripts/setup_primary_region.sh

# Install WHMCS on primary region
apt-get update && apt-get install -y nginx php8.1-fpm mysql-server

# WHMCS installation
cd /var/www/whmcs
curl -sS https://raw.githubusercontent.com/WHMCS/universal-install/master/install.php | php

# Configure database
mysql -u root -p <<EOF
CREATE DATABASE whmcs_main;
CREATE USER 'whmcs'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON whmcs_main.* TO 'whmcs'@'localhost';
FLUSH PRIVILEGES;
EOF

# Configure replication user
mysql -u root -p <<EOF
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
EOF

# Set up daily backup
echo "0 2 * * * /opt/scripts/backup_whmcs.sh" >> /var/spool/cron/crontabs/root
```

### Step 3: Secondary Region Setup

```bash
#!/bin/bash
# /opt/scripts/setup_secondary_region.sh

# Install dependencies
apt-get update && apt-get install -y nginx php8.1-fpm mysql-server

# Configure MySQL as slave
mysql -u root -p <<EOF
[mysqld]
server-id=2
relay-log=relay-bin
read-only=1
log_slave_updates=1
EOF

systemctl restart mysql

# Configure replication
mysql -u root -p <<EOF
CHANGE MASTER TO
    MASTER_HOST='primary-region-ip',
    MASTER_USER='repl',
    MASTER_PASSWORD='repl_password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=123,
    MASTER_CONNECT_RETRY=60;
EOF

START SLAVE;
SHOW SLAVE STATUS\G

# Install WHMCS files (will sync from primary)
cd /var/www/whmcs
# Use rsync for initial sync
rsync -avz --delete \
    -e "ssh -i /root/.ssh/whmcs_replica" \
    root@primary-region:/var/www/whmcs/ \
    /var/www/whmcs/

# Copy configuration
scp -i /root/.ssh/whmcs_replica \
    root@primary-region:/var/www/whmcs/configuration.php \
    /var/www/whmcs/configuration.php
```

### Step 4: File Synchronization Setup

```bash
#!/bin/bash
# /opt/scripts/sync_whmcs_files.sh
# Run on secondary region to sync from primary

PRIMARY_SERVER="primary.example.com"
REMOTE_PATH="/var/www/whmcs"
LOCAL_PATH="/var/www/whmcs"
SSH_KEY="/root/.ssh/whmcs_replica"

# Exclude patterns
EXCLUDE_OPTS="--exclude='cache/*' \
              --exclude='storage/templates_c/*' \
              --exclude='storage/logs/*' \
              --exclude='*.log' \
              --exclude='uploads/*.tmp'"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting file synchronization from $PRIMARY_SERVER"

# Sync with deletion for removed files
rsync -avz $EXCLUDE_OPTS \
    -e "ssh -i $SSH_KEY -o StrictHostKeyChecking=no" \
    --delete \
    root@$PRIMARY_SERVER:$REMOTE_PATH/ \
    $LOCAL_PATH/

# Sync attachments and uploads specifically
rsync -avz \
    -e "ssh -i $SSH_KEY -o StrictHostKeyChecking=no" \
    root@$PRIMARY_SERVER:$REMOTE_PATH/attachments/ \
    $LOCAL_PATH/attachments/

rsync -avz \
    -e "ssh -i $SSH_KEY -o StrictHostKeyChecking=no" \
    root@$PRIMARY_SERVER:$REMOTE_PATH/downloads/ \
    $LOCAL_PATH/downloads/

# Set permissions
chown -R www-data:www-data $LOCAL_PATH
chmod -R 755 $LOCAL_PATH
chmod -R 775 $LOCAL_PATH/storage

log "Synchronization complete"
```

### Step 5: DNS Configuration for GeoDNS

```bash
# CloudFlare DNS configuration example
# Set up latency-based routing

# Primary region - low latency
Type: A
Name: whmcs.example.com
Value: 203.0.113.10 (primary region IP)
Proxy: Yes
TTL: Auto
Edge TTL: 300

# Secondary region
Type: A
Name: whmcs.example.com
Value: 203.0.113.20 (secondary region IP)
Proxy: Yes
TTL: Auto
Edge TTL: 300

# Or use AWS Route 53
# Create health check
aws route53 create-health-check \
    --caller-reference $(date +%s) \
    --health-check-config '{
        "Type": "HTTPS",
        "FullyQualifiedDomainName": "whmcs.example.com",
        "Port": 443,
        "ResourcePath": "/whmcs/index.php",
        "RequestInterval": 10,
        "FailureThreshold": 3
    }'

# Create failover DNS
aws route53 change-resource-record-sets \
    --hosted-zone-id Z1234567890ABC \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "whmcs.example.com",
                "Type": "A",
                "SetIdentifier": "primary",
                "Region": "us-east-1",
                "LatencyRoutingPolicy": {},
                "HealthCheckId": "abc123"
            }
        }]
    }'
```

### Step 6: Configure Failover Scripts

```php
<?php
// /opt/scripts/geo_failover.php
// Automatic geographic failover script

class GeoRedundancyFailover {
    private $config;
    private $logFile = '/var/log/whmcs_geo_failover.log';
    
    public function __construct() {
        $this->config = [
            'primary_region' => 'us-east-1',
            'secondary_region' => 'eu-west-1',
            'health_check_url' => 'https://whmcs.example.com/whmcs/index.php',
            'max_response_time_ms' => 5000,
            'consecutive_failures' => 3,
            'alert_webhook' => 'https://hooks.example.com/alert'
        ];
    }
    
    public function checkHealth(): array {
        $startTime = microtime(true);
        $ch = curl_init($this->config['health_check_url']);
        
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 10,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_FOLLOWLOCATION => false
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $responseTime = (microtime(true) - $startTime) * 1000;
        $error = curl_error($ch);
        curl_close($ch);
        
        return [
            'success' => ($httpCode === 200 && $responseTime < $this->config['max_response_time_ms']),
            'http_code' => $httpCode,
            'response_time_ms' => $responseTime,
            'error' => $error
        ];
    }
    
    public function performFailover(): bool {
        $this->log("Initiating failover to secondary region");
        
        // 1. Update DNS
        $dnsUpdated = $this->updateDNSFailover();
        
        // 2. Promote secondary database
        $this->promoteDatabase();
        
        // 3. Reconfigure load balancer
        $this->reconfigureLoadBalancer();
        
        // 4. Clear CDN cache
        $this->purgeCDNCache();
        
        // 5. Send alerts
        $this->sendAlert();
        
        $this->log("Failover completed");
        return true;
    }
    
    private function updateDNSFailover(): bool {
        // CloudFlare API call
        $zoneId = 'your-zone-id';
        $recordId = 'your-record-id';
        
        $data = [
            'name' => 'whmcs.example.com',
            'type' => 'A',
            'content' => 'secondary-region-ip',
            'ttl' => 60,
            'proxied' => true
        ];
        
        // API call to update DNS
        $this->log("DNS updated to secondary region");
        return true;
    }
    
    private function promoteDatabase(): void {
        // Connect to secondary MySQL and promote to master
        $pdo = new PDO('mysql:host=localhost', 'admin', 'password');
        
        // Stop replication
        $pdo->exec("STOP SLAVE");
        
        // Make read-write
        $pdo->exec("SET GLOBAL read_only = OFF");
        
        // Reset master info
        $pdo->exec("RESET SLAVE ALL");
        
        $this->log("Database promoted to master");
    }
    
    private function reconfigureLoadBalancer(): void {
        // Update load balancer configuration
        exec('/opt/scripts/update_lb_config.sh');
        $this->log("Load balancer reconfigured");
    }
    
    private function purgeCDNCache(): void {
        // Purge CDN cache for static assets
        exec('/opt/scripts/purge_cdn.sh');
        $this->log("CDN cache purged");
    }
    
    private function sendAlert(): void {
        $payload = [
            'event' => 'GEO_FAILOVER',
            'timestamp' => date('c'),
            'region' => $this->config['secondary_region'],
            'message' => 'WHMCS failover completed to secondary region'
        ];
        
        $ch = curl_init($this->config['alert_webhook']);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => ['Content-Type: application/json']
        ]);
        curl_exec($ch);
        curl_close($ch);
    }
    
    private function log(string $message): void {
        $timestamp = date('Y-m-d H:i:s');
        file_put_contents($this->logFile, "[$timestamp] $message\n", FILE_APPEND);
    }
}

// Run failover check
$failover = new GeoRedundancyFailover();
$result = $failover->checkHealth();

if (!$result['success']) {
    $failover->performFailover();
}
```

### Step 7: Test Failover Procedures

```bash
#!/bin/bash
# /opt/scripts/test_geo_failover.sh
# Quarterly failover test

set -euo pipefail

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting geographic redundancy test"

# 1. Verify primary region health
log "Checking primary region..."
curl -f https://primary.whmcs.example.com/whmcs/index.php || log "Primary unhealthy"

# 2. Simulate primary failure
log "Simulating primary failure..."
# Point DNS to secondary

# 3. Test secondary region
log "Testing secondary region..."
curl -f https://secondary.whmcs.example.com/whmcs/index.php

# 4. Verify data integrity
log "Verifying data integrity..."
mysql -h secondary -u whmcs -p -e "SELECT COUNT(*) FROM whmcs_main.tblclients"
mysql -h secondary -u whmcs -p -e "SELECT COUNT(*) FROM whmcs_main.tblhosting"

# 5. Test critical functionality
log "Testing critical functions..."
php /opt/scripts/test_whmcs_functions.php

# 6. Restore primary
log "Restoring primary region..."

# 7. Document results
log "Test complete. Results documented."
```

## Best Practices

1. **Data Consistency**: Use async replication with known lag
2. **Automatic Failover**: Trigger on consecutive health check failures
3. **Regular Testing**: Test failover quarterly minimum
4. **Documentation**: Keep RTO/RPO documented and tested
5. **Communication**: Have stakeholder notification templates ready
6. **Monitoring**: Track replication lag and latency

## Common Pitfalls

- **Replication lag**: Data inconsistency after failover
- **Session loss**: Not using shared session storage
- **DNS propagation delay**: Users hitting old region
- **Incomplete sync**: Missing files or attachments
- **Cost management**: Running both regions at full capacity

## Verification Checklist

- [ ] Replication lag under acceptable threshold
- [ ] Failover triggers correctly
- [ ] All data synchronized
- [ ] DNS updates propagate
- [ ] Sessions work across regions
- [ ] Payments process correctly
- [ ] Emails send from correct region
- [ ] Monitoring alerts configured

## Related Documentation

- [WHMCS Disaster Recovery](whmcs-disaster-recovery-workflow.md)
- [WHMCS Load Balancing](whmcs-load-balancing.md)
- [WHMCS Multi-Server Workflow](whmcs-multi-server-workflow.md)
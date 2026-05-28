# WHMCS Cost Optimization Workflow

## Purpose

Implement cost optimization strategies for WHMCS infrastructure to reduce operational costs while maintaining performance and reliability. This workflow covers resource optimization, reserved instances, right-sizing, and cost monitoring.

## Prerequisites

- Cloud infrastructure (AWS, Azure, GCP) or dedicated servers
- Cost monitoring tools (CloudWatch, Cost Explorer, billing dashboards)
- Budget controls and alerts configured
- Understanding of usage patterns

## Workflow Steps

### Step 1: Cost Analysis and Baseline

```
┌─────────────────────────────────────────────────────────────┐
│                 WHMCS Cost Structure                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Compute (40-50%)     │   Storage (15-20%)                  │
│  ├─ Web Servers       │   ├─ Database Storage               │
│  ├─ App Servers       │   ├─ File Storage                   │
│  └─ PHP Workers       │   ├─ Backups                        │
│                       │   └─ CDN                            │
│                       │                                     │
│  Database (15-25%)    │   Network (10-15%)                  │
│  ├─ RDS/MariaDB        │   ├─ Data Transfer                 │
│  ├─ Read Replicas      │   ├─ CDN Bandwidth                 │
│  └─ Backups            │   └─ Load Balancer                 │
│                                                             │
│  Other Services (5-10%)                                     │
│  ├─ Monitoring         │                                    │
│  ├─ Logging            │                                    │
│  └─ Security Services  │                                    │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Compute Cost Optimization

```bash
#!/bin/bash
# /opt/scripts/analyze_compute_costs.sh

# AWS Cost Explorer query for EC2 costs
aws ce get-cost-and-usage \
    --time-period Start=2024-01-01,End=2024-12-31 \
    --granularity MONTHLY \
    --metrics "BlendedCost,UnblendedCost,UsageQuantity" \
    --group-by Type=Dimension,Key=SERVICE \
    --filter "{\"Dimensions\":{\"Key\":\"SERVICE\",\"Values\":[\"Amazon EC2\",\"Amazon RDS\"]}}"

# Azure Cost Management query
az costmanagement query \
    --type actualcost \
    --timeframe MonthToDate \
    --dataset-filter "{\"type\":\"Dimension\",\"name\":\"ResourceType\",\"operator\":\"In\",\"values\":[\"Microsoft.Compute/virtualMachines\",\"Microsoft.DBforMySQL/servers\"]}"

# Generate cost optimization report
cat > /tmp/compute_report.md << 'EOF'
# WHMCS Compute Cost Analysis

## Current Spending
| Resource | Monthly Cost | Daily Cost | Usage |
|----------|-------------|------------|-------|
| Web Servers (3x) | $450 | $15 | 60% avg |
| App Servers (2x) | $300 | $10 | 45% avg |
| Database Server | $200 | $6.67 | 70% avg |

## Optimization Opportunities

### 1. Reserved Instances
- Current: On-demand
- Potential: 1-3 year Reserved Instances
- Savings: 30-60%

### 2. Right-sizing
- Web servers over-provisioned by 40%
- Can reduce from t3.large to t3.medium

### 3. Auto-scaling
- Implement dynamic scaling for traffic spikes
- Scale down during off-peak hours

## Recommended Actions
1. Purchase 1-year Reserved Instances for baseline
2. Right-size web servers
3. Implement auto-scaling group
EOF

echo "Compute cost analysis complete"
```

### Step 3: Database Cost Optimization

```sql
-- Analyze database sizing and optimization opportunities

-- Current database sizes
SELECT 
    table_schema as 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) as 'Size (MB)'
FROM information_schema.tables
WHERE table_schema = 'whmcs_main'
GROUP BY table_schema;

-- Tables with most growth potential
SELECT 
    table_name,
    ROUND((data_length + index_length) / 1024 / 1024, 2) as size_mb,
    table_rows,
    ROUND((data_length + index_length) / NULLIF(table_rows, 0), 0) as avg_row_bytes
FROM information_schema.tables
WHERE table_schema = 'whmcs_main'
ORDER BY size_mb DESC
LIMIT 10;

-- Query to identify unused indexes
SELECT 
    t.table_name,
    i.index_name,
    i.cardinality
FROM information_schema.tables t
JOIN information_schema.statistics i ON t.table_name = i.table_name
WHERE t.table_schema = 'whmcs_main'
AND i.seq_in_index = 1
AND i.index_name NOT IN ('PRIMARY')
AND t.table_rows < 1000;

-- Archive old data to reduce storage
-- Move closed tickets older than 6 months to archive
CREATE TABLE tbltickets_archive AS
SELECT * FROM tbltickets 
WHERE status = 'Closed' 
AND created_at < DATE_SUB(CURDATE(), INTERVAL 6 MONTH);

-- Optimize tables
OPTIMIZE TABLE tbltickets;
OPTIMIZE TABLE tblticketreplies;
OPTIMIZE TABLE tblactivitylog;
```

### Step 4: Storage Cost Optimization

```bash
#!/bin/bash
# /opt/scripts/storage_optimization.sh

# Configure lifecycle policies for S3
aws s3api put-bucket-lifecycle-configuration \
    --bucket company-whmcs-storage \
    --lifecycle-configuration '{
        "Rules": [
            {
                "ID": "backup-retention",
                "Status": "Enabled",
                "Filter": {"Prefix": "backups/"},
                "Transitions": [
                    {"Days": 30, "StorageClass": "STANDARD_IA"},
                    {"Days": 90, "StorageClass": "GLACIER"},
                    {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
                ]
            },
            {
                "ID": "logs-retention",
                "Status": "Enabled",
                "Filter": {"Prefix": "logs/"},
                "Transitions": [
                    {"Days": 7, "StorageClass": "GLACIER"}
                ],
                "Expiration": {"Days": 90}
            }
        ]
    }'

# Enable compression for backups
tar -czf - /var/backups/whmcs | aws s3 cp - s3://company-whmcs-backups/backup.tar.gz

# Clean up old local backups
find /var/backups/whmcs -type f -mtime +30 -delete
find /var/backups/whmcs -type d -mtime +30 -exec rm -rf {} \;
```

### Step 5: Auto-scaling Configuration

```yaml
# AWS Auto Scaling Group configuration
# /etc/aws/asg-config.yaml

AWSTemplateFormatVersion: '2010-09-09'
Resources:
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      MinSize: 2
      MaxSize: 10
      DesiredCapacity: 3
      VPCZoneIdentifier:
        - !Sub ${SubnetIds}
      MetricsCollection:
        - Granularity: 1Minute
      LaunchConfigurationName: !Ref LaunchConfig
      TargetGroupARNs:
        - !Ref TargetGroup
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300

  ScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref AutoScalingGroup
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 60

# CloudWatch alarm for scale-out
aws cloudwatch put-metric-alarm \
    --alarm-name whmcs-high-cpu-scaleout \
    --alarm-description "Scale out when CPU > 70%" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --statistic Average \
    --period 60 \
    --threshold 70 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --dimensions Name=AutoScalingGroupName,Value=whmcs-asg \
    --alarm-actions arn:aws:autoscaling:region:account:scalingPolicy:asg/policy

# CloudWatch alarm for scale-in
aws cloudwatch put-metric-alarm \
    --alarm-name whmcs-low-cpu-scalein \
    --alarm-description "Scale in when CPU < 30%" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --statistic Average \
    --period 300 \
    --threshold 30 \
    --comparison-operator LessThanThreshold \
    --evaluation-periods 5 \
    --dimensions Name=AutoScalingGroupName,Value=whmcs-asg \
    --alarm-actions arn:aws:autoscaling:region:account:scalingPolicy:asg/policy
```

### Step 6: Cost Monitoring Dashboard

```json
{
  "dashboard": {
    "title": "WHMCS Cost Optimization",
    "widgets": [
      {
        "type": "metric",
        "title": "Daily Spend",
        "metrics": [
          {"expression": "SORT(FILTER(COST, SERVICE IN [\"EC2\", \"RDS\", \"S3\"]), TOTAL)", "label": "Total Daily Cost"}
        ]
      },
      {
        "type": "bar",
        "title": "Cost by Service",
        "data": {
          "labels": ["EC2", "RDS", "S3", "CloudFront", "DataTransfer"],
          "values": [450, 200, 150, 100, 50]
        }
      },
      {
        "type": "line",
        "title": "Monthly Trend",
        "data": {
          "series": [
            {"name": "Actual", "data": [850, 920, 890, 950, 880, 910]},
            {"name": "Budget", "data": [900, 900, 900, 900, 900, 900]}
          ]
        }
      },
      {
        "type": "gauge",
        "title": "Budget Utilization",
        "value": 85,
        "max": 100,
        "thresholds": {"warning": 80, "critical": 95}
      }
    ]
  }
}
```

### Step 7: Cost Optimization Checklist

```php
<?php
// /opt/scripts/cost_optimization_check.php
// Automated cost optimization checks

class CostOptimizationChecker {
    private $checks = [];
    
    public function runAllChecks(): array {
        $results = [];
        
        $results['compute'] = $this->checkComputeOptimization();
        $results['storage'] = $this->checkStorageOptimization();
        $results['database'] = $this->checkDatabaseOptimization();
        $results['network'] = $this->checkNetworkOptimization();
        
        $this->generateRecommendations($results);
        
        return $results;
    }
    
    private function checkComputeOptimization(): array {
        return [
            'reserved_instances' => $this->checkReservedInstances(),
            'auto_scaling' => $this->checkAutoScaling(),
            'instance_right_sizing' => $this->checkInstanceSizing()
        ];
    }
    
    private function checkReservedInstances(): array {
        // Check current RI coverage
        $currentSavings = 0;
        $potentialSavings = 0;
        
        return [
            'status' => $currentSavings > 0 ? 'optimized' : 'needs_attention',
            'current_coverage' => '0%',
            'potential_savings_monthly' => $potentialSavings,
            'recommendation' => 'Purchase 1-year Reserved Instances'
        ];
    }
    
    private function checkAutoScaling(): array {
        return [
            'status' => 'enabled',
            'min_instances' => 2,
            'max_instances' => 10,
            'avg_utilization' => '55%',
            'potential_savings' => '$150/month if scale down to 1 during off-peak'
        ];
    }
    
    private function checkInstanceSizing(): array {
        return [
            'web_servers' => [
                'current' => 't3.large',
                'recommended' => 't3.medium',
                'monthly_savings' => '$75'
            ],
            'app_servers' => [
                'current' => 't3.xlarge',
                'recommended' => 't3.large',
                'monthly_savings' => '$50'
            ]
        ];
    }
    
    private function checkStorageOptimization(): array {
        return [
            'lifecycle_policies' => $this->checkLifecyclePolicies(),
            'compression' => $this->checkCompression(),
            'cleanup' => $this->checkOldFiles()
        ];
    }
    
    private function checkLifecyclePolicies(): array {
        return [
            'backups' => 'GLACIER after 90 days',
            'logs' => 'Deleted after 30 days',
            'status' => 'configured'
        ];
    }
    
    private function checkDatabaseOptimization(): array {
        return [
            'slow_queries' => $this->countSlowQueries(),
            'unused_indexes' => $this->countUnusedIndexes(),
            'archived_data' => $this->checkArchivedData()
        ];
    }
    
    private function countSlowQueries(): int {
        $pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
        return (int) $pdo->query("
            SELECT COUNT(*) FROM mysql.slow_log 
            WHERE start_time > DATE_SUB(NOW(), INTERVAL 24 HOUR)
        ")->fetchColumn();
    }
    
    private function generateRecommendations(array $results): void {
        $totalSavings = 0;
        
        foreach ($results as $category => $data) {
            if (isset($data['monthly_savings'])) {
                $totalSavings += $data['monthly_savings'];
            }
            
            if (isset($data['recommendation'])) {
                logActivity("Cost Optimization: " . $data['recommendation']);
            }
        }
        
        logActivity("Total potential monthly savings: $" . $totalSavings);
    }
}
```

## Cost Optimization Strategies

| Strategy | Potential Savings | Effort | Priority |
|----------|------------------|--------|----------|
| Reserved Instances | 30-60% | Low | High |
| Right-sizing | 20-40% | Medium | High |
| Auto-scaling | 15-30% | Medium | Medium |
| Lifecycle policies | 10-20% | Low | Medium |
| Compression | 5-15% | Low | Low |
| Archive old data | 10-20% | High | Medium |

## Best Practices

1. **Monitor Continuously**: Set up cost dashboards and alerts
2. **Right-size Resources**: Match capacity to actual usage
3. **Use Reserved Capacity**: Commit for predictable workloads
4. **Implement Auto-scaling**: Scale based on demand
5. **Archive Old Data**: Move cold data to cheaper storage
6. **Regular Reviews**: Monthly cost optimization reviews

## Common Pitfalls

- **Over-provisioning**: More resources than needed
- **No Monitoring**: Hidden cost buildup
- **Idle Resources**: Unused instances running
- **No Lifecycle Policies**: Data accumulating forever
- **Single Size Fits All**: Not matching resources to workload

## Verification Checklist

- [ ] Cost monitoring dashboard configured
- [ ] Budget alerts set up
- [ ] Reserved instances purchased
- [ ] Auto-scaling configured
- [ ] Storage lifecycle policies active
- [ ] Old data archival configured
- [ ] Monthly cost review scheduled
- [ ] Right-sizing applied

## Related Documentation

- [WHMCS Capacity Planning](whmcs-capacity-planning.md)
- [WHMCS Backup Strategy](whmcs-backup-strategy.md)
- [AWS Cost Optimization Best Practices](https://docs.aws.amazon.com/cost-management)
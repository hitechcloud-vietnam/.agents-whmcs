# WHMCS Capacity Planning Workflow

## Purpose

Implement capacity planning for WHMCS to ensure infrastructure scales appropriately with demand, avoid performance issues, and optimize resource allocation. This workflow covers capacity modeling, forecasting, and scaling strategies.

## Prerequisites

- Historical metrics data (at least 3 months)
- Performance monitoring system
- Growth projections and business plans
- Understanding of system limits

## Workflow Steps

### Step 1: Capacity Model Design

```
┌─────────────────────────────────────────────────────────────┐
│               WHMCS Capacity Planning Model                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Demand Forecast                     │   │
│  │  • Historical usage patterns                         │   │
│  │  • Seasonal variations                               │   │
│  │  • Business growth projections                       │   │
│  │  • Marketing campaign impacts                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                             │                               │
│                    ┌─────────▼─────────┐                   │
│                    │  Capacity Model    │                   │
│                    │  ┌─────────────┐  │                   │
│                    │  │  Compute    │  │                   │
│                    │  │  Storage    │  │                   │
│                    │  │  Network    │  │                   │
│                    │  │  Database   │  │                   │
│                    │  └─────────────┘  │                   │
│                    └─────────┬─────────┘                   │
│                              │                              │
│                    ┌─────────▼─────────┐                   │
│                    │  Resource Plans    │                   │
│                    │  • Scale-up       │                   │
│                    │  • Scale-out      │                   │
│                    │  • Optimization   │                   │
│                    └───────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Current Capacity Assessment

```bash
#!/bin/bash
# /opt/scripts/assess_capacity.sh

echo "=== WHMCS Capacity Assessment ==="
echo "Generated: $(date)"
echo ""

# Web Server Capacity
echo "=== Web Server Capacity ==="
for server in web1 web2 web3; do
    echo "Server: $server"
    ssh $server "echo 'CPU: ' && top -bn1 | grep 'Cpu(s)' | awk '{print \$2}' && \
                 echo 'Memory: ' && free -h | grep Mem | awk '{print \$3\"/\" \$2}' && \
                 echo 'Load: ' && uptime | awk -F'load average:' '{print \$2}'"
done

# PHP-FPM Capacity
echo ""
echo "=== PHP-FPM Status ==="
curl -s http://localhost/status | grep "pool\|active\|max children\|idle"

# Database Capacity
echo ""
echo "=== Database Capacity ==="
mysql -u root -p -e "
SELECT 
    @@max_connections as max_conn,
    @@wait_timeout as wait_timeout,
    @@innodb_buffer_pool_size as buffer_pool;
SELECT 
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) as total_mb
FROM information_schema.tables WHERE table_schema = 'whmcs_main';
"

# Storage Capacity
echo ""
echo "=== Storage Capacity ==="
df -h /var/www/whmcs
du -sh /var/www/whmcs/attachments
du -sh /var/www/whmcs/uploads

# Network Capacity
echo ""
echo "=== Network Capacity ==="
cat /proc/net/dev | grep eth0 | awk '{print "RX: " $2 " TX: " $10}'
```

### Step 3: Usage Pattern Analysis

```python
#!/usr/bin/env python3
# /opt/scripts/analyze_capacity_patterns.py

import json
from datetime import datetime, timedelta
from collections import defaultdict

class CapacityAnalyzer:
    def __init__(self):
        self.data = self.load_metrics()
    
    def analyze_seasonality(self) -> dict:
        """Identify seasonal patterns in usage"""
        
        # Group metrics by month
        monthly_metrics = defaultdict(lambda: {'requests': 0, 'peak': 0})
        
        for record in self.data:
            month = record['timestamp'][:7]  # YYYY-MM
            monthly_metrics[month]['requests'] += record['requests']
            monthly_metrics[month]['peak'] = max(
                monthly_metrics[month]['peak'],
                record['peak_concurrent']
            )
        
        # Calculate growth rate
        months = sorted(monthly_metrics.keys())
        growth_rates = []
        for i in range(1, len(months)):
            prev = monthly_metrics[months[i-1]]['requests']
            curr = monthly_metrics[months[i]]['requests']
            if prev > 0:
                growth_rates.append((curr - prev) / prev)
        
        avg_growth = sum(growth_rates) / len(growth_rates) if growth_rates else 0
        
        return {
            'monthly_data': dict(monthly_metrics),
            'average_growth_rate': avg_growth,
            'projected_6_month': self.project_usage(6),
            'seasonal_factors': self.identify_seasonal_factors()
        }
    
    def identify_seasonal_factors(self) -> dict:
        """Identify seasonal patterns (month of year effects)"""
        
        monthly_avg = defaultdict(lambda: {'total': 0, 'count': 0})
        for record in self.data:
            month = int(record['timestamp'][5:7])
            monthly_avg[month]['total'] += record['requests']
            monthly_avg[month]['count'] += 1
        
        # Calculate average for each month
        seasonal_index = {}
        grand_avg = sum(m['total'] / m['count'] for m in monthly_avg.values()) / len(monthly_avg)
        
        for month, data in monthly_avg.items():
            avg = data['total'] / data['count']
            seasonal_index[month] = avg / grand_avg if grand_avg else 1
        
        return seasonal_index
    
    def project_usage(self, months: int) -> dict:
        """Project usage for next N months"""
        
        current_usage = self.get_current_usage()
        growth_rate = self.analyze_seasonality()['average_growth_rate']
        
        projections = {}
        for i in range(1, months + 1):
            factor = (1 + growth_rate) ** i
            projections[f"+{i}month"] = {
                'requests': current_usage['requests'] * factor,
                'storage_gb': current_usage['storage_gb'] * factor,
                'bandwidth_gb': current_usage['bandwidth_gb'] * factor
            }
        
        return projections
    
    def calculate_required_capacity(self) -> dict:
        """Calculate required capacity based on projections"""
        
        projections = self.project_usage(6)
        current = self.get_current_resources()
        
        # Add 20% buffer for safety
        buffer = 1.2
        
        required = {
            'servers': {
                'current': current['web_servers'],
                'required_6m': int(projections['+6month']['requests'] / 10000 * buffer),
                'scaling_needed': projections['+6month']['requests'] / 10000 > current['web_servers']
            },
            'storage': {
                'current_gb': current['storage_gb'],
                'required_6m': projections['+6month']['storage_gb'] * buffer,
                'purchase_plan': '1TB SSD now, plan 2TB for +6m'
            },
            'database': {
                'current': current['db_instances'],
                'required_6m': 'Consider read replica if queries > 500/s',
                'recommendation': 'Monitor query rate and add replica at 400/s'
            }
        }
        
        return required
    
    def generate_capacity_plan(self) -> str:
        """Generate comprehensive capacity plan"""
        
        projections = self.project_usage(12)
        required = self.calculate_required_capacity()
        
        plan = []
        plan.append("# WHMCS Capacity Plan")
        plan.append(f"Generated: {datetime.now().strftime('%Y-%m-%d')}")
        plan.append("")
        plan.append("## Current Capacity")
        plan.append(f"- Web Servers: {required['servers']['current']}")
        plan.append(f"- Storage: {required['storage']['current_gb']} GB")
        plan.append(f"- Database Instances: {required['database']['current']}")
        plan.append("")
        plan.append("## Projected Demand")
        
        for period, data in projections.items():
            plan.append(f"\n### {period}")
            plan.append(f"- Requests: {data['requests']:,.0f}")
            plan.append(f"- Storage: {data['storage_gb']:.1f} GB")
        
        plan.append("")
        plan.append("## Recommendations")
        
        if required['servers']['scaling_needed']:
            plan.append(f"\n### Scale Web Servers")
            plan.append(f"- Current: {required['servers']['current']}")
            plan.append(f"- Required in 6 months: {required['servers']['required_6m']}")
            plan.append("- Action: Add capacity before peak season")
        
        plan.append("\n### Storage")
        plan.append(f"- Current: {required['storage']['current_gb']} GB")
        plan.append(f"- Required: {required['storage']['required_6m']} GB")
        plan.append(f"- {required['storage']['purchase_plan']}")
        
        return "\n".join(plan)
```

### Step 4: Scaling Strategy Implementation

```yaml
# /etc/kubernetes/whmcs-capacity-config.yaml
# Kubernetes HPA (Horizontal Pod Autoscaler) configuration

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: whmcs-web-hpa
  namespace: whmcs
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: whmcs-web
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max

---
# Vertical Pod Autoscaler for database
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: whmcs-db-vpa
  namespace: whmcs
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: StatefulSet
    name: whmcs-mysql
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: mysql
        minAllowed:
          cpu: 500m
          memory: 512Mi
        maxAllowed:
          cpu: 8
          memory: 32Gi
```

### Step 5: Capacity Monitoring Alerts

```yaml
# /etc/prometheus/capacity_alerts.yml
# Capacity-based alerts

groups:
  - name: capacity_alerts
    rules:
      # CPU Capacity Warning
      - alert: CPUCapacityWarning
        expr: avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) < 0.3
        for: 30m
        labels:
          severity: warning
          type: capacity
        annotations:
          summary: "CPU capacity running low"
          description: "{{ $labels.instance }} CPU utilization above 70% for 30 minutes"

      # Memory Capacity Warning
      - alert: MemoryCapacityWarning
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.85
        for: 15m
        labels:
          severity: warning
          type: capacity
        annotations:
          summary: "Memory capacity running low"
          description: "{{ $labels.instance }} memory usage above 85%"

      # Disk Capacity Warning
      - alert: DiskCapacityWarning
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes) < 0.15
        for: 10m
        labels:
          severity: warning
          type: capacity
        annotations:
          summary: "Disk space running low"
          description: "Only {{ $value | humanizePercentage }} disk space remaining"

      # Database Connections Warning
      - alert: DatabaseConnectionsWarning
        expr: rate(mysql_global_status_threads_connected[5m]) / mysql_global_variable_max_connections > 0.75
        for: 15m
        labels:
          severity: warning
          type: capacity
        annotations:
          summary: "Database connections approaching limit"
          description: "Using {{ $value | humanizePercentage }} of max connections"

      # Scaling Threshold Alert
      - alert: ScalingRecommendation
        expr: |
          avg(rate(http_requests_total[15m])) / 1000 > 
          count(node_cpu_seconds_total) * 0.5
        for: 1h
        labels:
          severity: info
          type: capacity
        annotations:
          summary: "Consider scaling out WHMCS"
          description: "Request rate suggests need for more instances"
```

### Step 6: Capacity Testing

```bash
#!/bin/bash
# /opt/scripts/capacity_test.sh
# Load testing for capacity planning

set -euo pipefail

TARGET_URL="https://whmcs.example.com"
CONCURRENT_USERS=(100 200 500 1000)
DURATION=300  # 5 minutes per test

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

run_load_test() {
    local users=$1
    log "Testing with $users concurrent users"
    
    # Use Apache Bench or k6 for load testing
    ab -n 10000 -c $users -g "results_${users}.tsv" "$TARGET_URL/whmcs/index.php"
    
    # Use k6 for more realistic testing
    k6 run --vus $users --duration "${DURATION}s" << 'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
    thresholds: {
        http_req_duration: ['p(95)<2000'],
        http_req_failed: ['rate<0.01'],
    },
};

export default function() {
    let res = http.get('https://whmcs.example.com/whmcs/index.php');
    check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 2s': (r) => r.timings.duration < 2000,
    });
    sleep(1);
}
EOF
}

# Run tests with increasing load
for users in "${CONCURRENT_USERS[@]}"; do
    run_load_test $users
    
    # Analyze results before next test
    log "Analyzing results for $users users..."
    php /opt/scripts/analyze_load_results.php results_${users}.tsv
done

log "Capacity testing complete. Review results for scaling recommendations."
```

## Capacity Planning Metrics

| Metric | Current | 3 Months | 6 Months | 12 Months |
|--------|---------|----------|---------|-----------|
| Users | 10,000 | 12,000 | 15,000 | 20,000 |
| Concurrent Sessions | 500 | 600 | 750 | 1,000 |
| Monthly Requests | 500K | 600K | 750K | 1M |
| Storage (GB) | 100 | 120 | 150 | 200 |
| DB Size (GB) | 50 | 60 | 75 | 100 |

## Scaling Triggers

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| CPU Usage | >70% | >85% | Scale up/out |
| Memory Usage | >80% | >90% | Scale up/out |
| Disk Usage | >75% | >90% | Add storage |
| Response Time (P95) | >1s | >2s | Scale out |
| DB Connections | >70% | >85% | Add read replica |

## Best Practices

1. **Plan Ahead**: 6-12 months capacity planning
2. **Monitor Trends**: Track growth patterns continuously
3. **Buffer Capacity**: 20-30% headroom recommended
4. **Test Regularly**: Load test quarterly
5. **Automate Scaling**: Use auto-scaling where possible
6. **Document Decisions**: Keep capacity planning records

## Common Pitfalls

- **Last Minute Scaling**: Reactive instead of proactive
- **Ignoring Growth Trends**: Not accounting for growth
- **Single Resource Focus**: Only monitoring one metric
- **No Testing**: Unknown actual capacity limits
- **Over-provisioning**: Wasting resources and money

## Verification Checklist

- [ ] Current capacity assessed
- [ ] Growth trends analyzed
- [ ] 6-month projections created
- [ ] Scaling triggers configured
- [ ] Auto-scaling implemented
- [ ] Load testing performed
- [ ] Capacity plan documented
- [ ] Budget allocated for scaling

## Related Documentation

- [WHMCS Cost Optimization](whmcs-cost-optimization.md)
- [WHMCS Monitoring and Alerts](whmcs-monitoring-alerts.md)
- [WHMCS Load Balancing](whmcs-load-balancing.md)
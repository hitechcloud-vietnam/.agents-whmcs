# WHMCS Auto Scaler Module

Auto scaling configuration for resources based on usage metrics and thresholds.

## Features

- Configurable scaling rules per product/service
- Scale-up and scale-down triggers
- Metric-based auto scaling (CPU, RAM, disk, bandwidth)
- Scheduled scaling rules
- Cooldown periods to prevent oscillation
- Scale history and logs
- Manual override capability
- Multiple scaling policies per service

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/autoscaler/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure scaling rules per service

## Usage

```php
// Create scaling rule
$result = autoscaler_CreateRule(array(
    'rule_name' => 'High CPU Auto Scale',
    'service_id' => $serviceId,
    'metric' => 'cpu_usage',
    'condition' => '>',
    'threshold' => 80,
    'scale_action' => 'scale_up',
    'scale_value' => 1,
    'cooldown_seconds' => 300,
    'min_instances' => 1,
    'max_instances' => 5
));

// Get scaling rules
$rules = autoscaler_GetRules($serviceId);

// Update rule
autoscaler_UpdateRule($ruleId, array(
    'threshold' => 85,
    'cooldown_seconds' => 600
));

// Trigger scaling evaluation
$result = autoscaler_EvaluateService($serviceId);

// Get service metrics
$metrics = autoscaler_GetServiceMetrics($serviceId, 24);

// Record metric value
autoscaler_RecordMetric($serviceId, 'cpu_usage', 75.5);

// Get scale history
$history = autoscaler_GetScaleHistory($serviceId, 100);

// Get active scaling operations
$active = autoscaler_GetActiveScalingOps($serviceId);

// Cancel scaling operation
autoscaler_CancelScalingOp($opId);

// Get scaling statistics
$stats = autoscaler_GetStats($serviceId, 30);

// Create scheduled rule
autoscaler_CreateScheduledRule(array(
    'rule_name' => 'Peak Hours Scale',
    'service_id' => $serviceId,
    'scale_action' => 'scale_up',
    'scale_value' => 2,
    'schedule' => array(
        'days' => array('monday', 'tuesday', 'wednesday', 'thursday', 'friday'),
        'start_time' => '09:00',
        'end_time' => '18:00'
    )
));

// Add scheduled scale
autoscaler_ScheduleScale($serviceId, $scaleValue, $timestamp);

// Set manual override
autoscaler_SetOverride($serviceId, 3, 'Manual capacity increase');

// Clear manual override
autoscaler_ClearOverride($serviceId);

// Evaluate all services
autoscaler_EvaluateAll();
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| EnableAutoScale | yesno | yes | Enable auto scaling |
| DefaultCooldown | text | 300 | Default cooldown (seconds) |
| DefaultMinInstances | text | 1 | Minimum instances |
| DefaultMaxInstances | text | 10 | Maximum instances |
| EvaluationInterval | text | 60 | Evaluation interval (seconds) |
| EnableScheduledScale | yesno | yes | Enable scheduled scaling |
| EnableMetricsLogging | yesno | yes | Log all metrics |

## Supported Metrics

| Metric | Description | Unit |
|--------|-------------|------|
| cpu_usage | CPU utilization | % |
| ram_usage | RAM utilization | % |
| disk_usage | Disk utilization | % |
| bandwidth_in | Incoming bandwidth | Mbps |
| bandwidth_out | Outgoing bandwidth | Mbps |
| connection_count | Active connections | count |
| request_count | HTTP requests | count |
| response_time | Avg response time | ms |
| error_rate | Error rate | % |

## Scale Actions

| Action | Description |
|--------|-------------|
| scale_up | Increase capacity |
| scale_down | Decrease capacity |
| scale_to | Set specific value |
| restart | Restart service |

## Condition Operators

| Operator | Description |
|----------|-------------|
| > | Greater than |
| < | Less than |
| >= | Greater or equal |
| <= | Less or equal |
| == | Equal to |

## Database Tables

- `mod_autoscaler_rules` - Scaling rules
- `mod_autoscaler_metrics` - Metric history
- `mod_autoscaler_history` - Scale history
- `mod_autoscaler_scheduled` - Scheduled scaling
- `mod_autoscaler_overrides` - Manual overrides
- `mod_autoscaler_operations` - Active operations

# WHMCS Incident Response Workflow

## Purpose

Establish comprehensive incident response procedures for WHMCS to minimize impact of issues, restore service quickly, and learn from incidents. This workflow covers detection, response, communication, resolution, and post-incident activities.

## Prerequisites

- Incident response team defined
- Communication channels established
- Monitoring and alerting configured
- Runbooks for common incidents
- Escalation matrix defined

## Workflow Steps

### Step 1: Incident Classification

```
┌─────────────────────────────────────────────────────────────┐
│                 Incident Severity Levels                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SEV-1: Critical (Immediate Response)                      │
│  ├─ Complete service outage                                │
│  ├─ Data loss or corruption                                │
│  ├─ Security breach                                        │
│  └─ RTO: 1 hour                                           │
│                                                             │
│  SEV-2: High (Fast Response)                               │
│  ├─ Major functionality impaired                          │
│  ├─ Large percentage of users affected                    │
│  ├─ Workaround not available                               │
│  └─ RTO: 4 hours                                          │
│                                                             │
│  SEV-3: Medium (Standard Response)                         │
│  ├─ Minor functionality impaired                          │
│  ├─ Single user or small group affected                    │
│  ├─ Workaround available                                   │
│  └─ RTO: 24 hours                                         │
│                                                             │
│  SEV-4: Low (Normal Response)                              │
│  ├─ Cosmetic issues                                        │
│  ├─ Minor bugs with workarounds                           │
│  └─ RTO: 1 week                                           │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Incident Response Procedure

```bash
#!/bin/bash
# /opt/scripts/incident_response.sh

INCIDENT_ID=$(date +%Y%m%d%H%M%S)
INCIDENT_SEVERITY=${1:-SEV-3}
SLACK_WEBHOOK="https://hooks.slack.com/services/XXX/YYY/ZZZ"

log_incident() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INCIDENT-$INCIDENT_ID] $1"
}

# Step 1: Acknowledge incident
acknowledge_incident() {
    log_incident "Acknowledging incident - Severity: $INCIDENT_SEVERITY"
    
    # Update status page
    curl -X POST "https://status.example.com/api/incidents" \
        -H "Authorization: Bearer $STATUS_API_KEY" \
        -d "{
            \"name\": \"Investigating Issue\",
            \"status\": \"investigating\",
            \"severity\": \"$INCIDENT_SEVERITY\",
            \"incident_id\": \"$INCIDENT_ID\"
        }"
    
    # Create incident channel in Slack
    /opt/scripts/create_incident_channel.sh "incident-$INCIDENT_ID"
}

# Step 2: Assess impact
assess_impact() {
    log_incident "Assessing impact..."
    
    # Check affected services
    php /opt/scripts/check_service_status.php
    
    # Check number of affected users
    php /opt/scripts/count_affected_users.php
    
    # Check if data integrity affected
    php /opt/scripts/check_data_integrity.php
}

# Step 3: Communicate
communicate() {
    local message="$1"
    log_incident "Communication: $message"
    
    # Post to Slack incident channel
    curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-Type: application/json' \
        -d "{
            \"channel\": \"#incident-$INCIDENT_ID\",
            \"text\": \"$message\",
            \"attachments\": [{
                \"color\": \"danger\",
                \"fields\": [
                    {\"title\": \"Incident ID\", \"value\": \"$INCIDENT_ID\", \"short\": true},
                    {\"title\": \"Severity\", \"value\": \"$INCIDENT_SEVERITY\", \"short\": true}
                ]
            }]
        }"
    
    # Update status page
    curl -X PUT "https://status.example.com/api/incidents/$INCIDENT_ID" \
        -H "Authorization: Bearer $STATUS_API_KEY" \
        -d "{\"message\": \"$message\"}"
}

# Step 4: Resolve
resolve_incident() {
    local resolution="$1"
    log_incident "Resolving: $resolution"
    
    # Clear maintenance mode
    rm -f /var/www/whmcs/.maintenance
    
    # Verify resolution
    php /opt/scripts/verify_resolution.php
    
    # Update status page
    curl -X PUT "https://status.example.com/api/incidents/$INCIDENT_ID" \
        -H "Authorization: Bearer $STATUS_API_KEY" \
        -d "{
            \"status\": \"resolved\",
            \"message\": \"$resolution\"
        }"
    
    # Notify stakeholders
    /opt/scripts/notify_resolution.sh "$INCIDENT_ID" "$resolution"
}

# Step 5: Document
document_incident() {
    log_incident "Documenting incident..."
    
    php /opt/scripts/generate_incident_report.php "$INCIDENT_ID" > \
        /var/log/incidents/$INCIDENT_ID.json
}

# Main execution
case "${2:-}" in
    start)
        acknowledge_incident
        ;;
    assess)
        assess_impact
        ;;
    communicate)
        communicate "${3:-update}"
        ;;
    resolve)
        resolve_incident "${3:-Issue resolved}"
        ;;
    document)
        document_incident
        ;;
    *)
        echo "Usage: $0 <SEV-1|SEV-2|SEV-3|SEV-4> <start|assess|communicate|resolve|document> [message]"
        ;;
esac
```

### Step 3: Incident Communication Templates

```php
<?php
// /opt/scripts/incident_templates.php

class IncidentTemplates {
    
    // Initial notification
    public static function initialNotification(array $incident): string {
        return <<<MESSAGE
## Incident Declared: {$incident['id']}

**Severity:** {$incident['severity']}
**Status:** {$incident['status']}
**Affected Services:** {$incident['affected_services']}
**Time:** {$incident['timestamp']}

**What we know:**
{$incident['description']}

**What we're doing:**
- Investigating the root cause
- Working to restore service
- Will provide updates every 30 minutes

**Impact:**
{$incident['user_impact']}

Next update in 30 minutes.
MESSAGE;
    }
    
    // Status update
    public static function statusUpdate(array $incident): string {
        return <<<MESSAGE
## Status Update - {$incident['id']}

**Time:** {$incident['timestamp']}
**Status:** {$incident['status']}

**Update:**
{$incident['update']}

**Next steps:**
{$incident['next_steps']}

**ETA for resolution:** {$incident['eta']}
MESSAGE;
    }
    
    // Resolution notification
    public static function resolution(array $incident): string {
        return <<<MESSAGE
## Incident Resolved: {$incident['id']}

**Resolved at:** {$incident['resolved_at']}
**Total duration:** {$incident['duration']}

**Summary:**
{$incident['summary']}

**Root cause:**
{$incident['root_cause']}

**Resolution:**
{$incident['resolution']}

**Actions taken:**
{$incident['actions_taken']}

**What we're doing to prevent recurrence:**
{$incident['preventive_actions']}

We apologize for the disruption and appreciate your patience.
MESSAGE;
    }
    
    // Customer-facing status page update
    public static function statusPageUpdate(array $incident): array {
        return [
            'status' => $incident['status'], // operational/degraded/partial-outage/major-outage
            'message' => $incident['public_message'],
            'affected_services' => $incident['affected_services'],
            'scheduled_for' => $incident['scheduled_for'] ?? null
        ];
    }
}
```

### Step 4: Common Incident Runbooks

```markdown
## Runbook: WHMCS Database Connection Issues

### Symptoms
- "Too many connections" errors
- Connection timeouts
- Site loading very slowly

### Diagnosis
1. Check MySQL connection count: `mysql -e "SHOW STATUS LIKE 'threads_connected';"`
2. Check max connections: `mysql -e "SHOW VARIABLES LIKE 'max_connections';"`
3. Check active processes: `mysql -e "SHOW PROCESSLIST;"`
4. Check slow queries: `tail -100 /var/log/mysql/slow.log`

### Mitigation (in order)
1. **Kill idle connections**
   ```sql
   SELECT ID FROM information_schema.processlist 
   WHERE Command = 'Sleep' AND Time > 60;
   -- Then: KILL <ID> for each
   ```

2. **Increase max connections temporarily**
   ```sql
   SET GLOBAL max_connections = 500;
   ```

3. **Restart application servers** to release connections

4. **If DB is under attack**: Block IPs at firewall
   ```bash
   iptables -A INPUT -p tcp --dport 3306 -s <malicious_ip> -j DROP
   ```

### Resolution
- Identify connection leak source
- Fix application code
- Set appropriate max_connections value
- Configure connection pooling

### Prevention
- Monitor connection count
- Set connection timeout limits
- Implement connection pooling
- Regular query optimization

---

## Runbook: WHMCS High Load / Slow Response

### Symptoms
- Site loading slowly (>5s response)
- Users reporting timeout errors
- High load average on servers

### Diagnosis
1. Check load: `uptime`
2. Check PHP-FPM: `curl http://localhost/status`
3. Check MySQL: `SHOW FULL PROCESSLIST`
4. Check disk I/O: `iostat -x 1 5`
5. Check network: `netstat -an | grep :80 | wc -l`

### Mitigation
1. **Scale out** if multiple servers available
   ```bash
   # Add temporary capacity
   kubectl scale deployment whmcs-web --replicas=10
   ```

2. **Enable maintenance mode** to stop new traffic
   ```bash
   touch /var/www/whmcs/.maintenance
   ```

3. **Clear cache** if cache stampede
   ```bash
   rm -rf /var/www/whmcs/storage/templates_c/*
   ```

4. **Optimize database** - kill long-running queries

5. **Restart PHP-FPM** if worker exhaustion
   ```bash
   systemctl restart php8.1-fpm
   ```

### Resolution
- Identify bottleneck (CPU/DB/IO/Network)
- Address root cause
- Load test before removing maintenance mode

---

## Runbook: WHMCS Payment Processing Failure

### Symptoms
- Payments not processing
- "Payment gateway error" messages
- Orders stuck in pending

### Diagnosis
1. Check gateway logs: `/var/www/whmcs/storage/logs/gateways.log`
2. Check gateway status APIs
3. Verify API credentials
4. Check webhook delivery

### Mitigation
1. **Check payment gateway status**
   ```bash
   # Check Stripe status
   curl https://status.stripe.com/api/v2/status
   
   # Check PayPal status
   curl https://www.paypal.com/cgi-bin/webscr?cmd=_notification-Status
   ```

2. **Verify API credentials** in WHMCS configuration

3. **Enable manual payment mode** as fallback
   - Admin > System > Payment Gateways
   - Enable "Invoice" payment method

4. **Process queued payments manually**
   ```sql
   SELECT * FROM tblorders WHERE status = 'Pending' AND payment_method = 'stripe';
   ```

### Resolution
- Fix gateway issue
- Reprocess failed payments
- Verify all payments received

### Prevention
- Monitor gateway status
- Implement payment retry logic
- Regular gateway testing
```

### Step 5: Post-Incident Review Template

```markdown
# Post-Incident Review: [Incident ID]

**Date:** YYYY-MM-DD
**Duration:** X hours Y minutes
**Severity:** SEV-X
**Prepared by:** Name

## Summary
Brief description of what happened.

## Impact
- Users affected: X
- Revenue impact: $X
- Duration of impact: X hours

## Timeline
| Time | Event |
|------|-------|
| HH:MM | Issue detected |
| HH:MM | Incident declared |
| HH:MM | Investigation started |
| HH:MM | Mitigation applied |
| HH:MM | Service restored |
| HH:MM | Incident resolved |

## Root Cause
What actually caused the incident?

## Response Analysis
**What went well:**
-

**What could be improved:**
-

## Action Items

| Action | Owner | Due Date | Status |
|--------|-------|----------|--------|
| | | | |
| | | | |

## Lessons Learned
What did we learn from this incident?

## Related Incidents
Links to related past incidents.
```

## Escalation Matrix

| Severity | Initial Response | Escalation After | Management Notification |
|----------|-----------------|------------------|------------------------|
| SEV-1 | 15 minutes | 30 minutes | Immediate |
| SEV-2 | 1 hour | 2 hours | 1 hour |
| SEV-3 | 4 hours | 8 hours | End of day |
| SEV-4 | Next business day | 48 hours | Weekly |

## Communication Channels

| Channel | Purpose | Audience |
|---------|---------|----------|
| Slack #incident-xxx | Real-time coordination | Response team |
| Status page | Customer updates | All customers |
| Email | Detailed reports | Stakeholders |
| PagerDuty | Critical alerts | On-call team |

## Best Practices

1. **Declare early**: Don't wait for 100% certainty
2. **Communicate often**: Regular updates, even if "no change"
3. **Document everything**: Timestamps, decisions, actions
4. **Blame-free culture**: Focus on learning, not blame
5. **Practice**: Regular incident response drills
6. **Automate**: Reduce manual steps during incidents

## Common Pitfalls

- **Delayed declaration**: Waiting too long to call incident
- **Poor communication**: Not updating stakeholders
- **No documentation**: Forgetting what was done
- **No follow-up**: Not completing action items
- **Blaming individuals**: Hinders honest post-mortems

## Verification Checklist

- [ ] Incident response team defined
- [ ] Escalation matrix documented
- [ ] Communication templates ready
- [ ] Runbooks for common incidents
- [ ] On-call rotation configured
- [ ] Post-incident review process established
- [ ] Action item tracking system in place
- [ ] Regular incident drills scheduled

## Related Documentation

- [WHMCS Disaster Recovery](whmcs-disaster-recovery-workflow.md)
- [WHMCS Monitoring and Alerts](whmcs-monitoring-alerts.md)
- [WHMCS Emergency Response](whmcs-emergency-response.md)
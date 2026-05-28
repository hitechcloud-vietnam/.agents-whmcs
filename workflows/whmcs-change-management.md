# WHMCS Change Management Workflow

## Purpose

Implement change management processes for WHMCS to ensure controlled, tested, and safe deployments while minimizing risk and disruption. This workflow covers change requests, approvals, testing, deployment, and rollback.

## Prerequisites

- Change advisory board (CAB) established
- Change management system (Jira, ServiceNow, etc.)
- Testing environment mirroring production
- Rollback procedures documented
- Communication templates

## Workflow Steps

### Step 1: Change Request Process

```
┌─────────────────────────────────────────────────────────────┐
│                 Change Management Flow                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐    ┌──────────┐    ┌─────────┐    ┌─────────┐  │
│  │ Request │ -> │  Review  │ -> │ Approval│ -> │Implement│  │
│  │  Form   │    │   CAB    │    │   CAB   │    │         │  │
│  └─────────┘    └──────────┘    └─────────┘    └────┬────┘  │
│                                                      │       │
│                         ┌───────────────────────────┘       │
│                         │                                   │
│                    ┌────▼────┐                              │
│                    │ Testing │                              │
│                    └────┬────┘                              │
│                         │                                   │
│            ┌───────────┴───────────┐                       │
│            │                       │                        │
│      ┌─────▼─────┐          ┌──────▼─────┐                 │
│      │  Success  │          │   Failed   │                 │
│      │  Deploy   │          │   Rollback │                 │
│      └───────────┘          └────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Change Request Form

```markdown
## WHMCS Change Request Form

**Change ID:** CR-YYYY-XXXX
**Date:** YYYY-MM-DD
**Requester:** Name / Department

### 1. Change Summary
Brief description of the change:

### 2. Business Justification
Why is this change needed?
- Business benefit:
- Urgency level:
- Impact if not implemented:

### 3. Change Details
| Field | Value |
|-------|-------|
| Change type | [ ] Standard [ ] Normal [ ] Emergency |
| Risk level | [ ] Low [ ] Medium [ ] High [ ] Critical |
| Impact | [ ] None [ ] Minor [ ] Moderate [ ] Major |
| Downtime required | [ ] Yes [ ] No |
| Duration | hours/minutes |

### 4. Technical Details
Affected components:
-

Proposed implementation:
-

Rollback procedure:
-

### 5. Testing Plan
| Test | Expected Result | Pass/Fail |
|------|-----------------|-----------|
| Unit tests | | |
| Integration tests | | |
| UAT | | |
| Performance test | | |

### 6. Implementation Plan
1.
2.
3.

### 7. Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| | | | |

### 8. Approval
| Role | Name | Date | Signature |
|------|------|------|-----------|
| Requester | | | |
| Manager | | | |
| CAB Chair | | | |
| Security | | | |

### 9. Post-Implementation
Scheduled verification: YYYY-MM-DD HH:MM
Actual completion time:
Sign-off:
```

### Step 3: Change Request System

```php
<?php
// /opt/scripts/change_management.php

class ChangeRequest {
    private $db;
    private $pdo;
    
    public function __construct() {
        $this->pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
    }
    
    public function createChangeRequest(array $data): int {
        $stmt = $this->pdo->prepare("
            INSERT INTO change_requests (
                title, description, type, risk_level, 
                impact, downtime_required, duration,
                implementation_plan, rollback_procedure,
                created_by, created_at, status
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, NOW(), 'draft')
        ");
        
        $stmt->execute([
            $data['title'],
            $data['description'],
            $data['type'],
            $data['risk_level'],
            $data['impact'],
            $data['downtime_required'],
            $data['duration'],
            $data['implementation_plan'],
            $data['rollback_procedure'],
            $data['created_by']
        ]);
        
        return $this->pdo->lastInsertId();
    }
    
    public function submitForApproval(int $requestId): bool {
        $stmt = $this->pdo->prepare("
            UPDATE change_requests 
            SET status = 'pending_approval', submitted_at = NOW()
            WHERE id = ?
        ");
        
        return $stmt->execute([$requestId]);
    }
    
    public function approve(int $requestId, string $approver, string $comments): bool {
        $stmt = $this->pdo->prepare("
            UPDATE change_requests 
            SET status = 'approved', 
                approved_by = ?, 
                approved_at = NOW(),
                approval_comments = ?
            WHERE id = ?
        ");
        
        return $stmt->execute([$approver, $comments, $requestId]);
    }
    
    public function getApprovalQueue(): array {
        return $this->pdo->query("
            SELECT * FROM change_requests 
            WHERE status = 'pending_approval' 
            ORDER BY risk_level DESC, created_at ASC
        ")->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function schedule(int $requestId, string $scheduledDate): bool {
        $stmt = $this->pdo->prepare("
            UPDATE change_requests 
            SET scheduled_date = ?, status = 'scheduled'
            WHERE id = ?
        ");
        
        return $stmt->execute([$scheduledDate, $requestId]);
    }
    
    public function implement(int $requestId): bool {
        $stmt = $this->pdo->prepare("
            UPDATE change_requests 
            SET status = 'in_progress', implemented_at = NOW()
            WHERE id = ?
        ");
        
        return $stmt->execute([$requestId]);
    }
    
    public function complete(int $requestId, array $results): bool {
        $stmt = $this->pdo->prepare("
            UPDATE change_requests 
            SET status = 'completed', 
                completed_at = NOW(),
                completion_notes = ?,
                test_results = ?
            WHERE id = ?
        ");
        
        return $stmt->execute([
            $results['notes'],
            json_encode($results['tests']),
            $requestId
        ]);
    }
    
    public function rollback(int $requestId, string $reason): bool {
        $stmt = $this->pdo->prepare("
            UPDATE change_requests 
            SET status = 'rolled_back',
                rolled_back_at = NOW(),
                rollback_reason = ?
            WHERE id = ?
        ");
        
        return $stmt->execute([$reason, $requestId]);
    }
    
    public function getUpcomingChanges(int $days = 7): array {
        return $this->pdo->query("
            SELECT * FROM change_requests 
            WHERE status = 'scheduled' 
            AND scheduled_date BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL $days DAY)
            ORDER BY scheduled_date ASC
        ")->fetchAll(PDO::FETCH_ASSOC);
    }
}
```

### Step 4: Change Advisory Board (CAB) Process

```bash
#!/bin/bash
# /opt/scripts/cab_meeting.sh

# Run weekly CAB meeting
CRON_SCHEDULE="0 10 * * 1"  # Every Monday 10 AM

# List pending changes for review
echo "=== Change Advisory Board Review ==="
echo "Date: $(date)"
echo ""

# High priority changes
echo "### High Priority Changes (SEV-1, SEV-2) ###"
mysql -u whmcs -p -e "
SELECT id, title, risk_level, requester, scheduled_date 
FROM change_requests 
WHERE status = 'pending_approval' 
AND risk_level IN ('high', 'critical')
ORDER BY risk_level DESC, created_at ASC;
"

echo ""
echo "### Normal Changes This Week ###"
mysql -u whmcs -p -e "
SELECT id, title, risk_level, requester, scheduled_date 
FROM change_requests 
WHERE status = 'scheduled' 
AND scheduled_date BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 7 DAY)
ORDER BY scheduled_date ASC;
"

echo ""
echo "### Emergency Changes ###"
mysql -u whmcs -p -e "
SELECT id, title, description, requested_at, approved_by 
FROM change_requests 
WHERE type = 'emergency' 
AND status != 'completed'
ORDER BY requested_at DESC;
"

# Generate CAB minutes
cat > /tmp/cab_minutes_$(date +%Y%m%d).md << 'EOF'
# Change Advisory Board Minutes

**Date:** YYYY-MM-DD
**Chair:** Name
**Attendees:** Name1, Name2, Name3

## Agenda
1. Review pending change requests
2. Approve/reject changes
3. Schedule approved changes
4. Review previous change outcomes
5. Any other business

## Decisions Made
| Change ID | Decision | Reason |
|-----------|----------|--------|
| | | |

## Action Items
| Action | Owner | Due Date |
|--------|-------|----------|
| | | |

## Next Meeting
Date: YYYY-MM-DD
EOF

echo "CAB minutes generated"
```

### Step 5: Change Implementation Workflow

```bash
#!/bin/bash
# /opt/scripts/implement_change.sh

CHANGE_ID=$1
ENVIRONMENT=${2:-staging}

if [ -z "$CHANGE_ID" ]; then
    echo "Usage: $0 <change_id> [staging|production]"
    exit 1
fi

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [CR-$CHANGE_ID] $1"
}

# Verify approval
log "Verifying change approval..."
APPROVED=$(mysql -u whmcs -p -N -e "SELECT COUNT(*) FROM change_requests WHERE id = $CHANGE_ID AND status = 'approved'")
if [ "$APPROVED" -eq 0 ]; then
    log "ERROR: Change not approved or does not exist"
    exit 1
fi

# Create backup before change
log "Creating pre-change backup..."
/opt/scripts/backup_whmcs.sh "pre_change_$CHANGE_ID"

# Download change implementation package
log "Downloading implementation package..."
curl -o /tmp/cr_${CHANGE_ID}_package.tar.gz "https://artifacts.example.com/changes/$CHANGE_ID/package.tar.gz"

# Verify package integrity
log "Verifying package integrity..."
sha256sum /tmp/cr_${CHANGE_ID}_package.tar.gz
# Compare with stored checksum

# Put system in maintenance mode (if needed)
log "Enabling maintenance mode..."
touch /var/www/whmcs/.maintenance

# Stop related services
log "Stopping services..."
systemctl stop php8.1-fpm
systemctl stop cron

# Apply change
log "Applying change..."
tar -xzf /tmp/cr_${CHANGE_ID}_package.tar.gz -C /tmp/
cd /tmp/cr_${CHANGE_ID}/
./install.sh --env=$ENVIRONMENT

# Run verification tests
log "Running verification tests..."
php /opt/scripts/verify_change.php "$CHANGE_ID"

# Test result
if [ $? -eq 0 ]; then
    log "Change verification passed"
    
    # Remove maintenance mode
    rm /var/www/whmcs/.maintenance
    
    # Restart services
    systemctl start php8.1-fpm
    systemctl start cron
    
    # Mark change as complete
    /opt/scripts/complete_change.php "$CHANGE_ID"
    
    log "Change implementation completed successfully"
else
    log "Change verification failed - initiating rollback"
    
    # Rollback
    /opt/scripts/rollback_change.sh "$CHANGE_ID"
    
    # Mark as rolled back
    /opt/scripts/rollback_change.php "$CHANGE_ID"
    
    exit 1
fi

# Cleanup
rm -rf /tmp/cr_${CHANGE_ID}*
```

### Step 6: Emergency Change Process

```bash
#!/bin/bash
# /opt/scripts/emergency_change.sh

set -euo pipefail

CHANGE_ID=$1
DESCRIPTION=$2

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [EMERGENCY-$CHANGE_ID] $1"
}

# Create emergency change record
log "Creating emergency change record..."
mysql -u whmcs -p -e "
INSERT INTO change_requests (
    title, description, type, risk_level, 
    status, created_by, approved_by, approved_at
) VALUES (
    'EMERGENCY: $DESCRIPTION',
    '$DESCRIPTION',
    'emergency',
    'critical',
    'approved',  -- Auto-approve for emergencies
    'system',
    'auto_approved',
    NOW()
)
"

# Notify stakeholders
log "Notifying stakeholders..."
curl -X POST "$SLACK_WEBHOOK" -H 'Content-Type: application/json' -d '{
    "text": "Emergency change initiated: ' $DESCRIPTION '",
    "channel": "#changes"
}'

# Implement immediately
log "Implementing emergency change..."
/opt/scripts/implement_change.sh "$CHANGE_ID" "production"

log "Emergency change completed"
```

## Change Categories and SLAs

| Type | Definition | Risk Level | CAB Approval | Implementation Window |
|------|------------|------------|--------------|---------------------|
| Standard | Pre-approved, low risk | Low | Expedited | Any time |
| Normal | Standard change | Medium | Required | Business hours |
| Emergency | Urgent, time-critical | High/Critical | Retroactive | Immediate |

## Change Review Checklist

```
Pre-implementation:
□ Change approved by CAB
□ Rollback procedure documented
□ Backup completed
□ Testing completed
□ Implementation window scheduled
□ Stakeholders notified

Post-implementation:
□ Verification tests passed
□ Functionality confirmed
□ Performance acceptable
□ No errors in logs
□ Post-implementation review scheduled
```

## Best Practices

1. **Document everything**: All changes must have approval records
2. **Test thoroughly**: Test in staging before production
3. **Rollback ready**: Always have rollback plan
4. **Communicate**: Keep stakeholders informed
5. **Review**: Conduct post-implementation reviews
6. **Automate**: Use infrastructure as code

## Common Pitfalls

- **Skip approvals**: Implementing without authorization
- **Incomplete testing**: Not testing all scenarios
- **No rollback plan**: Not preparing for failure
- **Poor documentation**: Missing implementation details
- **Skip communication**: Not notifying stakeholders

## Verification Checklist

- [ ] Change request process documented
- [ ] CAB established with defined members
- [ ] Approval workflow implemented
- [ ] Rollback procedures documented
- [ ] Testing procedures defined
- [ ] Emergency change process established
- [ ] Change log maintained
- [ ] Post-implementation reviews conducted

## Related Documentation

- [WHMCS Deployment Best Practices](whmcs-deployment-best-practices.md)
- [WHMCS CI/CD Workflow](whmcs-ci-cd-workflow.md)
- [WHMCS Disaster Recovery](whmcs-disaster-recovery-workflow.md)
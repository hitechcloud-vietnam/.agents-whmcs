# WHMCS Firewall Setup Workflow

## Description
Configure firewall for WHMCS server security with proper port management and rules.

## Prerequisites
- Root/sudo access
- UFW, iptables, or firewalld installed
- Knowledge of required ports

## Security Principles
- Default deny all incoming
- Allow only necessary ports
- Restrict admin access by IP
- Log suspicious activity
- Regular rule review

## Steps

### Step 1: Identify Required Ports
```bash
# WHMCS Core Requirements
# Port 80/443 - HTTP/HTTPS for web traffic
# Port 22 - SSH access
# Port 3306 - MySQL (if local, block external)
# Port 6379 - Redis (if used, block external)
# Port 25 - SMTP (for local mail)

# Optional
# Port 8080 - Alternative web port
# Port 8443 - Admin SSL alternative
```

### Step 2: Configure UFW (Ubuntu/Debian)
```bash
# Install UFW if not present
apt update && apt install -y ufw

# Set default policies
ufw default deny incoming
ufw default allow outgoing

# Allow SSH (LIMIT prevents brute force)
ufw limit 22/tcp comment 'SSH Access'

# Allow HTTP and HTTPS
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'

# Allow admin IP (REPLACE with your IP)
ufw allow from 203.0.113.50 to any port 22 comment 'Admin SSH'
ufw allow from 203.0.113.50 to any port 443 comment 'Admin Panel'

# Allow monitoring ports (optional)
ufw allow 9090/tcp comment 'Prometheus Metrics'

# Enable UFW
ufw --force enable

# Check status
ufw status verbose
```

### Step 3: Configure iptables (Advanced)
```bash
# Create comprehensive iptables script
cat > /usr/local/bin/whmcs-firewall.sh << 'EOF'
#!/bin/bash

# Flush existing rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# Set default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# SSH with rate limiting
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --set
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Admin IP restrictions
ADMIN_IPS="203.0.113.50 198.51.100.100"
for ip in $ADMIN_IPS; do
    iptables -A INPUT -p tcp -s $ip --dport 443 -j ACCEPT
done

# MySQL local only
iptables -A INPUT -p tcp --dport 3306 -s 127.0.0.1 -j ACCEPT
iptables -A INPUT -p tcp --dport 3306 -s 10.0.0.0/8 -j ACCEPT

# Redis local only
iptables -A INPUT -p tcp --dport 6379 -s 127.0.0.1 -j ACCEPT
iptables -A INPUT -p tcp --dport 6379 -s 10.0.0.0/8 -j ACCEPT

# Block common attacks
# Block null packets
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP

# Block XMAS packets
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP

# Block SYN flood
iptables -A INPUT -p tcp --syn -m limit --limit 100/second --limit-burst 200 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# Log dropped packets
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables-dropped: "

echo "Firewall rules applied successfully"
EOF

chmod +x /usr/local/bin/whmcs-firewall.sh
/usr/local/bin/whmcs-firewall.sh
```

### Step 4: Configure firewalld (CentOS/RHEL)
```bash
# Install firewalld
yum install -y firewalld
systemctl start firewalld
systemctl enable firewalld

# Configure zones
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --zone=public --add-service=ssh --permanent

# Rich rules for admin access
firewall-cmd --permanent --zone=public \
    --add-source=203.0.113.50/32
firewall-cmd --permanent --zone=public \
    --add-port=443/tcp

# Allow MySQL from internal network only
firewall-cmd --permanent --zone=trusted --add-source=10.0.0.0/8
firewall-cmd --permanent --zone=trusted --add-service=mysql

# Apply changes
firewall-cmd --reload
firewall-cmd --list-all
```

### Step 5: Fail2Ban for Brute Force Protection
```bash
# Install Fail2Ban
apt install -y fail2ban

# Configure WHMCS protection
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5
destemail = admin@example.com
sender = fail2ban@example.com
action = %(action_mwl)s

[whmcs-admin]
enabled = true
port = 443
protocol = tcp
filter = whmcs-admin
logpath = /var/www/whmcs/logs/*.log
maxretry = 5
bantime = 86400

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
EOF

# Create WHMCS filter
cat > /etc/fail2ban/filter.d/whmcs-admin.conf << 'EOF'
[Definition]
failregex = ^.*\[WHMCS\] Failed login attempt.*clientarea\.php.*IP: <HOST>
ignoreregex =
EOF

systemctl restart fail2ban
systemctl enable fail2ban
```

### Step 6: CloudFlare Protection (if used)
```bash
# If using CloudFlare CDN
# Configure real IP logging

# Add to nginx config
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 104.16.0.0/13;
set_real_ip_from 104.24.0.0/14;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 131.0.72.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 162.158.0.0/15;
set_real_ip_from 172.64.0.0/13;
set_real_ip_from 192.168.127.0/21;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
real_ip_header X-Forwarded-For;

# Block non-CloudFlare IPs accessing directly
# Only allow CloudFlare IPs
```

### Step 7: Implement Geo-Blocking (Optional)
```bash
# Install geoip tools
apt install -y geoip-database

# Block countries (example blocks China and Russia)
iptables -A INPUT -m geoip --src-cc CN -j DROP
iptables -A INPUT -m geoip --src-cc RU -j DROP

# Or use country-specific IP lists
# Download and apply
```

### Step 8: Configure Intrusion Detection
```bash
# Install OSSEC or similar
apt install -y ossec-hids

# Or use fail2ban with custom rules
# Check /etc/fail2ban/jail.local
```

### Step 9: Test Firewall Rules
```bash
# Test from external IP
nmap -sS -p 22,80,443 your-server-ip

# Expected: Only 443 (or 22 from admin IP) should show open

# Test for open ports
netstat -tuln | grep LISTEN

# Check UFW logs
tail -f /var/log/ufw.log

# Test SSH rate limiting
# Try 4 failed connections, 5th should be blocked
```

### Step 10: Monitor and Alert
```bash
# Create monitoring script
cat > /usr/local/bin/firewall-monitor.sh << 'EOF'
#!/bin/bash

LOG_FILE="/var/log/firewall_alerts.log"
ALERT_EMAIL="admin@example.com"

# Check for suspicious activity
SUSPICIOUS=$(grep "iptables-dropped" /var/log/kern.log | tail -20 | wc -l)

if [ $SUSPICIOUS -gt 50 ]; then
    echo "High number of dropped packets detected: $SUSPICIOUS" | \
        mail -s "Firewall Alert: High Packet Drops" $ALERT_EMAIL
fi

# Report blocked countries
grep "DROP" /var/log/ufw.log | awk '{print $NF}' | sort | uniq -c | \
    sort -rn | head -10 >> $LOG_FILE
EOF

chmod +x /usr/local/bin/firewall-monitor.sh
echo "*/30 * * * * /usr/local/bin/firewall-monitor.sh" >> /etc/crontab
```

## Security Checklist
- [ ] Default deny policy enabled
- [ ] SSH restricted to specific IPs
- [ ] SSH rate limiting enabled
- [ ] Only necessary ports open
- [ ] MySQL restricted to localhost/internal
- [ ] Redis restricted (if used)
- [ ] Fail2Ban configured
- [ ] CloudFlare real IP configured
- [ ] Logs being monitored
- [ ] Regular rule audits scheduled

## Tags
- firewall
- security
- iptables
- ufw
- hardening
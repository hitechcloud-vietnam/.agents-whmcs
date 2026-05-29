---
name: whmcs-system-optimize
description: Optimize WHMCS system performance
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, optimize, performance, tuning]
---

# WHMCS Performance Optimization Workflow

## Purpose
Step-by-step guide for optimizing WHMCS performance.

## Prerequisites
- WHMCS admin access
- Performance issues identified
- Optimization tools available

## Step 1: Analyze Performance
- Check System Health
- Review error logs
- Analyze slow queries
- Check page load times
- Identify bottlenecks

## Step 2: Enable Caching
- Navigate to Configuration > System
- Enable caching:
  - OPCache
  - Memcached
  - Redis
- Configure cache settings
- Test cache functionality

## Step 3: Optimize Database
- Enable query caching
- Optimize tables
- Add indexes
- Clean old data
- Enable slow query log

## Step 4: Configure PHP
- Update PHP settings:
  - memory_limit
  - max_execution_time
  - opcache settings
- Enable JIT if available
- Configure opcache

## Step 5: Optimize Assets
- Enable minification
- Configure compression
- Optimize images
- Enable browser caching
- Use CDN for assets

## Step 6: Configure Web Server
- Enable Gzip compression
- Set up caching headers
- Configure keep-alive
- Enable HTTP/2
- Optimize SSL

## Step 7: Monitor Performance
- Test after changes
- Monitor metrics
- Track improvements
- Document changes
- Schedule regular checks

## Optimization Areas
- PHP Configuration
- Database Optimization
- Caching
- Asset Optimization
- Server Configuration

## Related Workflows
- whmcs-performance-optimization
- whmcs-system-health
- whmcs-cache-management
---
name: whmcs-system-cache
description: Manage system cache in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, cache, performance, system]
---

# WHMCS System Cache Management Workflow

## Purpose
Step-by-step guide for managing WHMCS cache.

## Prerequisites
- WHMCS admin access
- Cache configuration identified
- Performance optimization plan

## Step 1: Access Cache Settings
- Navigate to Configuration > System
- Find Cache Settings
- View cache configuration

## Step 2: Configure Cache Type
Select cache backend:
- File-based (default)
- Memcached
- Redis
- APC/APCu
- Custom

## Step 3: Set Cache Options
- Configure cache TTL
- Set cache size limits
- Enable/disable specific caches
- Set cache warming

## Step 4: Clear Cache
Manual cache clearing:
- Clear all cache
- Clear template cache
- Clear data cache
- Clear minified assets
- Schedule automatic clearing

## Step 5: Clear via Admin
- Navigate to Utilities > Cache
- Click "Clear Cache"
- Select what to clear
- Confirm action
- Verify cache cleared

## Step 6: Configure Cache Rules
- Set per-module cache
- Configure cache exemption
- Set cache for different areas
- Configure cache invalidation

## Step 7: Monitor Cache Performance
- Check cache hit rate
- Monitor cache size
- Review cache storage
- Optimize cache settings
- Document cache config

## Cache Areas
- Template Cache
- Data Cache
- Minified Assets
- API Cache
- Session Cache

## Related Workflows
- whmcs-system-optimize
- whmcs-cache-management
- whmcs-memory-cache
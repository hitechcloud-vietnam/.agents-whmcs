# WHMCS npm Package Workflow

## Overview
This workflow guides you through creating an npm package for WHMCS JavaScript components.

## Prerequisites
- Node.js 18+
- npm account (for publishing)

## Step-by-Step Guide

### Step 1: Initialize Package
```bash
mkdir whmcs-module-frontend
cd whmcs-module-frontend
npm init -y
```

### Step 2: Configure package.json
```json
{
    "name": "@vendor/whmcs-module",
    "version": "2.0.0",
    "description": "WHMCS Module Frontend Components",
    "main": "dist/index.js",
    "module": "dist/index.mjs",
    "types": "dist/index.d.ts",
    "exports": {
        ".": {
            "import": "./dist/index.mjs",
            "require": "./dist/index.js"
        }
    },
    "scripts": {
        "build": "vite build",
        "dev": "vite build --watch",
        "test": "vitest",
        "lint": "eslint src"
    },
    "peerDependencies": {
        "whmcs": "^8.0"
    },
    "devDependencies": {
        "vite": "^4.0",
        "vitest": "^0.25"
    }
}
```

### Step 3: Create Components
```javascript
// src/widget.js
export class DashboardWidget {
    constructor(container) {
        this.container = container;
    }

    render() {
        this.container.innerHTML = `
            <div class="whmcs-widget">
                <h3>Statistics</h3>
                <div id="stats-content">Loading...</div>
            </div>
        `;
        this.loadData();
    }

    async loadData() {
        const response = await fetch('/modules/addons/yourmodule/api/stats.php');
        const data = await response.json();
        this.updateStats(data);
    }

    updateStats(data) {
        document.getElementById('stats-content').innerHTML = `
            <p>Total: ${data.total}</p>
        `;
    }
}
```

### Step 4: Build
```bash
# Build for production
npm run build

# Watch mode
npm run dev
```

### Step 5: Publish
```bash
# Login to npm
npm login

# Publish
npm publish --access public
```

## npm Package Checklist

### Package
- [ ] package.json configured
- [ ] Build script working
- [ ] TypeScript types included
- [ ] Peer dependencies correct

### Publishing
- [ ] npm account verified
- [ ] Version bumped
- [ ] Package published
- [ ] Documentation updated

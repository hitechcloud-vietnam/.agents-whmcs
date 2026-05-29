# WHMCS Update Push Workflow

## Overview
This workflow guides you through pushing updates to WHMCS modules.

## Prerequisites
- WHMCS Marketplace account
- Update server access
- Version control

## Step-by-Step Guide

### Step 1: Prepare Update Package
```bash
#!/bin/bash
# prepare-update.sh

VERSION="${1}"
MODULE_NAME="yourmodule"
OUTPUT_DIR="/tmp/whmcs-update"

mkdir -p "$OUTPUT_DIR"

# Copy module files
cp -r "/var/www/module/$MODULE_NAME" "$OUTPUT_DIR/"

# Remove unnecessary files
rm -rf "$OUTPUT_DIR/$MODULE_NAME/.git"
rm -rf "$OUTPUT_DIR/$MODULE_NAME/node_modules"
rm -rf "$OUTPUT_DIR/$MODULE_NAME/tests"

# Create archive
cd "$OUTPUT_DIR"
tar -czf "{$MODULE_NAME}-{$VERSION}.tar.gz" "$MODULE_NAME"

# Create checksum
sha256sum "{$MODULE_NAME}-{$VERSION}.tar.gz" > "{$MODULE_NAME}-{$VERSION}.tar.gz.sha256"

echo "Update package ready: {$OUTPUT_DIR}/{$MODULE_NAME}-{$VERSION}.tar.gz"
```

### Step 2: Upload to Update Server
```bash
#!/bin/bash
# upload-update.sh

VERSION="${1}"
MODULE_NAME="yourmodule"
SERVER="updates.example.com"
API_KEY="your_api_key"

# Upload package
curl -X POST \
    -F "version=$VERSION" \
    -F "module=$MODULE_NAME" \
    -F "package=@/tmp/whmcs-update/{$MODULE_NAME}-{$VERSION}.tar.gz" \
    -F "checksum=@/tmp/whmcs-update/{$MODULE_NAME}-{$VERSION}.tar.gz.sha256" \
    -H "Authorization: Bearer $API_KEY" \
    "https://$SERVER/api/upload"

echo "Update uploaded for version $VERSION"
```

### Step 3: WHMCS Addon Update Hook
```php
// Check for updates
function yourmodule_check_update()
{
    $currentVersion = '1.5.0';
    
    $response = file_get_contents(
        'https://updates.example.com/api/check?' . http_build_query([
            'module' => 'yourmodule',
            'version' => $currentVersion,
        ])
    );
    
    $data = json_decode($response, true);
    
    if (version_compare($data['latest_version'], $currentVersion, '>')) {
        return [
            'update' => true,
            'version' => $data['latest_version'],
            'url' => $data['download_url'],
            'changelog' => $data['changelog'],
        ];
    }
    
    return ['update' => false];
}
```

### Step 4: Execute Update Push
```bash
# Prepare
./prepare-update.sh 2.0.0

# Upload
./upload-update.sh 2.0.0

# Verify
curl "https://updates.example.com/api/check?module=yourmodule&version=2.0.0"
```

## Update Push Checklist

### Package
- [ ] Version correct
- [ ] Files complete
- [ ] Checksum generated
- [ ] No unnecessary files

### Upload
- [ ] Server accessible
- [ ] API authenticated
- [ ] Upload verified

### Distribution
- [ ] WHMCS can detect update
- [ ] Download works
- [ ] Installation works

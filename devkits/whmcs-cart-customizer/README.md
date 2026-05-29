# WHMCS Cart Customizer Devkit

## Overview
This addon module customizes the WHMCS shopping cart experience with enhanced styling, layout options, and user features.

## WHMCS Version
- **Supported**: 7.8.0+
- **Minimum**: 7.0.0

## Features
- Custom cart theme support
- Product layout options (grid/list)
- Add-to-cart animations
- Mini-cart widget
- Cart summary customization
- AJAX cart updates

## File Structure
```
whmcs-cart-customizer/
├── DEVKIT.md
├── README.md
├── hooks/
│   └── CartCustomizerHooks.php
├── templates/
│   └── cart-customizer/
│       ├── view.tpl
│       ├── mini-cart.tpl
│       └── styles.css
├── assets/
│   ├── css/customizer.css
│   └── js/customizer.js
└── includes/
    └── CartCustomizer.php
```

## Installation

### Step 1: Upload Files
Upload the module folder to your WHMCS `/modules/addons/` directory.

### Step 2: Activate Module
1. Go to **Setup > Addon Modules**
2. Find "Cart Customizer" and click **Activate**
3. Configure permissions for admin roles

### Step 3: Configure Settings
1. Click **Configure** for the module
2. Set cart layout preferences
3. Configure styling options
4. Save settings

## Usage
The module automatically enhances the default WHMCS cart experience.
Navigate to your store cart page to see the customizations.
# WHMCS Knowledge Base Setup Workflow

## Purpose
Create and configure knowledge base for self-service support

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Knowledge Base

Navigate to: Setup > Support > Knowledgebase

## Step 2: Enable Knowledge Base

```
Enable Knowledgebase: Yes
Show in Client Area: Yes
Allow Client Submissions: Yes
Require Login to View: No
```

## Step 3: Create Categories

Navigate to: Setup > Support > Knowledgebase > Categories

### Create Main Categories
1. Click "Add Category"
2. Configure:
   ```
   Category Name: Getting Started
   Parent Category: None
   Description: Guides for new customers
   Display Order: 1
   ```
3. Save

### Subcategories
```
Parent: Getting Started
Name: Account Management
Description: Managing your account
```

### Recommended Categories
- Getting Started
- Account & Billing
- Technical Support
- Domain Management
- SSL Certificates
- Troubleshooting

## Step 4: Create Articles

Navigate to: Setup > Support > Knowledgebase > Articles

### Create Article
1. Click "Add Article"
2. Configure:
   ```
   Title: How to Reset Your Password
   Category: Getting Started
   Article Summary: Quick guide to password reset
   ```
3. Add content using WYSIWYG editor

### Article Template
```html
<h2>Overview</h2>
<p>Brief description of the topic.</p>

<h2>Step-by-Step Guide</h2>
<ol>
    <li>First step</li>
    <li>Second step</li>
    <li>Third step</li>
</ol>

<h2>Troubleshooting</h2>
<p>If you're still having issues, contact support.</p>

<h2>Related Articles</h2>
<ul>
    <li>Article 1</li>
    <li>Article 2</li>
</ul>
```

## Step 5: Add Helpful Votes

Navigate to: Setup > Support > Knowledgebase > Article Settings

```
Show Helpful/Not Helpful: Yes
Show View Count: Yes
Allow Comments: No
Require Approval for Comments: Yes
```

## Step 6: Configure Article Display

```
Article Template: Standard
Show Breadcrumbs: Yes
Show Print Button: Yes
Show PDF Export: No
Related Articles: Yes (3 articles)
```

## Step 7: Set Up Client Suggestions

Navigate to: Setup > Support > Knowledgebase > Suggestions

```
Enable Search Suggestions: Yes
Show Related on Ticket Submit: Yes
Minimum Match Score: 70%
```

## Step 8: Configure Public Categories

Navigate to: Setup > Support > Knowledgebase > Categories

1. Edit category
2. Toggle "Public" for visible categories

## Step 9: Add Media to Articles

### Upload Images
Navigate to: Setup > Support > Knowledgebase > Media Library

Upload:
- Tutorial images
- Screenshots
- Diagrams

### Embed Media
```html
<img src="{$kb_media_url}screenshot.png" alt="Screenshot">
```

## Step 10: Set Up Article Tags

Navigate to: Setup > Support > Knowledgebase > Tags

```
Tag Cloud: Enabled
Popular Tags: 20
```

## Step 11: Configure Access Control

Navigate to: Setup > Support > Knowledgebase > Access Control

```
Public Articles: All visitors
Protected Articles: Logged-in clients only
Hidden Articles: Admin only
```

## Step 12: Review Article Statistics

Navigate to: Setup > Support > Knowledgebase > Statistics

Monitor:
- Most viewed articles
- Helpful votes
- Search terms
- Missing articles

## Knowledge Base Checklist

- [ ] Knowledge base enabled
- [ ] Categories created
- [ ] Articles written
- [ ] Helpful votes enabled
- [ ] Search suggestions configured
- [ ] Media uploaded
- [ ] Tags set up
- [ ] Access control configured
- [ ] Statistics monitored

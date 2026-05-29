# WHMCS Knowledge Base System Devkit

## Overview
A comprehensive knowledge base system for WHMCS with article management, categories, search functionality, voting system, and SEO optimization.

## Features
- Article management
- Category hierarchy
- Full-text search
- Tagging system
- Article versioning
- Voting/rating system
- Views tracking
- Related articles
- SEO optimization
- Multi-language support
- Private articles
- Article templates

## WHMCS Integration Points
- Module: addon/KnowledgeBase
- Hook: ClientAreaPage
- Hook: SearchResults
- API integration

## Database Schema
```sql
CREATE TABLE mod_kb_categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    parent_id INT NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    icon VARCHAR(50),
    sort_order INT DEFAULT 0,
    is_published TINYINT(1) DEFAULT 1,
    articles_count INT DEFAULT 0,
    INDEX idx_parent (parent_id)
);

CREATE TABLE mod_kb_articles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    category_id INT NOT NULL,
    title VARCHAR(500) NOT NULL,
    slug VARCHAR(500) NOT NULL,
    content TEXT NOT NULL,
    excerpt TEXT,
    author_id INT,
    status ENUM('draft', 'published', 'archived') DEFAULT 'draft',
    views INT DEFAULT 0,
    helpful_yes INT DEFAULT 0,
    helpful_no INT DEFAULT 0,
    meta_title VARCHAR(255),
    meta_description TEXT,
    featured_image VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_category (category_id),
    INDEX idx_status (status)
);

CREATE TABLE mod_kb_tags (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    slug VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE mod_kb_article_tags (
    article_id INT NOT NULL,
    tag_id INT NOT NULL,
    PRIMARY KEY (article_id, tag_id)
);
```

## API Endpoints
- GET /api/kb/articles - List articles
- GET /api/kb/articles/:slug - Get article
- POST /api/kb/articles - Create article
- PUT /api/kb/articles/:id - Update article
- GET /api/kb/search - Search articles

## Testing Checklist
- [ ] Article CRUD
- [ ] Category navigation
- [ ] Search functionality
- [ ] Voting system
- [ ] SEO output

## Dependencies
- WHMCS 8.x+
- PHP 8.1+

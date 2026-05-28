# WHMCS Docker Deployment Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for deploying WHMCS in Docker environments.

## When to Use

- Setting up development environments
- Creating staging/production deployments
- Containerizing WHMCS modules

## Docker Deployment Patterns

### Dockerfile

```dockerfile
# WHMCS Container Dockerfile
FROM php:8.2-apache

# Install required extensions
RUN apt-get update && apt-get install -y \
    libzip-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    curl \
    libcurl4-openssl-dev \
    && docker-php-ext-install \
        pdo \
        pdo_mysql \
        zip \
        curl \
        mbstring \
        xml \
        gd \
        intl

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Enable Apache modules
RUN a2enmod headers rewrite

# Set document root
ENV APACHE_DOCUMENT_ROOT=/var/www/html

# Configure PHP
RUN echo "memory_limit=256M" >> /usr/local/etc/php/conf.d/whmcs.ini && \
    echo "upload_max_filesize=50M" >> /usr/local/etc/php/conf.d/whmcs.ini && \
    echo "post_max_size=50M" >> /usr/local/etc/php/conf.d/whmcs.ini

# Copy WHMCS files
COPY whmcs/ /var/www/html/

# Set permissions
RUN chown -R www-data:www-data /var/www/html && \
    chmod -R 755 /var/www/html && \
    chmod -R 775 /var/www/html/attachments /var/www/html/templates_c /var/www/html/vendor

# Expose port
EXPOSE 80

CMD ["apache2-foreground"]
```

### Docker Compose

```yaml
version: '3.8'

services:
  whmcs:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: whmcs
    ports:
      - "8080:80"
    environment:
      - WHMCS_DB_HOST=mysql
      - WHMCS_DB_NAME=whmcs
      - WHMCS_DB_USER=whmcs_user
      - WHMCS_DB_PASS=${DB_PASSWORD}
      - WHMCS_LICENSE_KEY=${LICENSE_KEY}
    volumes:
      - whmcs_data:/var/www/html
      - ./modules:/var/www/html/modules
      - ./custom_config.php:/var/www/html/custom_config.php
    depends_on:
      mysql:
        condition: service_healthy
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    container_name: whmcs_mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=whmcs
      - MYSQL_USER=whmcs_user
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: whmcs_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  whmcs_data:
  mysql_data:
  redis_data:
```

### Environment Configuration

```bash
# .env file
MYSQL_ROOT_PASSWORD=strong_root_password_here
DB_PASSWORD=strong_db_password_here
LICENSE_KEY=your-whmcs-license-key
WHMCS_ADMIN_USER=admin
WHMCS_ADMIN_PASSWORD=admin_password_here
```

### Development Setup

```bash
# Start development environment
docker-compose up -d

# Watch logs
docker-compose logs -f whmcs

# Access container shell
docker exec -it whmcs bash

# Run WHMCS CLI commands
docker exec -it whmcs php artisan update-db

# Run module tests
docker exec -it whmcs php ./modules/addons/yourmodule/tests/run.php
```

### Module Testing in Docker

```dockerfile
# modules/test.Dockerfile
FROM php:8.2-cli

RUN apt-get update && apt-get install -y \
    libzip-dev \
    libonig-dev \
    unzip \
    && docker-php-ext-install pdo pdo_mysql mbstring

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /app
COPY . /app

RUN composer install --no-interaction

CMD ["php", "vendor/bin/phpunit"]
```

```yaml
# Testing service in docker-compose.yml
  testing:
    build:
      context: ./modules/yourmodule
      dockerfile: Dockerfile.test
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      - WHMCS_DB_HOST=mysql
      - WHMCS_DB_NAME=whmcs_test
      - WHMCS_DB_USER=whmcs_user
      - WHMCS_DB_PASSWORD=${DB_PASSWORD}
    command: ["php", "vendor/bin/phpunit"]
```

### Production Deployment

```dockerfile
# Production-optimized Dockerfile
FROM php:8.2-fpm-alpine

RUN apk add --no-cache \
    libzip-dev \
    libxml2-dev \
    onig-dev \
    curl \
    && docker-php-ext-install \
        pdo \
        pdo_mysql \
        zip \
        mbstring \
        xml \
        gd \
        intl \
    && apk add --no-cache fcgi

COPY whmcs/ /var/www/html/

RUN chown -R nginx:nginx /var/www/html && \
    mkdir -p /var/www/html/attachments /var/www/html/templates_c && \
    chown -R nginx:nginx /var/www/html/attachments /var/www/html/templates_c && \
    rm -rf /var/www/html/install

EXPOSE 9000

CMD ["php-fpm"]
```

### nginx Configuration

```nginx
server {
    listen 80;
    server_name whmcs.example.com;
    root /var/www/html;
    index index.php;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass whmcs:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }

    location ^~ /attachments/ {
        deny all;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
        expires max;
        log_not_found off;
    }
}
```

### Health Check

```bash
#!/bin/bash
# healthcheck.sh
curl -f http://localhost/health.php || exit 1
```

---

**Related Skills:**
- whmcs-deployment
- whmcs-security-hardening
- whmcs-testing-qa

# WHMCS Docker Container Workflow

## Overview
This workflow guides you through containerizing a WHMCS module for development and deployment.

## Prerequisites
- Docker installed
- Docker Compose (optional)

## Step-by-Step Guide

### Step 1: Create Dockerfile
```dockerfile
# Dockerfile
FROM php:8.1-apache

# Install dependencies
RUN apt-get update && apt-get install -y \
    libzip-dev \
    libpng-dev \
    libcurl4-openssl-dev \
    libxml2-dev \
    && docker-php-ext-install \
    pdo \
    pdo_mysql \
    zip \
    gd \
    curl

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Enable mod_rewrite
RUN a2enmod rewrite

# Copy WHMCS
COPY whmcs/ /var/www/html/

# Set permissions
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html

# Copy module
COPY modules/ /var/www/html/modules/addons/

WORKDIR /var/www/html

EXPOSE 80
```

### Step 2: Create docker-compose.yml
```yaml
version: '3.8'

services:
  whmcs:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80"
    environment:
      - DB_HOST=mysql
      - DB_NAME=whmcs
      - DB_USER=whmcs
      - DB_PASS=whmcs
    volumes:
      - ./storage:/var/www/html/storage
      - ./config.php:/var/www/html/configuration.php
    depends_on:
      - mysql
    networks:
      - whmcs-network

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: whmcs
      MYSQL_USER: whmcs
      MYSQL_PASSWORD: whmcs
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - whmcs-network

networks:
  whmcs-network:

volumes:
  mysql_data:
```

### Step 3: Module Development Dockerfile
```dockerfile
# Dockerfile.dev
FROM php:8.1-cli

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    libzip-dev \
    libpng-dev \
    nodejs \
    npm \
    && docker-php-ext-install \
    pdo \
    pdo_mysql

WORKDIR /app
CMD ["bash"]
```

### Step 4: Build and Run
```bash
# Build image
docker build -t whmcs-module:latest .

# Run container
docker run -p 8080:80 whmcs-module:latest

# Run with Docker Compose
docker-compose up -d
```

### Step 5: Development Workflow
```bash
# Start development environment
docker-compose up -d

# Run tests inside container
docker-compose exec whmcs php /var/www/html/modules/addons/yourmodule/vendor/bin/phpunit

# View logs
docker-compose logs -f whmcs
```

## Docker Container Checklist

### Dockerfile
- [ ] Base image correct
- [ ] Extensions installed
- [ ] Permissions set
- [ ] Module copied

### Compose
- [ ] Services defined
- [ ] Networks configured
- [ ] Volumes mapped
- [ ] Environment set

### Development
- [ ] Containers start
- [ ] Module works
- [ ] Tests pass
- [ ] Volumes persist

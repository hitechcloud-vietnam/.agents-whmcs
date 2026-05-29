# WHMCS Docker Setup

## Skill Description
Set up Docker development environment for WHMCS modules with containerized services and reproducible development environments.

## Prerequisites
- Docker installed
- Docker Compose
- Basic container knowledge

## Step-by-Step Implementation

### 1. Dockerfile
```dockerfile
# Dockerfile

FROM php:8.1-fpm

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    unzip \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    libmariadb-dev \
    zip \
    curl \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www/html

# Copy application
COPY . /var/www/html

# Set permissions
RUN chown -R www-data:www-data /var/www/html

# Expose port
EXPOSE 9000

CMD ["php-fpm"]
```

### 2. Docker Compose
```yaml
# docker-compose.yml

version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: whmcs-app
    volumes:
      - .:/var/www/html
    depends_on:
      - mysql
      - redis
    environment:
      - APP_ENV=local
      - DB_HOST=mysql
      - DB_DATABASE=whmcs
      - DB_USERNAME=whmcs
      - DB_PASSWORD=secret

  mysql:
    image: mysql:8.0
    container_name: whmcs-mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: whmcs
      MYSQL_USER: whmcs
      MYSQL_PASSWORD: secret
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    container_name: whmcs-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  nginx:
    image: nginx:alpine
    container_name: whmcs-nginx
    ports:
      - "8080:80"
    volumes:
      - .:/var/www/html
      - ./docker/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

volumes:
  mysql_data:
  redis_data:
```

### 3. Nginx Configuration
```nginx
# docker/nginx.conf

server {
    listen 80;
    server_name localhost;
    root /var/www/html;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### 4. Makefile
```makefile
# Makefile

.PHONY: up down build logs shell

up:
	docker-compose up -d

down:
	docker-compose down

build:
	docker-compose build

logs:
	docker-compose logs -f

shell:
	docker-compose exec app bash

test:
	docker-compose exec app ./vendor/bin/phpunit

migrate:
	docker-compose exec app php artisan migrate

composer-install:
	docker-compose exec app composer install
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Port conflicts | Change port mappings |
| Volume permissions | Set correct ownership |
| Slow builds | Use build cache |

## Security Considerations

1. **Don't expose ports** - Use in development only
2. **Secure passwords** - Use environment variables
3. **Remove debug tools** - Not for production

## Testing Checklist

- [ ] Test container startup
- [ ] Test database connection
- [ ] Test application access
- [ ] Test file permissions

## Reference Links

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)

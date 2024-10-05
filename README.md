# Laravel Docker Setup with Supervisor for Long-Running Processes

This guide explains how to set up a Laravel application with Docker and Supervisor to handle long-running processes.



## Dockerfile

Create a `Dockerfile` in your project root:

```dockerfile
FROM php:8.2.12-apache

# Install dependencies including Git and Supervisor
RUN apt-get update && \
    apt-get install -y \
    libzip-dev \
    zip \
    libicu-dev \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libonig-dev \
    libxml2-dev \
    mariadb-client \
    curl \
    git \
    supervisor

# Enable mod_rewrite for Apache
RUN a2enmod rewrite

# Install PHP extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install pdo_mysql zip intl gd exif pcntl bcmath opcache

# Install Node.js 18.x and npm
RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs

# Set the document root to Laravel's public directory
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
RUN sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf

# Set ServerName to localhost to prevent Apache warning
RUN echo "ServerName localhost" >> /etc/apache2/conf-available/servername.conf && \
    a2enconf servername

# Set the working directory
WORKDIR /var/www/html

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Set environment variable to allow Composer to run as root
ENV COMPOSER_ALLOW_SUPERUSER=1

# Install Composer dependencies first to optimize Docker layer caching
COPY composer.json composer.lock /var/www/html/
RUN composer install --no-interaction --no-plugins --no-scripts

# Copy the rest of the application code
COPY . /var/www/html

# Install npm dependencies and build assets
RUN npm install && npm run build

# Ensure storage and bootstrap/cache directories exist and are writable
RUN mkdir -p /var/www/html/storage /var/www/html/bootstrap/cache \
    && chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache \
    && chmod -R 775 /var/www/html/storage /var/www/html/bootstrap/cache

# Set correct ownership for www-data
RUN chown -R www-data:www-data /var/www/html

RUN mkdir -p /etc/supervisor
# Copy Supervisor configuration
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Expose port 80
EXPOSE 80

# Use the custom entrypoint script
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]

# Set the default command to run Supervisor
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/conf.d/supervisord.conf"]```
```

Docker Entrypoint Script
Create a docker-entrypoint.sh file:

```docker-entrypoint
#!/bin/bash
set -e

# Ensure log directory exists and has correct permissions
mkdir -p /var/www/html/storage/logs
chown -R www-data:www-data /var/www/html/storage
chmod -R 775 /var/www/html/storage

# Run any necessary initialization steps
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Start Supervisor in the background
/usr/bin/supervisord -c /etc/supervisor/conf.d/supervisord.conf &

# Execute the main command (likely to be apache2-foreground)
exec "$@"
```

Supervisor Configuration
Create a supervisord.conf file:

```supervisord
[supervisord]
nodaemon=true
user=root
logfile=/var/www/html/storage/logs/supervisord.log
pidfile=/var/run/supervisord.pid

[program:apache2]
command=/usr/sbin/apache2ctl -D FOREGROUND
autostart=true
autorestart=true
priority=10

[program:laravel_queue_worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/html/artisan queue:work --sleep=3 --tries=3 --timeout=90 --verbose
autostart=true
autorestart=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/html/storage/logs/%(program_name)s_%(process_num)02d.log
stderr_logfile=/var/www/html/storage/logs/%(program_name)s_%(process_num)02d_error.log
```

Docker Compose Configuration
Create a docker-compose.yml file:

```docker-compose
services:
  app:
    image: your_image_name
    container_name: your_container_name
    restart: unless-stopped
    ports:
      - "8069:80"
    environment:
      APP_ENV: production
      APP_URL: https://yourdomain.com
      DB_HOST: mariadb
      DB_PORT: 3306
      DB_DATABASE: your_app_db
      DB_USERNAME: your_app_user
      DB_PASSWORD: your_strong_password
      MAIL_MAILER: ###
      MAIL_HOST: ###
      MAIL_PORT: ###
      MAIL_USERNAME: ###
      MAIL_PASSWORD: ###
      MAIL_ENCRYPTION: ###
      MAIL_FROM_NAME: ###
      MAIL_FROM_ADDRESS: ###
      QUEUE_CONNECTION: database
    depends_on:
      - mariadb
    volumes:
      - /your_app_path/storage:/var/www/html/storage
      - /your_app_path/bootstrap/cache:/var/www/html/bootstrap/cache

  mariadb:
    image: mariadb:10.6
    container_name: your_app_db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: your_app_db
      MYSQL_USER: your_app_user
      MYSQL_PASSWORD: your_strong_password
      MYSQL_ROOT_PASSWORD: your_app_root_password
    volumes:
      - /your_app_path/db:/var/lib/mysql

volumes:
  mariadb_data:
```

That's it! You now have a Docker setup for Laravel with Supervisor to manage long-running processes. If you encounter any problems or have any questions, please open an issue in this repository.

**this for setup laravel with docker and supervisor to work long process**

* create docker file on your project

`FROM php:8.2.12-apache


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


RUN a2enmod rewrite


RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
&& docker-php-ext-install pdo_mysql zip intl gd exif pcntl bcmath opcache


RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
&& apt-get install -y nodejs


ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
RUN sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf


RUN echo "ServerName localhost" >> /etc/apache2/conf-available/servername.conf && \
a2enconf servername


WORKDIR /var/www/html


RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer


ENV COMPOSER_ALLOW_SUPERUSER=1


COPY composer.json composer.lock /var/www/html/
RUN composer install --no-interaction --no-plugins --no-scripts


COPY . /var/www/html


RUN npm install && npm run build


RUN mkdir -p /var/www/html/storage /var/www/html/bootstrap/cache \
&& chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache \
&& chmod -R 775 /var/www/html/storage /var/www/html/bootstrap/cache


RUN chown -R www-data:www-data /var/www/html

RUN mkdir -p /etc/supervisor
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf


EXPOSE 80


COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]


CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/conf.d/supervisord.conf"]`

* create docker-entrypoint.sh

`#!/bin/bash
set -e


mkdir -p /var/www/html/storage/logs
chown -R www-data:www-data /var/www/html/storage
chmod -R 775 /var/www/html/storage

php artisan config:cache
php artisan route:cache
php artisan view:cache

/usr/bin/supervisord -c /etc/supervisor/conf.d/supervisord.conf &

exec "$@"`

* create supervisord.conf

`[supervisord]
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
stderr_logfile=/var/www/html/storage/logs/%(program_name)s_%(process_num)02d_error.log`


* build your image next docker composer


`services:
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
QUEUE_CONNECTION: ###
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
mariadb_data:`


* that's it

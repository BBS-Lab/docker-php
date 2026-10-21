ARG PHP_VERSION='8.5'

FROM php:${PHP_VERSION}-apache

ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
ENV APACHE_LOG_DIR=/var/log/apache2
ENV COMPOSER_ALLOW_SUPERUSER=1

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
COPY --from=ghcr.io/php/pie:1-bin /pie /usr/bin/pie

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    git curl wget gnupg2 zip unzip rsync ca-certificates vim htop cron sudo \
    # gd
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    # intl
    libicu-dev \
    # postgresql
    libpq-dev \
    # soap
    libxml2-dev \
    # zip
    libzip-dev \
    # mongodb
    libssl-dev \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/*

# opcache is built into PHP since 8.5
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) bcmath gd intl pcntl pdo_mysql pdo_pgsql soap zip exif \
        $(php -m | grep -q 'Zend OPcache' || echo opcache)

RUN pie install phpredis/phpredis

RUN echo "MAILTO=\"\"\n* * * * * www-data /usr/local/bin/php /var/www/html/artisan schedule:run >> /dev/null 2>&1" > /etc/cron.d/laravel

COPY etc/php/conf.d/web.ini /usr/local/etc/php/conf.d/web.ini

# Apache
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf \
    && a2disconf docker-php \
    && a2dissite 000-default.conf \
    && rm -rf /etc/apache2/sites-available/*

COPY etc/apache2/ /etc/apache2/

RUN a2ensite default \
    && a2enmod headers remoteip rewrite \
    && a2enconf remoteip security

COPY --chmod=755 entrypoint.sh /entrypoint

WORKDIR /var/www/html

EXPOSE 80

ENTRYPOINT ["/entrypoint"]

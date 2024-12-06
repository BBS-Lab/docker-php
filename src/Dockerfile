ARG PHP_VERSION='8.1'

FROM php:${PHP_VERSION}-apache

ARG PHP_VERSION

ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
ENV APACHE_LOG_DIR=/var/log/apache2
ENV COMPOSER_ALLOW_SUPERUSER=1

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        git curl wget gnupg2 zip unzip rsync ca-certificates vim htop cron sudo software-properties-common \
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
        libssl-dev

RUN if [ "$PHP_VERSION" = "7.4" ]; then \
    apt-get install -y --no-install-recommends libkrb5-3=1.18.3-6+deb11u5 tar=1.34+dfsg-1+deb11u1; \
elif [ "$PHP_VERSION" = "8.0" ]; then \
    apt-get install -y --no-install-recommends libgssapi-krb5-2=1.18.3-6+deb11u5 tar=1.34+dfsg-1+deb11u1; \
fi

RUN apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/*

RUN docker-php-ext-install -j$(nproc) bcmath gd intl opcache pcntl pdo_mysql pdo_pgsql soap zip exif

RUN pecl install -o -f redis \
    && rm -rf /tmp/pear \
    && docker-php-ext-enable redis

RUN echo "MAILTO=\"\"\n* * * * * www-data /usr/local/bin/php /var/www/html/artisan schedule:run >> /dev/null 2>&1" > /etc/cron.d/laravel

COPY etc/php/conf.d/web.ini /usr/local/etc/php/conf.d/web.ini

# Apache
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf \
    && sed -i 's/LogFormat "%h %l %u %t \\\"%r\\\" %>s %O \\\"/LogFormat "%a %l %u %t \\\"%r\\\" %>s %O \\\"/' /etc/apache2/apache2.conf \
    && service apache2 stop \
    && a2disconf docker-php \
    && a2dissite 000-default.conf \
    && rm -rf /etc/apache2/sites-available/*

COPY etc/apache2/ /etc/apache2/

RUN a2ensite default \
    && a2enmod headers remoteip rewrite \
    && a2enconf remoteip security

COPY entrypoint.sh /entrypoint

RUN chmod +x /entrypoint

WORKDIR /var/www/html

EXPOSE 80

ENTRYPOINT ["/entrypoint"]

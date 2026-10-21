# PHP Apache Docker image

[![Docker Pulls](https://img.shields.io/docker/pulls/bigbossstudio/php.svg?style=flat-square)](https://hub.docker.com/r/bigbossstudio/php)
[![Publish](https://img.shields.io/github/actions/workflow/status/BBS-Lab/docker-php/docker-publish-production.yml?branch=main&label=publish&style=flat-square)](https://github.com/BBS-Lab/docker-php/actions/workflows/docker-publish-production.yml)

A production-ready PHP + Apache image for Laravel applications, built on top of the official [`php:<version>-apache`](https://hub.docker.com/_/php) image (Debian 13). It ships the extensions, tools and Apache hardening we use on every Big Boss Studio project.

```bash
docker pull bigbossstudio/php:8.5-apache
```

## Tags

| Tag          | PHP | Platforms                       |
|--------------|-----|---------------------------------|
| `8.5-apache` | 8.5 | `linux/amd64`, `linux/arm64/v8` |
| `8.4-apache` | 8.4 | `linux/amd64`, `linux/arm64/v8` |

`linux/amd64` covers Intel Macs and x86 Linux servers, `linux/arm64/v8` covers Apple Silicon Macs, AWS Graviton and 64-bit ARM Linux. Docker picks the right one automatically.

Images are rebuilt on every push to `main` and every Monday at 08:00 UTC to pick up the latest PHP patch release and Debian security updates. Each build is also tagged `<version>-apache-<commit sha>`. The weekly rebuild updates that tag too: to freeze an image, pin its digest (`bigbossstudio/php:8.5-apache@sha256:…`).

## Usage

```dockerfile
ARG PHP_VERSION=8.5

FROM bigbossstudio/php:${PHP_VERSION}-apache AS base

# install dependencies, nodejs, etc.
# configure dev configurations (e.g opcache)

FROM base

WORKDIR /var/www/html
# copy application code, skipping files based on .dockerignore
COPY . /var/www/html

RUN composer install --optimize-autoloader --no-dev \
    && mkdir -p /var/www/html/storage/logs \
    && php artisan optimize:clear \
    && chown -R www-data:www-data /var/www/html \
    && chmod -R 775 /var/www/html/storage

# configure production configurations (e.g opcache)
```

The entrypoint starts Apache in the foreground. Any command passed to the container runs instead, as root (e.g. `docker run --rm bigbossstudio/php:8.5-apache php -v`).

## What's included

- **Apache** with `rewrite`, `headers` and `remoteip`. The document root is `/var/www/html/public`; override it with the `APACHE_DOCUMENT_ROOT` environment variable. Logs go to the container output, see [Logging](#logging).
- **[Composer 2](https://getcomposer.org)** and **[PIE](https://github.com/php/pie)** to add PHP extensions: `pie install vendor/extension`. The build dependencies (`gcc`, `make`, `phpize`…) are kept.
- **Tools**: `git`, `curl`, `wget`, `zip`, `unzip`, `rsync`, `gnupg2`, `vim`, `htop`, `cron`, `sudo`.
- **Laravel scheduler**: run it in a dedicated container with the command `php artisan schedule:work`. It inherits the container environment and its logs reach the container output. A cron file is also provided (`/etc/cron.d/laravel`, start it with `cron -f`), but cron jobs do not see the container environment variables and their output is discarded.

### PHP configuration

`/usr/local/etc/php/conf.d/web.ini`:

```ini
display_errors = Off
log_errors = On
upload_max_filesize = 16M
post_max_size = 20M
date.timezone = Europe/Paris
expose_php = Off
```

PHP errors are never displayed to visitors: they are written to stderr, see [Logging](#logging). `post_max_size` is a bit larger than `upload_max_filesize` so that a 16 MB file can be posted along with the other form fields.

### Security defaults

- `ServerTokens Prod`, `ServerSignature Off`, `TraceEnable Off` and `expose_php = Off`, so neither the Apache nor the PHP version is disclosed.
- Response headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: sameorigin`, `Referrer-Policy: strict-origin-when-cross-origin` and `Strict-Transport-Security: max-age=15552000; includeSubDomains`.
- Requests to hidden files and directories (`.env`, `.git`, `.user.ini`…) are denied, except `.well-known`. So are requests to `xmlrpc.php`.

### Client IP

The client IP (`REMOTE_ADDR`, access logs) is read from the `CF-Connecting-IP` header when the request comes from a [Cloudflare IP](https://www.cloudflare.com/ips/) or from a private network (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), i.e. a load balancer or reverse proxy in front of the container.

- **Behind Cloudflare**: make sure the container, or the load balancer in front of it, only accepts traffic from Cloudflare IPs. Otherwise any client can spoof its IP by sending this header.
- **Not behind Cloudflare** (AWS ALB without Cloudflare, Traefik, nginx…): disable it, and let your framework read `X-Forwarded-For` (Laravel: [trusted proxies](https://laravel.com/docs/requests#configuring-trusted-proxies)).

  ```dockerfile
  RUN a2disconf remoteip
  ```

### PHP modules

`bcmath`, `Core`, `ctype`, `curl`, `date`, `dom`, `exif`, `fileinfo`, `filter`, `gd` (FreeType, JPEG, PNG), `hash`, `iconv`, `intl`, `json`, `libxml`, `mbstring`, `mysqlnd`, `openssl`, `pcntl`, `pcre`, `PDO`, `pdo_mysql`, `pdo_pgsql`, `pdo_sqlite`, `Phar`, `posix`, `random`, `readline`, `redis`, `Reflection`, `session`, `SimpleXML`, `soap`, `sodium`, `SPL`, `sqlite3`, `standard`, `tokenizer`, `xml`, `xmlreader`, `xmlwriter`, `Zend OPcache`, `zip`, `zlib`.

PHP 8.5 also includes `lexbor` and `uri`.

## Logging

Everything goes to the container output, so it is picked up as is by `docker logs`, AWS CloudWatch (`awslogs` driver) or any log collector:

- **stdout**: Apache access logs, in the `combined` format followed by the response time in microseconds. AWS load balancer health checks (`ELB-HealthChecker` user agent from a private IP) are not logged.
- **stderr**: Apache and PHP errors, from warnings up (deprecations are not logged). Multi-line errors, such as an uncaught exception and its stack trace, are written on a single line.

Laravel writes to `storage/logs/laravel.log` by default, which is lost when the container stops. Send its logs to stderr as JSON: one line per log entry (stack trace included) whose fields CloudWatch Logs Insights parses automatically.

```dotenv
LOG_CHANNEL=stderr
LOG_STDERR_FORMATTER=Monolog\Formatter\JsonFormatter
```

## Recipes

### Node.js

```dockerfile
FROM bigbossstudio/php:8.5-apache

ARG NODE_VERSION=24

RUN curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION}.x | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*
```

### Puppeteer & Chromium

Uses the Chromium package from Debian instead of the one bundled with Puppeteer. Node.js is required (see above).

```dockerfile
ENV PUPPETEER_SKIP_DOWNLOAD=true
ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium
ENV NODE_PATH=/usr/lib/node_modules

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        chromium fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg fonts-freefont-ttf libxss1 \
    && rm -rf /var/lib/apt/lists/*

RUN npm install --global puppeteer
```

## Building locally

```bash
./dev.sh        # builds the versions listed in dev.sh
./dev.sh 8.5    # builds a single version
```

See [CONTRIBUTING](https://github.com/BBS-Lab/docker-php/blob/main/CONTRIBUTING.md).

## Security

Please report vulnerabilities by email, see [SECURITY](https://github.com/BBS-Lab/docker-php/blob/main/SECURITY.md).

## License

MIT. See [LICENSE.md](https://github.com/BBS-Lab/docker-php/blob/main/LICENSE.md).

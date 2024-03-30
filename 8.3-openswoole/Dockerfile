# -----------------------------------------------------
# App
# -----------------------------------------------------
FROM openswoole/swoole:22.1-php8.3-alpine

ARG PUBLIC_DIR=public

ENV PUBLIC_DIR=$PUBLIC_DIR

ENV REQUIRED_PACKAGES="git make zip curl libpq supervisor gettext acl fcgi python3 ca-certificates pcre rabbitmq-c icu libavif libwebp libpng freetype libjpeg-turbo libacl libheif libde265 jpegoptim optipng pngquant gifsicle inotify-tools linux-headers"
ENV EXTENSIONS="redis apcu ast maxminddb amqp zip pdo_mysql pdo_pgsql bcmath opcache gettext intl exif pcntl sysvmsg sysvsem sysvshm xsl uuid igbinary bz2 gmp gd inotify"

ENV DOCKER=true
ENV COMPOSER_ALLOW_SUPERUSER=1 COMPOSER_NO_INTERACTION=1 COMPOSER_CACHE_DIR="/tmp"

WORKDIR /app

# Copying manifest files to host
COPY ./8.3-openswoole/manifest /

# Composer install
COPY --from=composer/composer:2-bin /composer /usr/bin/composer

# php extensions installer: https://github.com/mlocati/docker-php-extension-installer
ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/
RUN chmod +x /usr/local/bin/install-php-extensions

# Install required packages
RUN apk add --update --no-cache $REQUIRED_PACKAGES

# Install PHP extensions
RUN install-php-extensions $EXTENSIONS

# Install sockets
RUN CFLAGS="$CFLAGS -D_GNU_SOURCE" docker-php-ext-install sockets && docker-php-ext-enable sockets

# Update ulimit
RUN ulimit -n 16384

# Swoole autoreload
RUN chmod +x /usr/local/bin/autoreload.sh

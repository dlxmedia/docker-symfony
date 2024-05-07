ARG PHP_VERSION=8.3
ARG CADDY_VERSION=2

# -----------------------------------------------------
# Caddy Install
# -----------------------------------------------------
FROM caddy:$CADDY_VERSION-builder AS builder
ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH xcaddy build

# -----------------------------------------------------
# App Itself
# -----------------------------------------------------
FROM php:$PHP_VERSION-fpm-alpine

ARG PORT=9001
ARG PUBLIC_DIR=public

ENV PORT=$PORT
ENV PUBLIC_DIR=$PUBLIC_DIR

ENV REQUIRED_PACKAGES="git make zip curl libpq supervisor gettext acl fcgi python3 ca-certificates pcre rabbitmq-c icu libavif libwebp libpng freetype libjpeg-turbo libacl libheif libde265 imagemagick imagemagick-dev imagemagick-libs jpegoptim optipng pngquant gifsicle linux-headers file"
ENV EXTENSIONS="redis apcu ast maxminddb amqp zip pdo_mysql pdo_pgsql bcmath opcache gettext intl exif pcntl sysvmsg sysvsem sysvshm xsl uuid igbinary bz2 gmp gd"

ENV DOCKER=true
ENV COMPOSER_ALLOW_SUPERUSER=1 COMPOSER_NO_INTERACTION=1 COMPOSER_CACHE_DIR="/tmp"

WORKDIR /app

# Copying manifest files to host
COPY ./8.3/manifest /

# Caddy
COPY --from=builder /usr/bin/caddy /usr/local/bin/caddy

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

# ImageMagick extension
RUN curl -L -o /tmp/imagick.tar.gz https://github.com/Imagick/imagick/archive/7088edc353f53c4bc644573a79cdcd67a726ae16.tar.gz \
    && tar --strip-components=1 -xf /tmp/imagick.tar.gz \
    && phpize \
    && ./configure \
    && make \
    && make install \
    && echo "extension=imagick.so" > /usr/local/etc/php/conf.d/ext-imagick.ini \
    && rm -rf /tmp/* \

# Update ulimit
RUN ulimit -n 16384

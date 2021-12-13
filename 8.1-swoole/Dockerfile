## -----------------------------------------------------
## Snowflake build
## -----------------------------------------------------
FROM golang:1.17-alpine as snowflake
ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

WORKDIR /app

RUN apk add git

RUN git clone https://github.com/dlxmedia/snowflake-json .
RUN go mod download && CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -a -o snowflake .

# -----------------------------------------------------
# App
# -----------------------------------------------------
FROM openswoole/swoole:4.8.1-php8.1-alpine

ARG PUBLIC_DIR=public

ENV PUBLIC_DIR=$PUBLIC_DIR

ENV REQUIRED_PACKAGES="git make openssl-dev zip curl curl-dev libpq supervisor python3 ca-certificates pcre linux-headers gettext-dev mysql-dev postgresql-dev rabbitmq-c icu libsodium-dev oniguruma-dev libjpeg-turbo-dev freetype-dev libpng-dev libwebp libwebp-dev libpng freetype libjpeg-turbo libxml2-dev libacl libheif libde265 libmagic imagemagick-dev imagemagick jpegoptim optipng pngquant gifsicle"
ENV DEVELOPMENT_PACKAGES="$PHPIZE_DEPS autoconf g++ openssh-client tar py-pip pcre-dev rabbitmq-c-dev icu-dev zlib-dev libzip-dev libmaxminddb-dev"
ENV PECL_PACKAGES="redis apcu ast inotify"
ENV EXT_PACKAGES="sockets pdo_mysql pdo_pgsql bcmath gettext intl gd exif pcntl sysvmsg sysvsem sysvshm"

ENV DOCKER=true
ENV COMPOSER_ALLOW_SUPERUSER 1
ENV COMPOSER_NO_INTERACTION 1
ENV COMPOSER_CACHE_DIR /tmp

WORKDIR /app

# Copying manifest files to host
COPY ./8.1-swoole/manifest /

# Snowflake
COPY --from=snowflake /app/snowflake /usr/local/bin/snowflake

# Copy Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Install Packages
RUN apk add --update --no-cache $REQUIRED_PACKAGES $DEVELOPMENT_PACKAGES

# Update ulimit
RUN ulimit -n 16384

# Fix Iconv
RUN apk add --no-cache --repository https://dl-cdn.alpinelinux.org/alpine/edge/community/ --allow-untrusted gnu-libiconv
ENV LD_PRELOAD /usr/lib/preloadable_libiconv.so php

# Configure GD to use freetype fonts
RUN docker-php-ext-configure gd --with-freetype --with-jpeg

# Install Zip extension
RUN docker-php-ext-configure zip && docker-php-ext-install zip

# Install Non-Pecl Packages
RUN docker-php-ext-install $EXT_PACKAGES

# Install Pecl Packages
#RUN apk add --no-cache $PHPIZE_DEPS && wget https://github.com/FriendsOfPHP/pickle/releases/latest/download/pickle.phar && mv pickle.phar /usr/local/bin/pickle && chmod +x /usr/local/bin/pickle
#RUN for package in $PICKLE_PACKAGES; do pickle install $package --defaults; done
RUN yes '' | pecl install -f $PECL_PACKAGES; \
    docker-php-ext-enable $PECL_PACKAGES

# AMQP from master @TODO upgrade to release version when php8 support is ready
ENV EXT_AMQP_VERSION=master
RUN docker-php-source extract \
    && apk -Uu add git rabbitmq-c-dev \
    && git clone --branch $EXT_AMQP_VERSION --depth 1 https://github.com/php-amqp/php-amqp.git /usr/src/php/ext/amqp \
    && cd /usr/src/php/ext/amqp && git submodule update --init \
    && docker-php-ext-install amqp \
    && docker-php-source delete

# Imagick from master @TODO upgrade to release version when php8 support is ready
ENV EXT_IMAGICK_VERSION=master
RUN docker-php-source extract \
    && git clone --branch $EXT_IMAGICK_VERSION --depth 1 https://github.com/Imagick/imagick.git /usr/src/php/ext/imagick \
    && cd /usr/src/php/ext/imagick && git submodule update --init \
    && docker-php-ext-install imagick \
    && docker-php-source delete

# MaxMind
RUN apk add --no-cache libmaxminddb \
    && git clone https://github.com/maxmind/MaxMind-DB-Reader-php.git \
    && ( \
      cd MaxMind-DB-Reader-php/ext \
      && phpize \
      && ./configure \
      && make \
      && make install \
    ) \
    && rm -r MaxMind-DB-Reader-php \
    && docker-php-ext-install maxminddb

# Swoole autoreload
RUN chmod +x /usr/local/bin/autoreload.sh

# see: https://github.com/swoole/ext-postgresql/issues/9
ENV EXT_POSTGRES_VERSION=master
RUN git clone --branch $EXT_POSTGRES_VERSION --depth 1 https://github.com/swoole/ext-postgresql.git  \
    && ( \
      cd ext-postgresql \
      && sed -i "s/#include <postgresql\/libpq-fe.h>/#include <\/usr\/include\/libpq-fe.h>/" php_swoole_postgresql.h \
      && phpize \
      && ./configure \
      && make \
      && make install \
    ) \
    && rm -r ext-postgresql \
    && docker-php-source delete \
    && /bin/sh -c docker-php-ext-enable swoole_postgresql

# Delete Non-Required Packages
RUN apk del $DEVELOPMENT_PACKAGES

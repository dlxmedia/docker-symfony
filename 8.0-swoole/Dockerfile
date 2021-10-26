# -----------------------------------------------------
# Snowflake build
# -----------------------------------------------------
FROM golang:1.16-alpine as snowflake

WORKDIR /app

RUN apk add git

RUN git clone https://github.com/dlxmedia/snowflake-json .
RUN go mod download && CGO_ENABLED=0 GOOS=linux go build -a -o snowflake .

# -----------------------------------------------------
# App
# -----------------------------------------------------
FROM phpswoole/swoole:4.8.0-php8.0-alpine

ARG PUBLIC_DIR=public

ENV PUBLIC_DIR=$PUBLIC_DIR

ENV REQUIRED_PACKAGES="git make openssl-dev zlib-dev libzip-dev zip curl curl-dev libpq supervisor python3 ca-certificates pcre linux-headers gettext-dev mysql-dev postgresql-dev rabbitmq-c icu libsodium-dev oniguruma-dev libjpeg-turbo-dev freetype-dev libpng-dev libwebp libwebp-dev libpng freetype libjpeg-turbo jpegoptim optipng pngquant gifsicle libheif libde265 libmagic imagemagick-dev imagemagick inotify-tools"
ENV DEVELOPMENT_PACKAGES="autoconf g++ openssh-client tar py-pip pcre-dev rabbitmq-c-dev icu-dev"
ENV PICKLE_PACKAGES="amqp"
ENV PECL_PACKAGES="redis apcu ast inotify"
ENV EXT_PACKAGES="zip sockets pdo_pgsql bcmath opcache mbstring iconv gettext intl exif sodium gd pcntl sysvmsg sysvsem sysvshm"

ENV DOCKER=true
ENV COMPOSER_ALLOW_SUPERUSER 1
ENV COMPOSER_NO_INTERACTION 1
ENV COMPOSER_CACHE_DIR /tmp

WORKDIR /app

# Copying manifest files to host
COPY ./8.0-swoole/manifest /

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

# Install Non-Pecl Packages
RUN docker-php-ext-install $EXT_PACKAGES

# Install Pecl Packages
RUN apk add --no-cache $PHPIZE_DEPS && wget https://github.com/FriendsOfPHP/pickle/releases/latest/download/pickle.phar && mv pickle.phar /usr/local/bin/pickle && chmod +x /usr/local/bin/pickle
RUN for package in $PICKLE_PACKAGES; do pickle install $package --defaults; done
RUN yes '' | pecl install -f $PECL_PACKAGES
RUN docker-php-ext-enable $PICKLE_PACKAGES $PECL_PACKAGES

# MaxMind
RUN apk add --no-cache libmaxminddb \
    && apk add --no-cache --virtual .build-deps $PHPIZE_DEPS git libmaxminddb-dev \
    && git clone https://github.com/maxmind/MaxMind-DB-Reader-php.git \
    && ( \
      cd MaxMind-DB-Reader-php/ext \
      && phpize \
      && ./configure \
      && make \
      && make install \
    ) \
    && rm -r MaxMind-DB-Reader-php \
    && docker-php-ext-enable maxminddb

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
    && docker-php-ext-enable swoole_postgresql

# Delete Non-Required Packages
RUN apk del $DEVELOPMENT_PACKAGES

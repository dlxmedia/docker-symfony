ARG PHP_VERSION=8.0
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
# Snowflake build
# -----------------------------------------------------
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
# App Itself
# -----------------------------------------------------
FROM php:$PHP_VERSION-fpm-alpine

ARG PORT=9001
ARG PUBLIC_DIR=public

ENV PORT=$PORT
ENV PUBLIC_DIR=$PUBLIC_DIR

ENV REQUIRED_PACKAGES="git make openssl-dev zip libzip curl curl-dev libpq supervisor python3 ca-certificates pcre linux-headers gettext-dev mysql-dev postgresql-dev rabbitmq-c icu libsodium-dev oniguruma-dev libjpeg-turbo-dev freetype-dev libpng-dev libwebp libwebp-dev libpng freetype libjpeg-turbo libxml2-dev libacl libheif libde265 libmagic imagemagick-dev imagemagick jpegoptim optipng pngquant gifsicle libmaxminddb libxslt-dev libgcrypt-dev"
ENV DEVELOPMENT_PACKAGES="autoconf g++ openssh-client tar py-pip pcre-dev rabbitmq-c-dev icu-dev zlib-dev libzip-dev libmaxminddb-dev"
ENV PECL_PACKAGES="redis apcu ast maxminddb"
ENV EXT_PACKAGES="zip pdo_mysql pdo_pgsql bcmath opcache mbstring gettext intl exif sodium pcntl sysvmsg sysvsem sysvshm xsl"

ENV DOCKER=true
ENV COMPOSER_ALLOW_SUPERUSER 1
ENV COMPOSER_NO_INTERACTION 1
ENV COMPOSER_CACHE_DIR /tmp

WORKDIR /app

# Copying manifest files to host
COPY ./8.0/manifest /

# Snowflake
COPY --from=snowflake /app/snowflake /usr/local/bin/snowflake

# Caddy
COPY --from=builder /usr/bin/caddy /usr/local/bin/caddy

# Composer install
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Install Packages
RUN apk add --update --no-cache $REQUIRED_PACKAGES $DEVELOPMENT_PACKAGES

# Update ulimit
RUN ulimit -n 16384

# Install Non-Pecl Packages
RUN docker-php-ext-install $EXT_PACKAGES

# Configure GD to use freetype fonts
RUN docker-php-ext-configure gd --enable-gd --with-freetype --with-jpeg --with-webp && docker-php-ext-install gd && docker-php-ext-enable gd

# Install sockets
RUN CFLAGS="$CFLAGS -D_GNU_SOURCE" docker-php-ext-install sockets && docker-php-ext-enable sockets

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

# Delete Non-Required Packages
RUN apk del $DEVELOPMENT_PACKAGES

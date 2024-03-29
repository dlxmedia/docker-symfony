ARG PHP_VERSION=8.0
ARG RR_VERSION=2.6.0
ARG RR_IMAGE=spiralscout/roadrunner:${RR_VERSION}

# -----------------------------------------------------
# Roadrunner
# -----------------------------------------------------
FROM ${RR_IMAGE} as rr

# -----------------------------------------------------
# App Itself
# -----------------------------------------------------
FROM php:${PHP_VERSION}-cli-alpine

ARG PORT=9001
ARG PUBLIC_DIR=public

ENV PORT=$PORT
ENV PUBLIC_DIR=$PUBLIC_DIR

ENV REQUIRED_PACKAGES="git make libzip zip curl supervisor python3 ca-certificates pcre linux-headers gettext-dev mysql-dev postgresql-dev rabbitmq-c icu libsodium-dev oniguruma-dev libjpeg-turbo-dev freetype-dev libpng-dev libwebp libwebp-dev libpng freetype libjpeg-turbo jpegoptim optipng pngquant gifsicle libheif libde265 libmagic imagemagick-dev imagemagick libmaxminddb libxslt-dev libgcrypt-dev"
ENV DEVELOPMENT_PACKAGES="autoconf g++ openssh-client tar py-pip pcre-dev rabbitmq-c-dev icu-dev libmaxminddb-dev zlib-dev libzip-dev"
ENV PECL_PACKAGES="redis apcu ast maxminddb"
ENV EXT_PACKAGES="zip pdo_mysql pdo_pgsql bcmath opcache mbstring gettext intl exif sodium gd pcntl sysvmsg sysvsem sysvshm xsl"

ENV DOCKER=true
ENV COMPOSER_ALLOW_SUPERUSER 1
ENV COMPOSER_NO_INTERACTION 1
ENV COMPOSER_CACHE_DIR /tmp

WORKDIR /app

# Copying manifest files to host
COPY ./8.0-rr/manifest /

# Copy RoadRunner
COPY --from=rr /usr/bin/rr /usr/bin/rr

# Copy Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Install Packages
RUN apk add --update --no-cache $REQUIRED_PACKAGES $DEVELOPMENT_PACKAGES

# Update ulimit
RUN ulimit -n 16384

# Configure GD to use freetype fonts
RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp

# Install Non-Pecl Packages
RUN docker-php-ext-install $EXT_PACKAGES

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

# syntax = docker/dockerfile:1.2
FROM dunglas/frankenphp:1.5.0-php8.2-bookworm as base

RUN apt-get update && apt-get install -y git bind9-utils mycli nodejs npm nano \
  && mkdir -p /root/.ssh \
  && ssh-keyscan github.com >> /root/.ssh/known_hosts

RUN install-php-extensions \
  pdo_mysql \
  gd \
  bcmath \
  redis \
  intl \
  zip \
  gmp \
  apcu \
  opcache \
  memcached \
  imagick \
  sockets \
  pcntl \
  @composer

RUN apt-get update && apt-get install -y --no-install-recommends \
  pkg-config libgeos-dev libgeos++-dev autoconf build-essential unzip \
  && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL -o php-geos.zip \
  https://github.com/libgeos/php-geos/archive/dfe1ab17b0f155cc315bc13c75689371676e02e1.zip \
  && unzip php-geos.zip \
  && rm php-geos.zip \
  && cd php-geos-* \
  && ./autogen.sh && ./configure && make -j"$(nproc)" install \
  && docker-php-ext-enable geos \
  && cd .. && rm -rf php-geos-*

RUN sed -e 's/^expose_php.*/expose_php = Off/' "$PHP_INI_DIR/php.ini-production" > "$PHP_INI_DIR/php.ini" \
  && sed -i -e 's/^upload_max_filesize.*/upload_max_filesize = 600M/' -e 's/^post_max_size.*/post_max_size = 0/' \
  -e 's/^memory_limit.*/memory_limit = 600M/' "$PHP_INI_DIR/php.ini"

RUN npm install -g chokidar pnpm ember-cli npm-cli-login

RUN mkdir -p ~/.pnpm
ENV PNPM_HOME /root/.pnpm
ENV PATH /root/.pnpm/bin:$PATH

ENV LOG_CHANNEL=stdout
ENV CACHE_DRIVER=null
ENV BROADCAST_DRIVER=socketcluster
ENV QUEUE_CONNECTION=redis
ENV CADDYFILE_PATH=/fleetbase/Caddyfile
ENV CONSOLE_PATH=/fleetbase/console
ENV OCTANE_SERVER=frankenphp

ARG ENVIRONMENT=production
ENV APP_ENV=$ENVIRONMENT

ARG GITHUB_AUTH_KEY

COPY --chown=www-data:www-data ./Caddyfile $CADDYFILE_PATH
RUN mkdir -p /fleetbase/api && mkdir -p /fleetbase/console && chown -R www-data:www-data /fleetbase

WORKDIR /fleetbase/api

RUN if [ -n "$GITHUB_AUTH_KEY" ]; then echo "{\"github-oauth\": {\"github.com\": \"$GITHUB_AUTH_KEY\"}}" > auth.json; fi

RUN mkdir -p /var/www/.cache/composer && chown -R www-data:www-data /var/www/.cache/composer

COPY --chown=www-data:www-data ./api/composer.json ./api/composer.lock /fleetbase/api/
RUN su www-data -s /bin/sh -c "composer install --no-scripts --optimize-autoloader --no-dev --no-cache"

COPY --chown=www-data:www-data ./api /fleetbase/api
RUN su www-data -s /bin/sh -c "composer dumpautoload"

RUN mkdir -p /root/.composer
RUN mkdir -p /fleetbase/api/.composer && chown www-data:www-data /fleetbase/api/.composer

RUN mkdir -p /fleetbase/api/storage/logs/ && touch /fleetbase/api/storage/logs/laravel-$(date +'%Y-%m-%d').log
RUN chown -R www-data:www-data /fleetbase/api/storage
RUN chmod -R 755 /fleetbase/api/storage

RUN chmod +x /fleetbase/api/deploy.sh

# Scheduler base
FROM base as scheduler
RUN curl -L https://github.com/webdevops/go-crond/releases/download/23.12.0/go-crond.linux.amd64 > /usr/local/bin/go-crond && chmod +x /usr/local/bin/go-crond
COPY docker/crontab ./crontab
RUN chmod 0600 ./crontab
CMD ["go-crond", "--verbose", "root:./crontab"]

# Queue worker
FROM base as worker
CMD ["php", "artisan", "queue:work"]

# Web app (API)
FROM base as app
CMD ["sh", "-c", "php artisan octane:frankenphp --max-requests=250 --port=8000 --host=0.0.0.0"]

# This argument defines the PHP version for the image.
# It defaults to 8.4 if no 'PHP_VERSION' build-arg is passed from docker-compose.
ARG PHP_VERSION=8.4

FROM php:${PHP_VERSION}-fpm

# Set timezone
ENV TZ=UTC
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Install system packages
RUN apt-get update && apt-get install -y supervisor cron tzdata curl rsync ca-certificates

# ----------------------------------------------------------------------
# Bake a known-working OpenCart release into the image at /opt/opencart-dist
# so we can overlay it onto the bind-mounted project at runtime when the
# local `upload/` tree is broken (mid-migration on master, mixed .html/.twig
# templates with invalid Twig syntax). The host `upload/` is NEVER modified.
# Override the pinned version with build arg OPENCART_DIST_VERSION.
# ----------------------------------------------------------------------
ARG OPENCART_DIST_VERSION=4.1.0.3
ENV OPENCART_DIST_VERSION=${OPENCART_DIST_VERSION}
ENV OPENCART_DIST_DIR=/opt/opencart-dist
RUN mkdir -p "${OPENCART_DIST_DIR}" && \
    curl -fsSL "https://codeload.github.com/opencart/opencart/tar.gz/refs/tags/${OPENCART_DIST_VERSION}" \
        -o /tmp/opencart-dist.tgz && \
    tar -xzf /tmp/opencart-dist.tgz -C "${OPENCART_DIST_DIR}" --strip-components=1 && \
    rm -f /tmp/opencart-dist.tgz && \
    echo "${OPENCART_DIST_VERSION}" > "${OPENCART_DIST_DIR}/.dist-version"

# Install PHP extensions
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/
RUN install-php-extensions \
    @composer \
    apcu \
    gd \
    intl \
    memcached \
    mysqli \
    opcache \
    pdo_mysql \
    pdo_pgsql \
    redis \
    zip

# Configure user home directory
ARG PHP_UNAME=www-data
ARG PHP_UID=33
ENV PHP_UNAME=${PHP_UNAME}
ENV PHP_UID=${PHP_UID}

RUN if ! id -u "$PHP_UNAME" >/dev/null 2>&1; then \
        useradd -m -G www-data,root -u "$PHP_UID" -d "/home/$PHP_UNAME" "$PHP_UNAME"; \
    fi && \
    chown -R "$PHP_UNAME":www-data /var/www && \
    chmod -R g+w /var/www

ENV HOME=/home/$PHP_UNAME

# Copy OpenCart's original php.ini
COPY upload/php.ini /usr/local/etc/php/conf.d/opencart.ini

# Copy project PHP-FPM and Supervisor configurations.
COPY ./docker/php/supervisor/supervisord.conf /etc/supervisor/supervisord.conf
COPY ./docker/php/supervisor/services/ /etc/supervisor/conf.d/services/

# Copy OpenCart entrypoint
COPY docker/php/docker-opencart-entrypoint /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-opencart-entrypoint

# Set working directory
WORKDIR /var/www

# Use OpenCart entrypoint
ENTRYPOINT ["docker-opencart-entrypoint"]
CMD ["supervisord", "-c", "/etc/supervisor/supervisord.conf"]

# ============================================================
# Stage 1: Composer builder — install PHP dependencies
# ============================================================
FROM composer:2 AS builder

WORKDIR /app

# Copy only dependency files first (cache-friendly layer)
COPY composer.json composer.lock ./

# Install production dependencies
# --ignore-platform-reqs: builder image có thể thiếu một số PHP extensions
# --no-scripts: tránh chạy artisan package:discover khi chưa có env/runtime đầy đủ
RUN composer install \
    --no-dev \
    --optimize-autoloader \
    --no-interaction \
    --no-progress \
    --prefer-dist \
    --ignore-platform-reqs \
    --no-scripts

# Copy the full application source
COPY . .

# ============================================================
# Stage 2: Runtime — PHP-FPM + Nginx in one container
# ============================================================
FROM php:8.2-fpm-alpine AS runtime

LABEL maintainer="Petty VMCS Team"

# Install system packages and PHP extensions needed by Laravel
RUN apk add --no-cache \
    nginx \
    supervisor \
    curl \
    bash \
    libpng-dev \
    libjpeg-turbo-dev \
    freetype-dev \
    libzip-dev \
    oniguruma-dev \
    icu-dev \
    zlib-dev \
    libxml2-dev \
    autoconf \
    g++ \
    make \
    linux-headers \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install \
        pdo \
        pdo_mysql \
        opcache \
        gd \
        bcmath \
        intl \
        mbstring \
        zip \
        exif \
    && docker-php-ext-enable opcache

# Copy application from builder stage
WORKDIR /var/www/html
COPY --from=builder /app ./

# Create Laravel required directories and set permissions
RUN mkdir -p \
        storage/logs \
        storage/framework/cache/data \
        storage/framework/sessions \
        storage/framework/views \
        bootstrap/cache \
    && chown -R www-data:www-data /var/www/html \
    && chmod -R 775 storage bootstrap/cache

# Copy Nginx and Supervisor configuration files
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/supervisord.conf /etc/supervisord.conf
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# PHP-FPM config
RUN sed -i 's|^;*daemonize =.*|daemonize = no|g' /usr/local/etc/php-fpm.conf || true \
    && sed -i 's|^user = .*|user = www-data|g' /usr/local/etc/php-fpm.d/www.conf \
    && sed -i 's|^group = .*|group = www-data|g' /usr/local/etc/php-fpm.d/www.conf \
    && sed -i 's|^listen = .*|listen = 127.0.0.1:9000|g' /usr/local/etc/php-fpm.d/www.conf \
    && sed -i 's|^;clear_env = no|clear_env = no|g' /usr/local/etc/php-fpm.d/www.conf || true

# Expose HTTP port
EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -f http://localhost/api/health || exit 1

ENTRYPOINT ["/entrypoint.sh"]

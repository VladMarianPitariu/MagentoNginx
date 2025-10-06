FROM php:8.2-fpm

# System deps
RUN apt-get update && apt-get install -y \
    git unzip libzip-dev libicu-dev libxml2-dev libxslt1-dev libpng-dev \
    libonig-dev libjpeg-dev libfreetype6-dev libssl-dev libcurl4-openssl-dev \
    && rm -rf /var/lib/apt/lists/*

# PHP extensions needed by Magento
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
       bcmath intl pdo_mysql soap xsl zip gd opcache ftp sockets

# Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# PHP config tweaks via environment
RUN { \
  echo "memory_limit=${PHP_MEMORY_LIMIT}"; \
  echo "upload_max_filesize=${PHP_UPLOAD_MAX_FILESIZE}"; \
  echo "post_max_size=${PHP_POST_MAX_SIZE}"; \
  echo "max_execution_time=1800"; \
} > /usr/local/etc/php/conf.d/99-magento.ini

WORKDIR /var/www/html

# www-data owns the code (host bind mount will override perms at runtime)
RUN usermod -u 1000 www-data && groupmod -g 1000 www-data

CMD ["php-fpm"]
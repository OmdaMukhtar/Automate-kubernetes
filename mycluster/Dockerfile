FROM php:8.2-fpm

RUN apt-get update && apt-get install -y \
    zip unzip git curl libonig-dev libpng-dev libjpeg-dev libfreetype6-dev \
    && docker-php-ext-install pdo pdo_mysql mbstring gd

WORKDIR /var/www/html

COPY ./laravel .

RUN curl -sS https://getcomposer.org/installer | php -- \
    --install-dir=/usr/local/bin --filename=composer

RUN composer install --no-dev --optimize-autoloader


RUN chown -R www-data:www-data /var/www/html

CMD ["php-fpm"]

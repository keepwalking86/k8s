FROM composer:1.8.3 as build

WORKDIR /app
COPY app /app
RUN composer install

FROM php:7.2-apache
ENV TZ='Asia/Ho_Chi_Minh'
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

EXPOSE 80
COPY --from=build /app /app
# Install package
RUN apt-get update
RUN apt-get install vim git zip unzip libpng-dev libzip-dev librabbitmq-dev libicu-dev cron -y
# Database
RUN pecl install mongodb \
        && docker-php-ext-enable mongodb
RUN docker-php-ext-install pdo_mysql zip bcmath gd opcache
RUN pecl install amqp \
    && docker-php-ext-enable amqp

# Intl
RUN apt-get install zlib1g-dev libicu-dev -y
RUN docker-php-ext-configure intl
RUN docker-php-ext-install intl

# Timezone
RUN echo "date.timezone = Asia/Ho_Chi_Minh" >> /usr/local/etc/php/php.ini
RUN echo "memory_limit = 1024M" >> /usr/local/etc/php/php.ini
COPY vhost.conf /etc/apache2/sites-available/000-default.conf
RUN chown -R www-data:www-data /app \
    && a2enmod rewrite

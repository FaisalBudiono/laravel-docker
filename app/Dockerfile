FROM php:8.2-fpm AS php-base
WORKDIR /var/www/html
ENV EXTENSION_DIR=no-debug-non-zts-20220829
# Install Composer
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" \
    && php composer-setup.php \
    && php -r "unlink('composer-setup.php');" \
    && mv composer.phar /usr/local/bin/composer
# Set php.ini
RUN cp /usr/local/etc/php/php.ini-development /usr/local/etc/php/php.ini
# Install phpredis
RUN apt-get update \
    && printf "\n" | printf "\n" | pecl install redis \
    && docker-php-ext-enable redis
# Install zip PHP dependencies
RUN apt-get update \
    && apt-get install -y git zip libzip-dev
RUN docker-php-ext-install zip \
    && docker-php-ext-enable zip
# Install PDO and bcmath
RUN docker-php-ext-install pdo_mysql bcmath
# Install gd jpeg, webp, etc. support
RUN apt-get update \
    && apt-get install -y libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    libwebp-dev \
    libmagickwand-dev
RUN docker-php-ext-configure gd --enable-gd --with-freetype --with-jpeg --with-webp \
    && cd /usr/src/php/ext/gd && make \
    && cp /usr/src/php/ext/gd/modules/gd.so /usr/local/lib/php/extensions/${EXTENSION_DIR}/gd.so \
    && docker-php-ext-install -j$(nproc) gd
# Install sodium
RUN apt-get update \
    && apt-get install libsodium-dev \
    && docker-php-ext-install sodium
# Modify max upload filesize
RUN cp /usr/local/etc/php/php.ini /usr/local/etc/php/php.ini-backup \
    && sed 's/upload_max_filesize = 2M/upload_max_filesize = 100M/' </usr/local/etc/php/php.ini-backup >/usr/local/etc/php/php.ini \
    && cp /usr/local/etc/php/php.ini /usr/local/etc/php/php.ini-backup \
    && sed 's/post_max_size = 8M/post_max_size = 100M/' </usr/local/etc/php/php.ini-backup >/usr/local/etc/php/php.ini \
    && rm /usr/local/etc/php/php.ini-backup


FROM php-base AS php-final
RUN composer global require spatie/phpunit-watcher \
    && echo 'export PATH="$PATH:$HOME/.composer/vendor/bin"' >> ~/.bashrc
RUN pecl install xdebug \
    && echo "zend_extension=/usr/local/lib/php/extensions/${EXTENSION_DIR}/xdebug.so" >> /usr/local/etc/php/php.ini \
    && echo "xdebug.mode=coverage" >> /usr/local/etc/php/php.ini
RUN echo 'alias pw="phpunit-watcher watch --testdox"' >> ~/.bashrc \
    && echo "alias pp='php artisan test --parallel'" >> ~/.bashrc \
    # && echo "alias ppc='pp --coverage --cache-directory ./ ./tests'" >> ~/.bashrc \
    && echo "alias ppc='pp --coverage'" >> ~/.bashrc \
    && echo "alias pwf='pw --filter'" >> ~/.bashrc \
    && echo "alias ppf='./vendor/bin/paratest --functional --filter'" >> ~/.bashrc \
    && echo "alias ppff='./vendor/bin/paratest --functional'" >> ~/.bashrc \
    && echo "alias ppfg='./vendor/bin/paratest --functional --group'" >> ~/.bashrc \
    && echo "alias pps='./vendor/bin/paratest --stop-on-failure'" >> ~/.bashrc \
    && echo "alias ppsf='./vendor/bin/paratest --stop-on-failure --functional --filter'" >> ~/.bashrc \
    && echo "alias cw='chown -R www-data:www-data'" >> ~/.bashrc


FROM php-base as php-worker
RUN apt-get update \
    && apt-get install -y supervisor
COPY laravel-worker-api.conf /etc/supervisor/supervisord.conf
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf", "-n"]

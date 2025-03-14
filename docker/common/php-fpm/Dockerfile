FROM crpi-abrasomrm9oq6os4.cn-shanghai.personal.cr.aliyuncs.com/singlab/php:8.3-fpm AS builder
ARG APP_VERSION=1.0.0
LABEL version=${APP_VERSION}

# 配置阿里云镜像源并安装依赖
RUN mv /etc/apt/sources.list.d/debian.sources /etc/apt/debian.sources_bak && \
    echo "deb http://mirrors.aliyun.com/debian/ bookworm main non-free-firmware contrib" > /etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/debian-security/ bookworm-security main" >> /etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/debian/ bookworm-updates main non-free-firmware contrib" >> /etc/apt/sources.list && \
    apt-get update && apt-get install -y --no-install-recommends \
    curl unzip libpng-dev libpq-dev libonig-dev libssl-dev libxml2-dev \
    libcurl4-openssl-dev libicu-dev libzip-dev && \
    docker-php-ext-install -j$(nproc) pdo_mysql pdo_pgsql pgsql opcache intl zip bcmath soap && \
    pecl install redis && docker-php-ext-enable redis && \
    apt-get autoremove -y && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# 安装依赖和构建
WORKDIR /var/www
COPY . .
RUN  php composer.php --install-dir=/usr/local/bin --filename=composer && \
    composer config -g repo.packagist composer https://packages.aliyun.com/composer && \
    composer update --lock && composer install --no-dev --optimize-autoloader --no-interaction --no-progress --prefer-dist

FROM crpi-abrasomrm9oq6os4.cn-shanghai.personal.cr.aliyuncs.com/singlab/php:8.3-fpm AS production

# 安装生产环境必需的依赖
RUN mv /etc/apt/sources.list.d/debian.sources /etc/apt/debian.sources_bak && \
    echo "deb http://mirrors.aliyun.com/debian/ bookworm main non-free-firmware contrib" > /etc/apt/sources.list && \
    apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev libicu-dev libzip-dev libfcgi-bin procps && \
    apt-get autoremove -y && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# 配置 PHP 和 PHP-FPM
COPY docker/common/php-fpm/php-fpm-healthcheck /usr/local/bin/php-fpm-healthcheck
COPY docker/production/php-fpm/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/php-fpm-healthcheck /usr/local/bin/entrypoint.sh && \
    mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini" && \
    sed -i '/\[www\]/a pm.status_path = /status' /usr/local/etc/php-fpm.d/zz-docker.conf

# 复制应用文件和 PHP 扩展
COPY ./storage /var/www/storage-init
COPY --from=builder /usr/local/lib/php/extensions/ /usr/local/lib/php/extensions/
COPY --from=builder /usr/local/etc/php/conf.d/ /usr/local/etc/php/conf.d/
COPY --from=builder /usr/local/bin/docker-php-ext-* /usr/local/bin/
COPY --from=builder /var/www /var/www

WORKDIR /var/www
RUN chown -R www-data:www-data /var/www
USER www-data

EXPOSE 9000
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["php-fpm"]

FROM production AS development
ARG APP_VERSION=1.0.0
LABEL version=${APP_VERSION}

# 安装开发工具
USER root
RUN pecl install xdebug && docker-php-ext-enable xdebug && \
    echo "xdebug.mode=develop,coverage,debug,profile" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini && \
    echo "xdebug.client_host=host.docker.internal" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini && \
    echo "xdebug.start_with_request=yes" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini

# 配置开发用户
ARG UID=1000
ARG GID=1000
RUN if getent group ${GID}; then \
      group_name=$(getent group ${GID} | cut -d: -f1); \
      useradd -m -u ${UID} -g ${GID} -s /bin/bash www; \
    else \
      groupadd -g ${GID} www && \
      useradd -m -u ${UID} -g www -s /bin/bash www; \
      group_name=www; \
    fi && \
    sed -i "s/user = www-data/user = www/g" /usr/local/etc/php-fpm.d/www.conf && \
    sed -i "s/group = www-data/group = $group_name/g" /usr/local/etc/php-fpm.d/www.conf

COPY docker/development/php-fpm/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

USER www-data
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
EXPOSE 9000
CMD ["php-fpm"]

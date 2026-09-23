FROM php:8.2-cli-alpine

WORKDIR /app

# Install runtime and build dependencies for PostgreSQL and cURL extensions
RUN apk add --no-cache libpq libcurl \
 && apk add --no-cache --virtual .build-deps postgresql-dev curl-dev \
 && docker-php-ext-install pdo pdo_pgsql curl \
 && apk del .build-deps

# Copy application source
COPY . .

EXPOSE 8087

CMD ["php", "-S", "0.0.0.0:8087", "router.php"]

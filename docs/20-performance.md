# ЭТАП 20. Производительность

## Redis Cache

### Установка и настройка Redis

```bash
# Установка Redis
sudo apt update
sudo apt install redis-server

# Настройка в конфигурационном файле
sudo nano /etc/redis/redis.conf
```

### Конфигурация Redis для высокой производительности

```bash
# В файле /etc/redis/redis.conf

# Максимальное использование памяти
maxmemory 1gb

# Политика очистки при переполнении памяти
maxmemory-policy allkeys-lru

# Включить сохранение данных
save 900 1
save 300 10
save 60 10000

# Количество подключений
maxclients 10000

# Таймаут для простаивающих клиентов
timeout 300
```

### Перезапуск Redis

```bash
sudo systemctl restart redis
sudo systemctl enable redis
```

### Проверка производительности Redis

```bash
# Бенчмарк Redis
redis-benchmark -n 100000 -c 50

# Мониторинг команд
redis-cli monitor

# Статистика
redis-cli info stats
redis-cli info memory
```

## Queue Workers

### Настройка количества воркеров

```ini
# /etc/supervisor/conf.d/laravel-worker.conf
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/autoshop/artisan queue:work redis --sleep=1 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=5
redirect_stderr=true
stdout_logfile=/var/www/autoshop/storage/logs/worker.log
stopwaitsecs=3600

# Лимиты памяти
# В PHP-FPM пул: php_admin_value[memory_limit] = 512M
```

### Разделение очередей по приоритетам

```php
// routes/web.php или app/Providers/EventServiceProvider.php

Route::post('/checkout', [CheckoutController::class, 'store']);
```

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\SerializesModels;

class SendEmailJob implements ShouldQueue
{
    use Queueable, SerializesModels;

    public $queue = 'high';

    // ...
}

class UpdateCacheJob implements ShouldQueue
{
    use Queueable, SerializesModels;

    public $queue = 'low';

    // ...
}
```

### Supervisor для разных очередей

```ini
# /etc/supervisor/conf.d/laravel-worker-high.conf
[program:laravel-worker-high]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/autoshop/artisan queue:work redis --queue=high --sleep=1 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=www-data
numprocs=3
redirect_stderr=true
stdout_logfile=/var/www/autoshop/storage/logs/worker-high.log

# /etc/supervisor/conf.d/laravel-worker-default.conf
[program:laravel-worker-default]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/autoshop/artisan queue:work redis --queue=default --sleep=1 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=www-data
numprocs=3
redirect_stderr=true
stdout_logfile=/var/www/autoshop/storage/logs/worker-default.log

# /etc/supervisor/conf.d/laravel-worker-low.conf
[program:laravel-worker-low]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/autoshop/artisan queue:work redis --queue=low --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/autoshop/storage/logs/worker-low.log
```

### Управление воркерами

```bash
# Перезапустить все воркеры
sudo supervisorctl restart all

# Перезапустить конкретную группу
sudo supervisorctl restart laravel-worker-high:*

# Проверить статус
sudo supervisorctl status

# Посмотреть логи конкретного воркера
sudo tail -f /var/www/autoshop/storage/logs/worker-default.log
```

## Nginx Cache

### Базовая настройка Nginx для кеширования

```nginx
# /etc/nginx/sites-available/autoshop

upstream php-fpm {
    server unix:/var/run/php/php8.3-fpm.sock;
}

# Параметры кеширования
proxy_cache_path /var/cache/nginx/fastcgi levels=1:2 keys_zone=fastcgi_cache:100m inactive=60m max_size=1g;

server {
    listen 80;
    server_name autoshop.ru www.autoshop.ru;

    root /var/www/autoshop/public;

    index index.php;

    # Кеширование статических файлов
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # FastCGI кеширование
    location ~ \.php$ {
        fastcgi_pass php-fpm;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # Настройки кеширования
        fastcgi_cache fastcgi_cache;
        fastcgi_cache_valid 200 60m;
        fastcgi_cache_valid 301 1h;
        fastcgi_cache_valid 404 1m;
        fastcgi_cache_bypass $http_pragma $http_authorization;
        fastcgi_no_cache $http_pragma $http_authorization;

        # Уникальный ключ кеширования
        fastcgi_cache_key "$scheme$request_method$host$request_uri";

        # Добавить заголовки кеширования
        add_header X-Cache-Status $upstream_cache_status;

        # Не кешировать POST запросы и авторизованных пользователей
        if ($request_method = POST) {
            set $skip_cache 1;
        }

        if ($http_cookie ~* "laravel_session|auth") {
            set $skip_cache 1;
        }

        fastcgi_cache_bypass $skip_cache;
        fastcgi_no_cache $skip_cache;
    }

    # Запрет доступа к скрытым файлам
    location ~ /\. {
        deny all;
    }

    # Laravel
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Gzip сжатие
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json;
}
```

### Microcaching для динамического контента

```nginx
# Добавить в location ~ \.php$
fastcgi_cache microcache;

# Определить microcache
proxy_cache_path /var/cache/nginx/microcache levels=1:2 keys_zone=microcache:10m inactive=5m max_size=100m;

# Короткое время кеширования (5 секунд)
fastcgi_cache_valid 200 5s;
```

## Оптимизация базы данных

### Индексы для основных запросов

```sql
-- Таблица parts
ALTER TABLE parts ADD INDEX idx_search (name, brand, category);
ALTER TABLE parts ADD INDEX idx_price (price);
ALTER TABLE parts ADD INDEX idx_active (is_active, created_at);

-- Таблица search_logs
ALTER TABLE search_logs ADD INDEX idx_query_date (query, created_at);
ALTER TABLE search_logs ADD INDEX idx_popularity (results_count, created_at);

-- Таблица orders
ALTER TABLE orders ADD INDEX idx_status_date (status, created_at);
ALTER TABLE orders ADD INDEX idx_user (user_id);
```

### Оптимизация конфигурации MySQL

```ini
# /etc/mysql/mysql.conf.d/mysqld.cnf

[mysqld]
# Размер буфера InnoDB
innodb_buffer_pool_size = 1G

# Размер лог-файла
innodb_log_file_size = 256M

# Количество потоков
innodb_read_io_threads = 4
innodb_write_io_threads = 4

# Кеширование запросов
query_cache_size = 64M
query_cache_type = 1

# Соединения
max_connections = 500

# Таблицы
table_open_cache = 2000
```

### Перезапуск MySQL

```bash
sudo systemctl restart mysql
```

## Оптимизация PHP-FPM

### Конфигурация пула

```ini
# /etc/php/8.3/fpm/pool.d/www.conf

; Количество дочерних процессов
pm = dynamic

; Максимальное количество процессов
pm.max_children = 50

; Минимальное количество процессов при запуске
pm.start_servers = 10

; Минимальное количество простаивающих процессов
pm.min_spare_servers = 5

; Максимальное количество простаивающих процессов
pm.max_spare_servers = 15

; Максимальное запросов перед перезапуском
pm.max_requests = 1000

; Лимит памяти для каждого процесса
php_admin_value[memory_limit] = 256M

; Время выполнения скрипта
php_admin_value[max_execution_time] = 300

; Количество процессов на пользователя
pm.max_requests = 500
```

### Перезапуск PHP-FPM

```bash
sudo systemctl restart php8.3-fpm
```

## Оптимизация Laravel

### Кеширование конфигурации и маршрутов

```bash
# Кеширование конфигурации
php artisan config:cache

# Кеширование маршрутов
php artisan route:cache

# Кеширование представлений
php artisan view:cache

# Кеширование событий
php artisan event:cache
```

### Отключение отладки в продакшене

```env
# .env
APP_ENV=production
APP_DEBUG=false
```

### Оптимизация Composer

```bash
# Оптимизировать автозагрузчик
composer dump-autoload --optimize

# Обновить зависимости
composer update --optimize-autoloader
```

## HTTP/2 и TLS

### Включение HTTP/2

```nginx
server {
    listen 443 ssl http2;
    server_name autoshop.ru www.autoshop.ru;

    # SSL сертификат
    ssl_certificate /etc/letsencrypt/live/autoshop.ru/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/autoshop.ru/privkey.pem;

    # Параметры SSL
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # ... остальная конфигурация
}

# Перенаправление HTTP на HTTPS
server {
    listen 80;
    server_name autoshop.ru www.autoshop.ru;

    return 301 https://$server_name$request_uri;
}
```

### Получение SSL сертификата Let's Encrypt

```bash
# Установка Certbot
sudo apt install certbot python3-certbot-nginx

# Получение сертификата
sudo certbot --nginx -d autoshop.ru -d www.autoshop.ru

# Автоматическое продление
sudo certbot renew --dry-run
```

## CDN для статических файлов

### Настройка CDN (например, Cloudflare)

```env
# .env
ASSET_URL=https://cdn.autoshop.ru
```

### Использование в Laravel

```php
// В шаблонах
{{ asset('images/logo.png') }} // https://cdn.autoshop.ru/images/logo.png
```

## Мониторинг производительности

### Laravel Telescope

```bash
# Установка
composer require laravel/telescope

# Публикация
php artisan telescope:install
php artisan migrate

# Публикация ресурсов
php artisan telescope:publish
```

### Laravel Debugbar (только для разработки)

```bash
composer require barryvdh/laravel-debugbar --dev
```

### Бенчмаркинг с Apache Bench

```bash
# Установка
sudo apt install apache2-utils

# Тестирование главной страницы
ab -n 1000 -c 10 https://autoshop.ru/

# Тестирование страницы товара
ab -n 500 -c 10 https://autoshop.ru/parts/123
```

### Мониторинг с htop

```bash
# Установка
sudo apt install htop

# Запуск
htop
```

## Консольные команды

```bash
# Очистить кеширование
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear

# Кеширование для продакшена
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Оптимизация Composer
composer dump-autoload --optimize

# Проверка статуса Redis
redis-cli info
redis-cli info stats

# Проверка очередей
php artisan queue:monitor redis:default,redis:high,redis:low --max=100

# Проверка логов воркеров
tail -f /var/www/autoshop/storage/logs/worker.log

# Перезапуск сервисов
sudo systemctl restart nginx
sudo systemctl restart php8.3-fpm
sudo systemctl restart redis
sudo systemctl restart mysql
```

## Git

```bash
git add .
git commit -m "feat: оптимизация производительности: Redis, очереди, Nginx cache"
git push
```

# ЭТАП 25. Развёртывание

## Серверные требования

- Ubuntu 22.04 LTS
- PHP 8.3
- Nginx
- MySQL 8.0
- Redis
- Supervisor
- Composer 2.x

## Установка ПО на сервере

### Обновление системы

```bash
sudo apt update
sudo apt upgrade -y
```

### Установка Nginx

```bash
sudo apt install nginx -y

# Запуск Nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Проверка статуса
sudo systemctl status nginx
```

### Установка PHP 8.3

```bash
# Добавление репозитория PHP
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

# Установка PHP и расширений
sudo apt install php8.3 php8.3-fpm php8.3-cli php8.3-common \
    php8.3-mysql php8.3-redis php8.3-xml php8.3-curl \
    php8.3-mbstring php8.3-zip php8.3-gd php8.3-bcmath \
    php8.3-intl php8.3-imagick -y

# Проверка версии PHP
php -v
```

### Установка MySQL

```bash
sudo apt install mysql-server -y

# Запуск MySQL
sudo systemctl start mysql
sudo systemctl enable mysql

# Безопасная настройка
sudo mysql_secure_installation

# Создание базы данных
sudo mysql -u root -p

mysql> CREATE DATABASE autoshop CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
mysql> CREATE USER 'autoshop_user'@'localhost' IDENTIFIED BY 'your_secure_password';
mysql> GRANT ALL PRIVILEGES ON autoshop.* TO 'autoshop_user'@'localhost';
mysql> FLUSH PRIVILEGES;
mysql> EXIT;
```

### Установка Redis

```bash
sudo apt install redis-server -y

# Настройка Redis
sudo nano /etc/redis/redis.conf
# Раскомментировать: maxmemory 256mb
# Раскомментировать: maxmemory-policy allkeys-lru

# Перезапуск
sudo systemctl start redis
sudo systemctl enable redis
sudo systemctl restart redis
```

### Установка Composer

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
sudo chmod +x /usr/local/bin/composer

# Проверка версии
composer --version
```

### Установка Supervisor

```bash
sudo apt install supervisor -y

# Запуск Supervisor
sudo systemctl start supervisor
sudo systemctl enable supervisor
```

### Установка Git

```bash
sudo apt install git -y
```

## Настройка проекта

### Создание директории проекта

```bash
sudo mkdir -p /var/www/autoshop
sudo chown -R www-data:www-data /var/www/autoshop
```

### Клонирование репозитория

```bash
# Переключиться на пользователя www-data
sudo -u www-data -i

cd /var/www/autoshop

# Клонировать репозиторий
git clone git@github.com:ibalar666/autoshop.git .
```

### Установка зависимостей

```bash
cd /var/www/autoshop

# Установка composer зависимостей
composer install --optimize-autoloader --no-dev

# Оптимизация composer dump-autoload
composer dump-autoload --optimize
```

### Настройка переменных окружения

```bash
# Копирование файла .env
cp .env.example .env

# Редактирование .env
nano .env
```

### Пример файла .env

```env
APP_NAME="АвтоШоп"
APP_ENV=production
APP_KEY=base64:ваш_app_key
APP_DEBUG=false
APP_URL=https://autoshop.ru

LOG_CHANNEL=stack
LOG_LEVEL=warning

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=autoshop
DB_USERNAME=autoshop_user
DB_PASSWORD=your_secure_password

BROADCAST_DRIVER=log
CACHE_DRIVER=redis
FILESYSTEM_DISK=local
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
SESSION_LIFETIME=120

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS="noreply@autoshop.ru"
MAIL_FROM_NAME="${APP_NAME}"
```

### Генерация APP_KEY

```bash
php artisan key:generate
```

### Запуск миграций

```bash
php artisan migrate --force
```

### Кеширование конфигурации

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
```

### Установка прав доступа

```bash
# Директории storage и bootstrap/cache должны быть доступны для записи
sudo chmod -R 775 storage bootstrap/cache

# В некоторых случаях
sudo chmod -R 777 storage bootstrap/cache

# Смена владельца
sudo chown -R www-data:www-data storage bootstrap/cache
```

## Настройка Nginx

### Конфигурационный файл

```bash
sudo nano /etc/nginx/sites-available/autoshop
```

### Пример конфигурации Nginx

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name autoshop.ru www.autoshop.ru;

    root /var/www/autoshop/public;

    index index.php index.html index.htm;

    # Логи
    access_log /var/log/nginx/autoshop_access.log;
    error_log /var/log/nginx/autoshop_error.log;

    # Максимальный размер загружаемого файла
    client_max_body_size 20M;

    # Кеширование статических файлов
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # PHP-FPM
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;

        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;

        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        fastcgi_buffer_size 128k;
        fastcgi_buffers 256 16k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
        fastcgi_read_timeout 240;
    }

    # Запрет доступа к скрытым файлам
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Laravel
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Gzip сжатие
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript
               application/x-javascript application/xml+rss application/json;
}
```

### Активация сайта

```bash
# Создание символической ссылки
sudo ln -s /etc/nginx/sites-available/autoshop /etc/nginx/sites-enabled/

# Проверка конфигурации
sudo nginx -t

# Перезагрузка Nginx
sudo systemctl reload nginx
```

## Настройка Supervisor

### Конфигурация для Laravel workers

```bash
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```

### Пример конфигурации Supervisor

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/autoshop/artisan queue:work redis --sleep=1 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=4
redirect_stderr=true
stdout_logfile=/var/www/autoshop/storage/logs/worker.log
stopwaitsecs=3600
```

### Управление Supervisor

```bash
# Обновление конфигурации
sudo supervisorctl reread
sudo supervisorctl update

# Запуск workers
sudo supervisorctl start laravel-worker:*

# Проверка статуса
sudo supervisorctl status

# Перезапуск workers
sudo supervisorctl restart laravel-worker:*

# Просмотр логов
sudo tail -f /var/www/autoshop/storage/logs/worker.log
```

## SSL сертификат (Let's Encrypt)

### Установка Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Получение сертификата

```bash
sudo certbot --nginx -d autoshop.ru -d www.autoshop.ru
```

### Автоматическое продление

```bash
sudo certbot renew --dry-run
```

### Cron для автопродления

```bash
sudo crontab -e

# Добавить строку:
0 0 * * * certbot renew --quiet
```

## Настройка планировщика (Cron)

### Добавление задачи в crontab

```bash
sudo crontab -e -u www-data
```

### Пример cron задач

```bash
# Выполнять schedule каждую минуту
* * * * * cd /var/www/autoshop && php artisan schedule:run >> /dev/null 2>&1

# Обновлять популярные товары каждые 6 часов
0 */6 * * * cd /var/www/autoshop && php artisan popular:update >> /var/log/autoshop/popular.log 2>&1

# Очищать старые логи поиска каждый день в 3 часа ночи
0 3 * * * cd /var/www/autoshop && php artisan search:clear-logs --days=90 >> /var/log/autoshop/cleanup.log 2>&1

# Резервное копирование базы данных каждый день в 2 часа ночи
0 2 * * * cd /var/www/autoshop && php artisan db:backup >> /var/log/autoshop/backup.log 2>&1
```

## Обновление проекта

### Скрипт для обновления deploy.sh

```bash
#!/bin/bash

echo "Начало деплоя..."

# Остановка workers
sudo supervisorctl stop laravel-worker:*

# Переход в директорию проекта
cd /var/www/autoshop

# Сохранение текущей версии
sudo cp -r storage storage_backup

# Получение последней версии
git fetch origin
git reset --hard origin/main

# Установка зависимостей
composer install --optimize-autoloader --no-dev

# Запуск миграций
php artisan migrate --force

# Кеширование конфигурации
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Очистка кеширования
php artisan cache:clear

# Оптимизация composer
composer dump-autoload --optimize

# Запуск workers
sudo supervisorctl start laravel-worker:*

# Перезапуск Nginx
sudo systemctl reload nginx
sudo systemctl reload php8.3-fpm

echo "Деплой завершен!"
```

### Права на выполнение скрипта

```bash
chmod +x deploy.sh
./deploy.sh
```

## Мониторинг и логи

### Логи Laravel

```bash
tail -f /var/www/autoshop/storage/logs/laravel.log
```

### Логи Nginx

```bash
tail -f /var/log/nginx/autoshop_access.log
tail -f /var/log/nginx/autoshop_error.log
```

### Логи PHP-FPM

```bash
tail -f /var/log/php8.3-fpm.log
```

### Логи Supervisor

```bash
sudo tail -f /var/www/autoshop/storage/logs/worker.log
```

## Резервное копирование

### Скрипт бекапа backup.sh

```bash
#!/bin/bash

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/backups/autoshop"

# Создание директории для бекапов
mkdir -p $BACKUP_DIR

# Бекап базы данных
mysqldump -u autoshop_user -p'your_password' autoshop > $BACKUP_DIR/db_$DATE.sql

# Бекап файлов
tar -czf $BACKUP_DIR/files_$DATE.tar.gz /var/www/autoshop/storage

# Удаление бекапов старее 7 дней
find $BACKUP_DIR -type f -mtime +7 -delete

echo "Бекап создан: $BACKUP_DIR/db_$DATE.sql"
```

### Cron для автоматического бекапа

```bash
sudo crontab -e

# Бекап каждый день в 1 час ночи
0 1 * * * /path/to/backup.sh >> /var/log/autoshop/backup.log 2>&1
```

## Оптимизация производительности

### Настройка PHP-FPM

```bash
sudo nano /etc/php/8.3/fpm/pool.d/www.conf
```

```ini
pm = dynamic
pm.max_children = 50
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 15
pm.max_requests = 500
```

### Настройка MySQL

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

```ini
[mysqld]
innodb_buffer_pool_size = 1G
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit = 2
query_cache_size = 64M
max_connections = 500
```

### Перезапуск сервисов

```bash
sudo systemctl restart php8.3-fpm
sudo systemctl restart mysql
```

## Проверка работоспособности

```bash
# Проверка PHP-FPM
php -v
php artisan --version

# Проверка Nginx
curl -I http://localhost

# Проверка MySQL
mysql -u autoshop_user -p -e "SELECT 1;"

# Проверка Redis
redis-cli ping

# Проверка очередей
php artisan queue:work --once
```

## Консольные команды

```bash
# Статус всех сервисов
sudo systemctl status nginx
sudo systemctl status php8.3-fpm
sudo systemctl status mysql
sudo systemctl status redis
sudo systemctl status supervisor

# Перезапуск всех сервисов
sudo systemctl restart nginx
sudo systemctl restart php8.3-fpm
sudo systemctl restart mysql
sudo systemctl restart redis
sudo systemctl restart supervisor

# Логи
journalctl -u nginx -f
journalctl -u php8.3-fpm -f
```

## Git

```bash
# Клонирование репозитория
git clone git@github.com:ibalar666/autoshop.git /var/www/autoshop

# Добавление origin
git remote add origin git@github.com:ibalar666/autoshop.git

# Получение изменений
git pull origin main

# Проверка статуса
git status
```

## Firewall (UFW)

```bash
# Включение firewall
sudo ufw enable

# Разрешение SSH
sudo ufw allow OpenSSH

# Разрешение HTTP и HTTPS
sudo ufw allow 'Nginx Full'

# Проверка статуса
sudo ufw status
```

## Git

```bash
git add .
git commit -m "docs: добавлена документация по развёртыванию"
git push
```

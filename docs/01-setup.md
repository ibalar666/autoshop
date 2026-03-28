# Установка Laravel

## ЭТАП 1. Создание проекта Laravel

```bash
composer create-project laravel/laravel autoshop
cd autoshop
```

## Настройка окружения

1. Копирование .env.example:

```bash
cp .env.example .env
php artisan key:generate
```

2. Настройка .env для MySQL и Redis:

```env
APP_NAME=Autoshop
APP_ENV=local
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=autoshop
DB_USERNAME=root
DB_PASSWORD=

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

3. Создание базы данных:

```bash
mysql -u root -p -e "CREATE DATABASE autoshop CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

## Проверка установки

```bash
php artisan serve
```

## Рекомендуемый стек сервера

- PHP 8.3
- MySQL 8.0
- Redis 6.0+
- Nginx
- Composer 2.x

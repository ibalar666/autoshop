# Быстрый старт

Руководство по установке и запуску проекта Autoshop.

## Требования

Перед началом установки убедитесь, что у вас установлено:

- **PHP 8.3** или выше
- **MySQL 8.0** или выше
- **Redis** 6.0 или выше
- **Composer** 2.0 или выше

## Установка

### Шаг 1: Клонирование репозитория

```bash
git clone https://github.com/ibalar666/autoshop.git
cd autoshop
```

### Шаг 2: Установка зависимостей

```bash
composer install
```

### Шаг 3: Настройка окружения

```bash
cp .env.example .env
php artisan key:generate
```

### Шаг 4: Настройка базы данных

Отредактируйте файл `.env` и укажите параметры подключения:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=autoshop
DB_USERNAME=root
DB_PASSWORD=your_password

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

Создайте базу данных и выполните миграции:

```bash
mysql -u root -p -e "CREATE DATABASE autoshop;"
php artisan migrate
php artisan db:seed
```

### Шаг 5: Настройка API поставщиков

Добавьте в `.env` ключи доступа к API поставщиков:

```env
AUTOOPT_API_KEY=your_api_key_here
AUTOOPT_API_URL=https://api.autoopt.ru/v2
```

## Запуск проекта

### Локальная разработка

```bash
php artisan serve
```

Приложение будет доступно по адресу: `http://localhost:8000`

### Очереди (для фоновых задач)

```bash
php artisan queue:work
```

### Планировщик задач

Добавьте в crontab:

```bash
* * * * * cd /path/to/autoshop && php artisan schedule:run >> /dev/null 2>&1
```

## Полезные команды

### Очистка кэша

```bash
php artisan cache:clear
php artisan config:clear
php artisan route:clear
```

### Проверка установки

Откройте в браузере:

- `http://localhost:8000` — основное приложение
- `http://localhost:8000/admin` — панель управления MoonShine

### Устранение неполадок

#### Ошибка подключения к Redis

Проверьте, что Redis запущен:

```bash
redis-cli ping
```

Должен вернуть `PONG`.

#### Ошибка прав доступа

```bash
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```

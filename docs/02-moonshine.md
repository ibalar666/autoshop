# Установка MoonShine 4

## ЭТАП 2. Установка MoonShine 4

```bash
composer require moonshine/moonshine
php artisan moonshine:install
```

## Создание администратора

```bash
php artisan moonshine:user
```

## Настройка MoonShine

Публикация конфигурации:

```bash
php artisan vendor:publish --provider="MoonShine\Providers\MoonShineServiceProvider" --tag=config
```

## Структура админ-панели

```
app/MoonShine/
├── Resources/           # CRUD ресурсы
├── Pages/              # Кастомные страницы
└── Forms/              # Кастомные формы
```

## Доступ к админке

URL: `/admin`

## Кастомизация

Логотип, цвета, меню — в конфиге `config/moonshine.php`

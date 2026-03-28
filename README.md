# Autoshop

Проект интеграции с API поставщиков автозапчастей (Autoopt). Платформа для поиска, сравнения и заказа автозапчастей через единый интерфейс.

## Стек технологий

- **PHP 8.3** — основной язык разработки
- **Laravel 11** — PHP-фреймворк
- **MoonShine 4** — административная панель
- **MySQL 8.0** — основная база данных
- **Redis** — кэширование и очереди
- **Composer** — управление зависимостями

## Архитектура проекта

```
User Request
     ↓
Controllers (HTTP Layer)
     ↓
Services (Business Logic)
     ↓
DTO (Data Transfer Objects)
     ↓
Suppliers API (External APIs)
     ↓
Cache / Database
```

### Слои архитектуры

| Слой | Назначение | Примеры |
|------|------------|---------|
| **Controllers** | HTTP-обработка, валидация запросов | `ProductController`, `OrderController` |
| **Services** | Бизнес-логика, координация | `ProductService`, `SearchService` |
| **DTO** | Передача данных между слоями | `ProductDTO`, `SearchRequestDTO` |
| **Suppliers API** | Интеграция с внешними API | `AutooptApiClient`, `SupplierClient` |

## Структура проекта

```
autoshop/
├── app/
│   ├── Http/
│   │   └── Controllers/      # Контроллеры
│   ├── Services/             # Бизнес-логика
│   ├── DTO/                  # Data Transfer Objects
│   ├── Clients/              # HTTP-клиенты для API поставщиков
│   └── Models/               # Eloquent модели
├── config/                   # Конфигурации
├── database/
│   ├── migrations/           # Миграции БД
│   └── seeders/              # Сидеры
├── docs/                     # Документация
│   ├── quick-start.md        # Быстрый старт
│   └── architecture.md       # Архитектура проекта
├── routes/                   # Маршруты
├── resources/                # Views, assets
├── storage/                  # Логи, кэш, файлы
└── tests/                    # Тесты
```

## Документация

- [Быстрый старт](docs/quick-start.md) — установка и запуск проекта
- [Архитектура](docs/architecture.md) — описание архитектуры и принципов

## Спецификация API

- [Спецификация API Autoopt](https://api.autoopt.ru/v2/docs) — документация внешнего API поставщика

## Лицензия

MIT

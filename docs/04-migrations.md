# Миграции базы данных

## ЭТАП 4. Миграции

## Структура таблиц

### categories
- `id` - первичный ключ (bigIncrements)
- `name` - название категории (string)
- `slug` - URL-слаг (string, unique)
- `description` - описание (text, nullable)
- `image` - путь к изображению (string, nullable)
- `parent_id` - ID родительской категории (unsignedBigInteger, nullable)
- `is_active` - активность категории (boolean, default: true)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### brands
- `id` - первичный ключ (bigIncrements)
- `name` - название бренда (string)
- `slug` - URL-слаг (string, unique)
- `logo` - путь к логотипу (string, nullable)
- `is_active` - активность бренда (boolean, default: true)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### part_cache
- `id` - первичный ключ (bigIncrements)
- `article` - артикул запчасти (string)
- `brand` - название бренда (string)
- `name` - название запчасти (string)
- `data_json` - JSON данные о запчасти (json)
- `expires_at` - срок действия кэша (timestamp)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### carts
- `id` - первичный ключ (bigIncrements)
- `user_id` - ID пользователя (unsignedBigInteger, nullable)
- `session_id` - ID сессии (string, nullable)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### cart_items
- `id` - первичный ключ (bigIncrements)
- `cart_id` - ID корзины (unsignedBigInteger)
- `article` - артикул запчасти (string)
- `brand` - название бренда (string)
- `price` - цена (decimal, 10, 2)
- `quantity` - количество (integer)
- `supplier` - поставщик (string)
- `delivery_days` - срок доставки в днях (integer)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### orders
- `id` - первичный ключ (bigIncrements)
- `user_id` - ID пользователя (unsignedBigInteger, nullable)
- `status` - статус заказа (string)
- `total` - общая сумма (decimal, 10, 2)
- `customer_name` - имя клиента (string)
- `phone` - телефон клиента (string)
- `email` - email клиента (string, nullable)
- `comment` - комментарий (text, nullable)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### order_items
- `id` - первичный ключ (bigIncrements)
- `order_id` - ID заказа (unsignedBigInteger)
- `article` - артикул запчасти (string)
- `brand` - название бренда (string)
- `name` - название запчасти (string)
- `price` - цена (decimal, 10, 2)
- `quantity` - количество (integer)
- `supplier` - поставщик (string)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### markups
- `id` - первичный ключ (bigIncrements)
- `brand` - название бренда (string, nullable)
- `category` - название категории (string, nullable)
- `percent` - процент наценки (decimal, 5, 2)
- `is_active` - активность наценки (boolean, default: true)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### search_logs
- `id` - первичный ключ (bigIncrements)
- `query` - поисковый запрос (string)
- `results_count` - количество результатов (integer)
- `ip` - IP адрес клиента (string)
- `user_agent` - User-Agent браузера (string, nullable)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### seo_pages
- `id` - первичный ключ (bigIncrements)
- `url` - URL страницы (string, unique)
- `title` - мета-тег title (string)
- `description` - мета-тег description (string, nullable)
- `h1` - заголовок H1 (string)
- `content` - контент страницы (text, nullable)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

### api_settings
- `id` - первичный ключ (bigIncrements)
- `supplier` - название поставщика (string, unique)
- `api_key` - API ключ (string)
- `api_url` - URL API (string)
- `is_active` - активность настройки (boolean, default: true)
- `created_at`, `updated_at` - временные метки
- `deleted_at` - мягкое удаление (timestamp, nullable)

## Пример миграции: create_categories_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('categories', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('slug')->unique();
            $table->text('description')->nullable();
            $table->string('image')->nullable();
            $table->unsignedBigInteger('parent_id')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('slug');
            $table->index('parent_id');
            $table->index('is_active');

            // Внешний ключ
            $table->foreign('parent_id')
                  ->references('id')
                  ->on('categories')
                  ->nullOnDelete();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('categories');
    }
};
```

## Пример миграции: create_brands_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('brands', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('slug')->unique();
            $table->string('logo')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('slug');
            $table->index('is_active');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('brands');
    }
};
```

## Пример миграции: create_part_cache_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('part_cache', function (Blueprint $table) {
            $table->id();
            $table->string('article');
            $table->string('brand');
            $table->string('name');
            $table->json('data_json');
            $table->timestamp('expires_at');
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index(['article', 'brand']);
            $table->index('expires_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('part_cache');
    }
};
```

## Пример миграции: create_carts_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('carts', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('user_id')->nullable();
            $table->string('session_id')->nullable();
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('user_id');
            $table->index('session_id');

            // Внешний ключ
            $table->foreign('user_id')
                  ->references('id')
                  ->on('users')
                  ->nullOnDelete();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('carts');
    }
};
```

## Пример миграции: create_cart_items_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('cart_items', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('cart_id');
            $table->string('article');
            $table->string('brand');
            $table->decimal('price', 10, 2);
            $table->integer('quantity')->default(1);
            $table->string('supplier');
            $table->integer('delivery_days')->default(0);
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('cart_id');
            $table->index(['article', 'brand']);

            // Внешний ключ
            $table->foreign('cart_id')
                  ->references('id')
                  ->on('carts')
                  ->cascadeOnDelete();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('cart_items');
    }
};
```

## Пример миграции: create_orders_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('user_id')->nullable();
            $table->string('status')->default('pending');
            $table->decimal('total', 10, 2);
            $table->string('customer_name');
            $table->string('phone');
            $table->string('email')->nullable();
            $table->text('comment')->nullable();
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('user_id');
            $table->index('status');
            $table->index('created_at');

            // Внешний ключ
            $table->foreign('user_id')
                  ->references('id')
                  ->on('users')
                  ->nullOnDelete();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};
```

## Пример миграции: create_order_items_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('order_items', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('order_id');
            $table->string('article');
            $table->string('brand');
            $table->string('name');
            $table->decimal('price', 10, 2);
            $table->integer('quantity');
            $table->string('supplier');
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('order_id');
            $table->index(['article', 'brand']);

            // Внешний ключ
            $table->foreign('order_id')
                  ->references('id')
                  ->on('orders')
                  ->cascadeOnDelete();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('order_items');
    }
};
```

## Пример миграции: create_markups_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('markups', function (Blueprint $table) {
            $table->id();
            $table->string('brand')->nullable();
            $table->string('category')->nullable();
            $table->decimal('percent', 5, 2);
            $table->boolean('is_active')->default(true);
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('brand');
            $table->index('category');
            $table->index('is_active');
            $table->index(['brand', 'category']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('markups');
    }
};
```

## Пример миграции: create_search_logs_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('search_logs', function (Blueprint $table) {
            $table->id();
            $table->string('query');
            $table->integer('results_count')->default(0);
            $table->string('ip');
            $table->text('user_agent')->nullable();
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('query');
            $table->index('ip');
            $table->index('created_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('search_logs');
    }
};
```

## Пример миграции: create_seo_pages_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('seo_pages', function (Blueprint $table) {
            $table->id();
            $table->string('url')->unique();
            $table->string('title');
            $table->string('description')->nullable();
            $table->string('h1');
            $table->text('content')->nullable();
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('url');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('seo_pages');
    }
};
```

## Пример миграции: create_api_settings_table

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('api_settings', function (Blueprint $table) {
            $table->id();
            $table->string('supplier')->unique();
            $table->string('api_key');
            $table->string('api_url');
            $table->boolean('is_active')->default(true);
            $table->timestamps();
            $table->softDeletes();

            // Индексы
            $table->index('supplier');
            $table->index('is_active');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('api_settings');
    }
};
```

## Выполнение миграций

Для создания всех таблиц в базе данных:

```bash
php artisan migrate
```

Для отката последней миграции:

```bash
php artisan migrate:rollback
```

Для отката всех миграций и повторного выполнения:

```bash
php artisan migrate:refresh
```

Для сброса всех таблиц и повторного выполнения всех миграций:

```bash
php artisan migrate:fresh
```

## Рекомендации по индексам

1. **Foreign Keys**: Все внешние ключи должны быть проиндексированы
2. **Search Fields**: Поля, по которым часто ищут (slug, email, article и т.д.)
3. **Composite Indexes**: Для запросов с несколькими условиями (например, article + brand)
4. **Status Fields**: Индексы на поля статусов (is_active, status)
5. **Timestamps**: Индексы на created_at для сортировки по дате

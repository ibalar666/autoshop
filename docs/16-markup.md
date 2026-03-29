# ЭТАП 16. Наценки

## Таблица markups

```sql
CREATE TABLE markups (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    brand VARCHAR(100) NOT NULL,
    category VARCHAR(100) NOT NULL,
    percent DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_brand (brand),
    INDEX idx_category (category),
    UNIQUE KEY unique_brand_category (brand, category)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Примеры наценок

```sql
INSERT INTO markups (brand, category, percent) VALUES
('Bosch', 'Фильтры', 15.00),
('Bosch', 'Свечи', 20.00),
('Valeo', 'Генераторы', 25.00),
('Valeo', 'Стартеры', 25.00),
('Mann', 'Фильтры', 18.00),
('Sachs', 'Амортизаторы', 30.00),
('*', '*', 20.00);  -- Наценка по умолчанию
```

## PriceService

```php
<?php

namespace App\Services;

use App\Models\Markup;
use App\Models\Part;

class PriceService
{
    /**
     * Получить цену с наценкой
     */
    public function getPriceWithMarkup(float $basePrice, ?string $brand = null, ?string $category = null): float
    {
        $markup = $this->getMarkup($brand, $category);
        return $basePrice * (1 + $markup / 100);
    }

    /**
     * Получить наценку для бренда и категории
     */
    public function getMarkup(?string $brand, ?string $category): float
    {
        // Ищем конкретную наценку
        if ($brand && $category) {
            $markup = Markup::where('brand', $brand)
                ->where('category', $category)
                ->first();
            if ($markup) {
                return (float) $markup->percent;
            }
        }

        // Ищем наценку по бренду
        if ($brand) {
            $markup = Markup::where('brand', $brand)
                ->where('category', '*')
                ->first();
            if ($markup) {
                return (float) $markup->percent;
            }
        }

        // Ищем наценку по категории
        if ($category) {
            $markup = Markup::where('brand', '*')
                ->where('category', $category)
                ->first();
            if ($markup) {
                return (float) $markup->percent;
            }
        }

        // Наценка по умолчанию
        $markup = Markup::where('brand', '*')
            ->where('category', '*')
            ->first();

        return $markup ? (float) $markup->percent : 20.00;
    }

    /**
     * Обновить цены всех товаров с учетом наценок
     */
    public function updateAllPrices(): int
    {
        $parts = Part::whereNotNull('base_price')->get();
        $updated = 0;

        foreach ($parts as $part) {
            $newPrice = $this->getPriceWithMarkup(
                $part->base_price,
                $part->brand,
                $part->category
            );

            if ($part->price != $newPrice) {
                $part->price = $newPrice;
                $part->save();
                $updated++;
            }
        }

        return $updated;
    }

    /**
     * Пересчитать цену конкретного товара
     */
    public function recalculatePrice(Part $part): void
    {
        $part->price = $this->getPriceWithMarkup(
            $part->base_price ?? $part->price,
            $part->brand,
            $part->category
        );
        $part->save();
    }
}
```

## Использование в контроллерах

```php
<?php

namespace App\Http\Controllers;

use App\Services\PriceService;
use App\Models\Part;

class PartController extends Controller
{
    protected PriceService $priceService;

    public function __construct(PriceService $priceService)
    {
        $this->priceService = $priceService;
    }

    /**
     * Отобразить товар с учётом наценки
     */
    public function show(Part $part)
    {
        $priceWithMarkup = $this->priceService->getPriceWithMarkup(
            $part->base_price ?? $part->price,
            $part->brand,
            $part->category
        );

        return view('parts.show', compact('part', 'priceWithMarkup'));
    }
}
```

## Artisan команды

### Обновление цен через artisan команду

```php
<?php

namespace App\Console\Commands;

use App\Services\PriceService;
use Illuminate\Console\Command;

class UpdatePricesCommand extends Command
{
    protected $signature = 'prices:update {--all : Обновить все цены}';
    protected $description = 'Пересчитать цены с учётом наценок';

    public function __construct(
        protected PriceService $priceService
    ) {
        parent::__construct();
    }

    public function handle()
    {
        if ($this->option('all')) {
            $updated = $this->priceService->updateAllPrices();
            $this->info("Обновлено цен: {$updated}");
        } else {
            $this->error('Укажите опцию --all для обновления всех цен');
            return 1;
        }

        return 0;
    }
}
```

### Регистрация команды

В `app/Console/Kernel.php`:

```php
protected $commands = [
    \App\Console\Commands\UpdatePricesCommand::class,
];
```

### Запуск команды

```bash
php artisan prices:update --all
```

## Миграция для добавления base_price

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::table('parts', function (Blueprint $table) {
            $table->decimal('base_price', 10, 2)->nullable()->after('price');
        });

        // Скопируем текущие цены в base_price
        DB::statement('UPDATE parts SET base_price = price WHERE base_price IS NULL');
    }

    public function down()
    {
        Schema::table('parts', function (Blueprint $table) {
            $table->dropColumn('base_price');
        });
    }
};
```

## Консольные команды

```bash
# Создать миграцию
php artisan make:migration add_base_price_to_parts_table

# Запустить миграцию
php artisan migrate

# Создать команду
php artisan make:command UpdatePricesCommand

# Обновить цены
php artisan prices:update --all
```

## Git

```bash
git add .
git commit -m "feat: добавлена система наценок"
git push
```

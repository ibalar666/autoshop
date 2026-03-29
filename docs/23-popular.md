# ЭТАП 23. Популярные товары

## Таблица parts_cache

```sql
CREATE TABLE parts_cache (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    part_id BIGINT UNSIGNED NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    brand VARCHAR(100) NOT NULL,
    category VARCHAR(100) NULL,
    price DECIMAL(10,2) NOT NULL,
    discount DECIMAL(3,2) DEFAULT 0.00,
    main_image VARCHAR(255) NULL,
    slug VARCHAR(255) NULL,
    views INT UNSIGNED DEFAULT 0,
    sales_count INT UNSIGNED DEFAULT 0,
    popularity_score DECIMAL(10,2) DEFAULT 0.00,
    is_popular TINYINT(1) DEFAULT 0,
    is_new TINYINT(1) DEFAULT 0,
    is_sale TINYINT(1) DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_popularity (popularity_score DESC),
    INDEX idx_views (views DESC),
    INDEX idx_sales (sales_count DESC),
    INDEX idx_is_popular (is_popular),
    INDEX idx_is_new (is_new),
    INDEX idx_is_sale (is_sale),
    INDEX idx_brand (brand),
    INDEX idx_category (category),
    FOREIGN KEY (part_id) REFERENCES parts(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Модель PartCache

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class PartCache extends Model
{
    protected $table = 'parts_cache';

    protected $fillable = [
        'part_id',
        'name',
        'brand',
        'category',
        'price',
        'discount',
        'main_image',
        'slug',
        'views',
        'sales_count',
        'popularity_score',
        'is_popular',
        'is_new',
        'is_sale',
    ];

    protected $casts = [
        'price' => 'decimal:2',
        'discount' => 'decimal:2',
        'popularity_score' => 'decimal:2',
        'is_popular' => 'boolean',
        'is_new' => 'boolean',
        'is_sale' => 'boolean',
        'updated_at' => 'datetime',
    ];

    /**
     * Связь с оригинальным товаром
     */
    public function part(): BelongsTo
    {
        return $this->belongsTo(Part::class);
    }

    /**
     * Популярные товары
     */
    public static function getPopular(int $limit = 20)
    {
        return static::where('is_popular', true)
            ->orderBy('popularity_score', 'desc')
            ->limit($limit)
            ->get();
    }

    /**
     * Новые товары
     */
    public static function getNew(int $limit = 20)
    {
        return static::where('is_new', true)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }

    /**
     * Товары со скидкой
     */
    public static function getSale(int $limit = 20)
    {
        return static::where('is_sale', true)
            ->where('discount', '>', 0)
            ->orderBy('discount', 'desc')
            ->limit($limit)
            ->get();
    }

    /**
     * Популярные в категории
     */
    public static function getPopularInCategory(string $category, int $limit = 10)
    {
        return static::where('category', $category)
            ->where('is_popular', true)
            ->orderBy('popularity_score', 'desc')
            ->limit($limit)
            ->get();
    }

    /**
     * Популярные по бренду
     */
    public static function getPopularByBrand(string $brand, int $limit = 10)
    {
        return static::where('brand', $brand)
            ->where('is_popular', true)
            ->orderBy('popularity_score', 'desc')
            ->limit($limit)
            ->get();
    }
}
```

## PopularPartsService

```php
<?php

namespace App\Services;

use App\Models\Part;
use App\Models\PartCache;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;

class PopularPartsService
{
    /**
     * Обновить кеширование товаров
     */
    public function updateCache(): int
    {
        Log::info('Начало обновления кеширования товаров');

        DB::beginTransaction();
        try {
            // Получаем все активные товары
            $parts = Part::where('is_active', true)
                ->with('category')
                ->get();

            $updated = 0;
            foreach ($parts as $part) {
                $cacheData = [
                    'name' => $part->name,
                    'brand' => $part->brand,
                    'category' => $part->category ? $part->category->name : null,
                    'price' => $part->price,
                    'discount' => $part->discount ?? 0,
                    'main_image' => $part->main_image,
                    'slug' => $part->slug,
                    'views' => $part->views ?? 0,
                    'sales_count' => $part->sales_count ?? 0,
                ];

                // Рассчитываем популярность
                $popularityScore = $this->calculatePopularityScore(
                    $cacheData['views'],
                    $cacheData['sales_count'],
                    $part->created_at
                );
                $cacheData['popularity_score'] = $popularityScore;

                // Определяем флаги
                $cacheData['is_popular'] = $popularityScore > 50;
                $cacheData['is_new'] = $part->created_at->diffInDays(now()) <= 30;
                $cacheData['is_sale'] = ($cacheData['discount'] ?? 0) > 0;

                PartCache::updateOrCreate(
                    ['part_id' => $part->id],
                    $cacheData
                );

                $updated++;
            }

            DB::commit();
            Log::info('Обновление кеширования товаров завершено', ['updated' => $updated]);

            // Очищаем кеширование
            $this->clearCache();

            return $updated;
        } catch (\Exception $e) {
            DB::rollBack();
            Log::error('Ошибка обновления кеширования товаров', ['error' => $e->getMessage()]);
            throw $e;
        }
    }

    /**
     * Рассчитать оценку популярности
     */
    protected function calculatePopularityScore(int $views, int $salesCount, $createdAt): float
    {
        // Веса для разных факторов
        $viewsWeight = 0.3;
        $salesWeight = 0.5;
        $freshnessWeight = 0.2;

        // Нормализованные значения (0-100)
        $viewsScore = min($views / 10, 100);
        $salesScore = min($salesCount * 5, 100);

        // Свежесть товара (новые товары получают бонус)
        $daysSinceCreation = $createdAt->diffInDays(now());
        $freshnessScore = max(0, 100 - $daysSinceCreation / 3);

        // Итоговая оценка
        return ($viewsScore * $viewsWeight) +
               ($salesScore * $salesWeight) +
               ($freshnessScore * $freshnessWeight);
    }

    /**
     * Обновить счётчик просмотров
     */
    public function incrementViews(int $partId): void
    {
        try {
            Part::where('id', $partId)->increment('views');
            PartCache::where('part_id', $partId)->increment('views');

            // Очищаем кеширование популярных товаров
            Cache::forget('popular:parts');
            Cache::forget('popular:category:*');
        } catch (\Exception $e) {
            Log::error('Ошибка обновления просмотров', ['part_id' => $partId]);
        }
    }

    /**
     * Обновить счётчик продаж
     */
    public function incrementSales(int $partId, int $quantity = 1): void
    {
        try {
            Part::where('id', $partId)->increment('sales_count', $quantity);
            PartCache::where('part_id', $partId)->increment('sales_count', $quantity);

            // Очищаем кеширование
            $this->clearCache();
        } catch (\Exception $e) {
            Log::error('Ошибка обновления продаж', ['part_id' => $partId]);
        }
    }

    /**
     * Получить популярные товары с кешированием
     */
    public function getPopular(int $limit = 20): \Illuminate\Support\Collection
    {
        return Cache::remember(
            'popular:parts:' . $limit,
            3600, // 1 час
            function () use ($limit) {
                return PartCache::getPopular($limit);
            }
        );
    }

    /**
     * Получить новые товары с кешированием
     */
    public function getNew(int $limit = 20): \Illuminate\Support\Collection
    {
        return Cache::remember(
            'new:parts:' . $limit,
            3600,
            function () use ($limit) {
                return PartCache::getNew($limit);
            }
        );
    }

    /**
     * Получить товары со скидкой с кешированием
     */
    public function getSale(int $limit = 20): \Illuminate\Support\Collection
    {
        return Cache::remember(
            'sale:parts:' . $limit,
            3600,
            function () use ($limit) {
                return PartCache::getSale($limit);
            }
        );
    }

    /**
     * Получить популярные в категории с кешированием
     */
    public function getPopularInCategory(string $category, int $limit = 10): \Illuminate\Support\Collection
    {
        return Cache::remember(
            'popular:category:' . $category . ':' . $limit,
            1800, // 30 минут
            function () use ($category, $limit) {
                return PartCache::getPopularInCategory($category, $limit);
            }
        );
    }

    /**
     * Получить популярные по бренду с кешированием
     */
    public function getPopularByBrand(string $brand, int $limit = 10): \Illuminate\Support\Collection
    {
        return Cache::remember(
            'popular:brand:' . $brand . ':' . $limit,
            1800,
            function () use ($brand, $limit) {
                return PartCache::getPopularByBrand($brand, $limit);
            }
        );
    }

    /**
     * Очистить кеширование
     */
    protected function clearCache(): void
    {
        $redis = Cache::getStore();
        $keys = $redis->connection('cache')->keys('popular:*');
        foreach ($keys as $key) {
            Cache::forget($key);
        }
    }
}
```

## Job для обновления популярности

```php
<?php

namespace App\Jobs;

use App\Services\PopularPartsService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class UpdatePopularPartsJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 600;

    public function __construct()
    {}

    public function handle(PopularPartsService $service): void
    {
        Log::info('Начало обновления популярности товаров');

        try {
            $updated = $service->updateCache();

            Log::info('Обновление популярности товаров завершено', ['updated' => $updated]);
        } catch (\Exception $e) {
            Log::error('Ошибка обновления популярности товаров', ['error' => $e->getMessage()]);
            throw $e;
        }
    }

    public function failed(\Throwable $exception): void
    {
        Log::error('Job обновления популярности завершился с ошибкой', [
            'error' => $exception->getMessage(),
        ]);
    }
}
```

## Observer для автоматического обновления счётчиков

```php
<?php

namespace App\Observers;

use App\Services\PopularPartsService;

class PartObserver
{
    protected PopularPartsService $popularPartsService;

    public function __construct(PopularPartsService $popularPartsService)
    {
        $this->popularPartsService = $popularPartsService;
    }

    /**
     * При просмотре товара (вызывается из контроллера)
     */
    public function viewed($part): void
    {
        $this->popularPartsService->incrementViews($part->id);
    }
}
```

## Использование в контроллерах

### HomeController

```php
<?php

namespace App\Http\Controllers;

use App\Services\PopularPartsService;

class HomeController extends Controller
{
    protected PopularPartsService $popularPartsService;

    public function __construct(PopularPartsService $popularPartsService)
    {
        $this->popularPartsService = $popularPartsService;
    }

    public function index()
    {
        $popularParts = $this->popularPartsService->getPopular(12);
        $newParts = $this->popularPartsService->getNew(12);
        $saleParts = $this->popularPartsService->getSale(8);

        return view('home', compact('popularParts', 'newParts', 'saleParts'));
    }
}
```

### PartController

```php
<?php

namespace App\Http\Controllers;

use App\Services\PopularPartsService;
use App\Models\Part;

class PartController extends Controller
{
    protected PopularPartsService $popularPartsService;

    public function __construct(PopularPartsService $popularPartsService)
    {
        $this->popularPartsService = $popularPartsService;
    }

    public function show(Part $part)
    {
        // Увеличиваем счётчик просмотров
        $this->popularPartsService->incrementViews($part->id);

        // Получаем популярные товары этой категории
        if ($part->category) {
            $popularInCategory = $this->popularPartsService->getPopularInCategory(
                $part->category->name,
                6
            );
        } else {
            $popularInCategory = collect();
        }

        // Получаем популярные товары этого бренда
        $popularByBrand = $this->popularPartsService->getPopularByBrand(
            $part->brand,
            6
        );

        return view('parts.show', compact('part', 'popularInCategory', 'popularByBrand'));
    }
}
```

### OrderController

```php
<?php

namespace App\Http\Controllers;

use App\Services\PopularPartsService;

class OrderController extends Controller
{
    protected PopularPartsService $popularPartsService;

    public function __construct(PopularPartsService $popularPartsService)
    {
        $this->popularPartsService = $popularPartsService;
    }

    public function store(Request $request)
    {
        // ... создание заказа

        // Обновляем счётчики продаж
        foreach ($cart->items() as $item) {
            $this->popularPartsService->incrementSales(
                $item['id'],
                $item['quantity']
            );
        }

        return redirect()->route('checkout.success');
    }
}
```

## Artisan команды

```php
<?php

namespace App\Console\Commands;

use App\Services\PopularPartsService;
use Illuminate\Console\Command;

class UpdatePopularPartsCommand extends Command
{
    protected $signature = 'popular:update';
    protected $description = 'Обновить популярные товары';

    public function __construct(
        protected PopularPartsService $service
    ) {
        parent::__construct();
    }

    public function handle()
    {
        $this->info('Обновление популярности товаров...');

        $updated = $this->service->updateCache();

        $this->info("Обновлено товаров: {$updated}");

        return 0;
    }
}
```

## Cron для автоматического обновления

```bash
# Добавить в crontab
crontab -e

# Обновлять популярность каждые 6 часов
0 */6 * * * cd /var/www/autoshop && php artisan popular:update >> /var/log/autoshop/popular.log 2>&1
```

## Консольные команды

```bash
# Создать миграцию
php artisan make:migration create_parts_cache_table

# Запустить миграцию
php artisan migrate

# Создать модель
php artisan make:model PartCache

# Создать команду
php artisan make:command UpdatePopularPartsCommand

# Обновить популярность вручную
php artisan popular:update

# Отправить job в очередь
php artisan queue:work --once
```

## Git

```bash
git add .
git commit -m "feat: добавлена система популярных товаров с кешированием"
git push
```

# ЭТАП 17. Кеширование Redis

## Установка Redis

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install redis-server

# Запуск Redis
sudo systemctl start redis
sudo systemctl enable redis

# Проверка статуса
sudo systemctl status redis

# Тест соединения
redis-cli ping
# Должно вернуть: PONG
```

## Установка PHP расширения

```bash
# Ubuntu/Debian
sudo apt install php8.3-redis

# Перезапуск PHP-FPM
sudo systemctl restart php8.3-fpm

# Проверка
php -m | grep redis
```

## Настройка Laravel

### Файл .env

```env
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis
```

### Файл config/cache.php

```php
'default' => env('CACHE_DRIVER', 'redis'),

'stores' => [
    'redis' => [
        'driver' => 'redis',
        'connection' => 'cache',
        'lock_connection' => 'default',
    ],
],
```

### Файл config/database.php

```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'phpredis'),

    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', 'laravel_'),
    ],

    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],

    'cache' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_CACHE_DB', '1'),
    ],
],
```

## Кеширование поиска

```php
<?php

namespace App\Services;

use App\Models\Part;
use Illuminate\Support\Facades\Cache;

class SearchService
{
    /**
     * Поиск товаров с кешированием
     */
    public function search(string $query, int $page = 1, int $perPage = 20): array
    {
        $cacheKey = $this->getSearchCacheKey($query, $page, $perPage);

        return Cache::remember($cacheKey, 3600, function () use ($query, $page, $perPage) {
            $searchQuery = Part::where('is_active', true);

            // Поиск по названию
            $searchQuery->where(function ($q) use ($query) {
                $q->where('name', 'like', "%{$query}%")
                  ->orWhere('sku', 'like', "%{$query}%")
                  ->orWhere('brand', 'like', "%{$query}%");
            });

            $parts = $searchQuery->paginate($perPage, ['*'], 'page', $page);

            return [
                'data' => $parts->items(),
                'total' => $parts->total(),
                'page' => $parts->currentPage(),
                'per_page' => $parts->perPage(),
            ];
        });
    }

    /**
     * Ключ кеширования для поиска
     */
    protected function getSearchCacheKey(string $query, int $page, int $perPage): string
    {
        return 'search:' . md5($query) . ':' . $page . ':' . $perPage;
    }

    /**
     * Очистка кеширования поиска
     */
    public function clearSearchCache(?string $query = null): void
    {
        if ($query) {
            $cacheKey = $this->getSearchCacheKey($query, 1, 20);
            Cache::forget($cacheKey);
        } else {
            // Очистить все кеши поиска
            $redis = Cache::getStore();
            $keys = $redis->connection('cache')->keys('search:*');
            foreach ($keys as $key) {
                Cache::forget($key);
            }
        }
    }
}
```

## Кеширование карточек товаров

```php
<?php

namespace App\Services;

use App\Models\Part;
use Illuminate\Support\Facades\Cache;

class PartCacheService
{
    /**
     * Получить товар из кеширования
     */
    public function getPart(int $partId): ?Part
    {
        return Cache::remember(
            "part:{$partId}",
            86400, // 24 часа
            function () use ($partId) {
                return Part::with(['category', 'images'])->find($partId);
            }
        );
    }

    /**
     * Получить связанные товары
     */
    public function getRelatedParts(int $partId, string $brand, string $category, int $limit = 8): array
    {
        return Cache::remember(
            "parts:related:{$partId}",
            3600, // 1 час
            function () use ($partId, $brand, $category, $limit) {
                return Part::where('is_active', true)
                    ->where('id', '!=', $partId)
                    ->where(function ($q) use ($brand, $category) {
                        $q->where('brand', $brand)
                          ->orWhere('category', $category);
                    })
                    ->limit($limit)
                    ->get()
                    ->toArray();
            }
        );
    }

    /**
     * Получить товары категории
     */
    public function getCategoryParts(string $category, int $page = 1, int $perPage = 20): array
    {
        return Cache::remember(
            "category:{$category}:page:{$page}",
            1800, // 30 минут
            function () use ($category, $page, $perPage) {
                $parts = Part::where('category', $category)
                    ->where('is_active', true)
                    ->paginate($perPage, ['*'], 'page', $page);

                return [
                    'data' => $parts->items(),
                    'total' => $parts->total(),
                    'page' => $parts->currentPage(),
                    'per_page' => $parts->perPage(),
                ];
            }
        );
    }

    /**
     * Сбросить кеширование товара
     */
    public function clearPartCache(int $partId): void
    {
        Cache::forget("part:{$partId}");
        Cache::forget("parts:related:{$partId}");
    }

    /**
     * Сбросить кеширование категории
     */
    public function clearCategoryCache(string $category): void
    {
        $redis = Cache::getStore();
        $keys = $redis->connection('cache')->keys("category:{$category}:*");
        foreach ($keys as $key) {
            Cache::forget($key);
        }
    }

    /**
     * Предзагрузка популярных товаров
     */
    public function warmupPopularCache(): int
    {
        $popularParts = Part::where('is_active', true)
            ->orderBy('views', 'desc')
            ->limit(100)
            ->get();

        $warmed = 0;
        foreach ($popularParts as $part) {
            $this->getPart($part->id);
            $warmed++;
        }

        return $warmed;
    }
}
```

## Использование в контроллерах

```php
<?php

namespace App\Http\Controllers;

use App\Services\PartCacheService;
use App\Services\SearchService;
use Illuminate\Http\Request;

class PartController extends Controller
{
    protected PartCacheService $partCacheService;
    protected SearchService $searchService;

    public function __construct(
        PartCacheService $partCacheService,
        SearchService $searchService
    ) {
        $this->partCacheService = $partCacheService;
        $this->searchService = $searchService;
    }

    /**
     * Страница товара
     */
    public function show(int $id)
    {
        $part = $this->partCacheService->getPart($id);

        if (!$part) {
            abort(404);
        }

        $related = $this->partCacheService->getRelatedParts(
            $part->id,
            $part->brand,
            $part->category
        );

        return view('parts.show', compact('part', 'related'));
    }

    /**
     * Поиск товаров
     */
    public function search(Request $request)
    {
        $query = $request->get('q', '');
        $page = $request->get('page', 1);

        if (empty($query)) {
            return redirect()->route('catalog');
        }

        $results = $this->searchService->search($query, $page);

        return view('search.results', compact('query', 'results'));
    }
}
```

## Очистка кеши при обновлении товаров

```php
<?php

namespace App\Observers;

use App\Services\PartCacheService;

class PartObserver
{
    protected PartCacheService $cacheService;

    public function __construct(PartCacheService $cacheService)
    {
        $this->cacheService = $cacheService;
    }

    public function updated($part)
    {
        $this->cacheService->clearPartCache($part->id);
        $this->cacheService->clearCategoryCache($part->category);
    }

    public function deleted($part)
    {
        $this->cacheService->clearPartCache($part->id);
        $this->cacheService->clearCategoryCache($part->category);
    }
}
```

## Artisan команды

### Очистка кеширования

```php
<?php

namespace App\Console\Commands;

use App\Services\PartCacheService;
use App\Services\SearchService;
use Illuminate\Console\Command;

class ClearCacheCommand extends Command
{
    protected $signature = 'cache:clear-all {--type=all : Тип кеширования (all|parts|search)}';
    protected $description = 'Очистка кеширования';

    public function __construct(
        protected PartCacheService $partCacheService,
        protected SearchService $searchService
    ) {
        parent::__construct();
    }

    public function handle()
    {
        $type = $this->option('type');

        switch ($type) {
            case 'all':
                Cache::flush();
                $this->info('Весь кеширование очищен');
                break;

            case 'parts':
                $redis = Cache::getStore();
                $keys = $redis->connection('cache')->keys('part:*');
                foreach ($keys as $key) {
                    Cache::forget($key);
                }
                $this->info('Кеширование товаров очищено');
                break;

            case 'search':
                $this->searchService->clearSearchCache();
                $this->info('Кеширование поиска очищено');
                break;

            default:
                $this->error('Неизвестный тип кеширования');
                return 1;
        }

        return 0;
    }
}
```

### Предзагрузка кеширования

```php
<?php

namespace App\Console\Commands;

use App\Services\PartCacheService;
use Illuminate\Console\Command;

class WarmupCacheCommand extends Command
{
    protected $signature = 'cache:warmup';
    protected $description = 'Предзагрузка популярного кеширования';

    public function __construct(
        protected PartCacheService $cacheService
    ) {
        parent::__construct();
    }

    public function handle()
    {
        $this->info('Предзагрузка кеширования...');

        $warmed = $this->cacheService->warmupPopularCache();

        $this->info("Загружено в кеширование: {$warmed} товаров");

        return 0;
    }
}
```

## Консольные команды

```bash
# Очистить весь кеширование
php artisan cache:clear

# Очистить только кеширование поиска
php artisan cache:clear-all --type=search

# Очистить только кеширование товаров
php artisan cache:clear-all --type=parts

# Предзагрузить популярные товары
php artisan cache:warmup

# Посмотреть кеширование Redis
redis-cli
127.0.0.1:6379> KEYS laravel_cache:*
127.0.0.1:6379> GET laravel_cache:search:abc123:1:20
127.0.0.1:6379> exit
```

## Git

```bash
git add .
git commit -m "feat: добавлено Redis кеширование для поиска и товаров"
git push
```

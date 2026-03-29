# ЭТАП 18. Очереди

## Настройка очередей в Laravel

### Файл .env

```env
QUEUE_CONNECTION=redis
```

### Создание таблицы для failed jobs

```bash
php artisan queue:failed-table
php artisan migrate
```

## Создание Job для обновления цен

```php
<?php

namespace App\Jobs;

use App\Models\Part;
use App\Services\PriceService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class UpdatePricesJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 300;

    /**
     * Создать новый job
     */
    public function __construct(
        protected ?int $partId = null
    ) {}

    /**
     * Выполнить job
     */
    public function handle(PriceService $priceService): void
    {
        Log::info('Начало обновления цен', ['part_id' => $this->partId]);

        try {
            if ($this->partId) {
                // Обновить цену конкретного товара
                $part = Part::find($this->partId);
                if ($part) {
                    $priceService->recalculatePrice($part);
                    Log::info('Цена обновлена', ['part_id' => $part->id]);
                }
            } else {
                // Обновить все цены
                $updated = $priceService->updateAllPrices();
                Log::info('Цены обновлены', ['updated' => $updated]);
            }
        } catch (\Exception $e) {
            Log::error('Ошибка обновления цен', [
                'part_id' => $this->partId,
                'error' => $e->getMessage(),
            ]);
            throw $e;
        }
    }

    /**
     * Обработка неудачной попытки
     */
    public function failed(\Throwable $exception): void
    {
        Log::error('Job обновления цен завершился с ошибкой', [
            'part_id' => $this->partId,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

## Создание Job для кеширования товаров

```php
<?php

namespace App\Jobs;

use App\Services\PartCacheService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class CachePartJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 60;

    /**
     * Создать новый job
     */
    public function __construct(
        protected int $partId
    ) {}

    /**
     * Выполнить job
     */
    public function handle(PartCacheService $cacheService): void
    {
        Log::info('Начало кеширования товара', ['part_id' => $this->partId]);

        try {
            $part = $cacheService->getPart($this->partId);

            if ($part) {
                $cacheService->getRelatedParts(
                    $part->id,
                    $part->brand,
                    $part->category
                );

                Log::info('Товар закеширован', ['part_id' => $this->partId]);
            } else {
                Log::warning('Товар не найден', ['part_id' => $this->partId]);
            }
        } catch (\Exception $e) {
            Log::error('Ошибка кеширования товара', [
                'part_id' => $this->partId,
                'error' => $e->getMessage(),
            ]);
            throw $e;
        }
    }

    /**
     * Обработка неудачной попытки
     */
    public function failed(\Throwable $exception): void
    {
        Log::error('Job кеширования товара завершился с ошибкой', [
            'part_id' => $this->partId,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

## Создание Job для массового кеширования

```php
<?php

namespace App\Jobs;

use App\Models\Part;
use App\Services\PartCacheService;
use Illuminate\Bus\Batchable;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class BatchCachePartsJob implements ShouldQueue
{
    use Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 300;

    /**
     * Создать новый job
     */
    public function __construct(
        protected array $partIds
    ) {}

    /**
     * Выполнить job
     */
    public function handle(PartCacheService $cacheService): void
    {
        Log::info('Начало массового кеширования', [
            'count' => count($this->partIds),
        ]);

        $cached = 0;
        foreach ($this->partIds as $partId) {
            try {
                $cacheService->getPart($partId);
                $cached++;
            } catch (\Exception $e) {
                Log::warning('Ошибка кеширования товара', [
                    'part_id' => $partId,
                    'error' => $e->getMessage(),
                ]);
            }
        }

        Log::info('Массовое кеширование завершено', [
            'cached' => $cached,
            'total' => count($this->partIds),
        ]);
    }
}
```

## Создание Job для записи логов

```php
<?php

namespace App\Jobs;

use App\Models\SearchLog;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class LogSearchJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 30;

    /**
     * Создать новый job
     */
    public function __construct(
        protected string $query,
        protected ?int $userId = null,
        protected string $ip,
        protected ?string $userAgent = null
    ) {}

    /**
     * Выполнить job
     */
    public function handle(): void
    {
        SearchLog::create([
            'query' => $this->query,
            'user_id' => $this->userId,
            'ip' => $this->ip,
            'user_agent' => $this->userAgent,
            'results_count' => 0, // Будет обновлено позже
            'created_at' => now(),
        ]);
    }
}
```

## Отправка Jobs в очереди

### В контроллере поиска

```php
<?php

namespace App\Http\Controllers;

use App\Jobs\LogSearchJob;
use App\Services\SearchService;
use Illuminate\Http\Request;

class SearchController extends Controller
{
    public function search(Request $request, SearchService $searchService)
    {
        $query = $request->get('q', '');

        if (empty($query)) {
            return redirect()->route('catalog');
        }

        // Логируем поиск через очередь
        LogSearchJob::dispatch(
            $query,
            auth()->id(),
            $request->ip(),
            $request->userAgent()
        );

        // Выполняем поиск
        $results = $searchService->search($query, $request->get('page', 1));

        return view('search.results', compact('query', 'results'));
    }
}
```

### При обновлении цены товара

```php
<?php

namespace App\Observers;

use App\Jobs\CachePartJob;
use App\Jobs\UpdatePricesJob;
use App\Models\Part;

class PartObserver
{
    public function updated(Part $part)
    {
        // Обновить цену в очереди
        UpdatePricesJob::dispatch($part->id);

        // Обновить кеширование
        CachePartJob::dispatch($part->id);
    }
}
```

### Массовое обновление цен

```php
<?php

namespace App\Http\Controllers;

use App\Jobs\UpdatePricesJob;
use Illuminate\Http\Request;

class PriceController extends Controller
{
    public function updateAll()
    {
        // Отправляем job для обновления всех цен
        UpdatePricesJob::dispatch();

        return back()->with('success', 'Задача обновления цен отправлена в очередь');
    }
}
```

### Массовое кеширование товаров

```php
<?php

namespace App\Http\Controllers;

use App\Jobs\BatchCachePartsJob;
use App\Models\Part;
use Illuminate\Http\Request;
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

class CacheController extends Controller
{
    public function warmup()
    {
        // Получаем ID всех товаров
        $partIds = Part::where('is_active', true)
            ->pluck('id')
            ->toArray();

        // Разбиваем на батчи по 100 товаров
        $batches = array_chunk($partIds, 100);

        $jobs = [];
        foreach ($batches as $batch) {
            $jobs[] = new BatchCachePartsJob($batch);
        }

        // Отправляем батчи в очередь
        Bus::batch($jobs)
            ->then(function (Batch $batch) {
                logger('Батчи кеширования завершены', ['batch_id' => $batch->id]);
            })
            ->catch(function (Batch $batch, \Throwable $e) {
                logger('Ошибка батча кеширования', [
                    'batch_id' => $batch->id,
                    'error' => $e->getMessage(),
                ]);
            })
            ->dispatch();

        return back()->with('success', 'Задачи кеширования отправлены в очередь');
    }
}
```

## Supervisor для запуска workers

### Установка Supervisor

```bash
sudo apt update
sudo apt install supervisor
```

### Конфигурация Laravel worker

Создать файл `/etc/supervisor/conf.d/laravel-worker.conf`:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/autoshop/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=3
redirect_stderr=true
stdout_logfile=/var/www/autoshop/storage/logs/worker.log
stopwaitsecs=3600
```

### Управление supervisor

```bash
# Прочитать конфигурацию
sudo supervisorctl reread

# Обновить supervisor
sudo supervisorctl update

# Запустить workers
sudo supervisorctl start laravel-worker:*

# Остановить workers
sudo supervisorctl stop laravel-worker:*

# Перезапустить workers
sudo supervisorctl restart laravel-worker:*

# Проверить статус
sudo supervisorctl status

# Посмотреть логи
sudo tail -f /var/www/autoshop/storage/logs/worker.log
```

## Консольные команды

```bash
# Запустить worker в консоли (для отладки)
php artisan queue:work redis

# Запустить worker с одним job
php artisan queue:work --once

# Запустить worker с ограничением попыток
php artisan queue:work redis --tries=3

# Запустить worker с таймаутом
php artisan queue:work redis --timeout=300

# Просмотреть список jobs в очереди
php artisan queue:monitor redis:default,redis:high,redis:low --max=100

# Очистить все failed jobs
php artisan queue:flush

# Повторить все failed jobs
php artisan queue:retry all

# Повторить конкретный failed job
php artisan queue:retry <id>

# Очистить очереди
php artisan queue:clear redis
```

## Мониторинг очередей

### Dashboard для мониторинга

```bash
# Установить Horizon
composer require laravel/horizon

# Опубликовать конфигурацию
php artisan horizon:install

# Запустить horizon
php artisan horizon
```

### Конфигурация Horizon в .env

```env
HORIZON_ENABLED=true
HORIZON_PREFIX=autoshop_horizon:
```

## Git

```bash
git add .
git commit -m "feat: добавлена система очередей для обновления цен, кеширования и логов"
git push
```

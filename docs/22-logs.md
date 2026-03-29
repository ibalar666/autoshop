# ЭТАП 22. Логи поиска

## Таблица search_logs

```sql
CREATE TABLE search_logs (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    query VARCHAR(255) NOT NULL,
    user_id BIGINT UNSIGNED NULL,
    ip VARCHAR(45) NOT NULL,
    user_agent TEXT NULL,
    results_count INT UNSIGNED DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_query (query),
    INDEX idx_user (user_id),
    INDEX idx_created_at (created_at),
    INDEX idx_popularity (results_count, created_at),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Модель SearchLog

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class SearchLog extends Model
{
    protected $fillable = [
        'query',
        'user_id',
        'ip',
        'user_agent',
        'results_count',
    ];

    protected $casts = [
        'created_at' => 'datetime',
    ];

    /**
     * Связь с пользователем
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Популярные поисковые запросы
     */
    public static function getPopularQueries(int $limit = 10, int $days = 30): \Illuminate\Support\Collection
    {
        return static::select('query', \DB::raw('COUNT(*) as count'))
            ->where('created_at', '>=', now()->subDays($days))
            ->where('results_count', '>', 0)
            ->groupBy('query')
            ->orderBy('count', 'desc')
            ->limit($limit)
            ->get();
    }

    /**
     * Поисковые запросы без результатов
     */
    public static function getNoResultQueries(int $limit = 20, int $days = 7): \Illuminate\Support\Collection
    {
        return static::select('query', \DB::raw('COUNT(*) as count'))
            ->where('created_at', '>=', now()->subDays($days))
            ->where('results_count', '=', 0)
            ->groupBy('query')
            ->orderBy('count', 'desc')
            ->limit($limit)
            ->get();
    }

    /**
     * Статистика поиска
     */
    public static function getSearchStats(int $days = 30): array
    {
        return [
            'total_searches' => static::where('created_at', '>=', now()->subDays($days))->count(),
            'unique_users' => static::where('created_at', '>=', now()->subDays($days))
                ->whereNotNull('user_id')
                ->distinct('user_id')
                ->count(),
            'avg_results' => static::where('created_at', '>=', now()->subDays($days))
                ->avg('results_count'),
            'no_results_count' => static::where('created_at', '>=', now()->subDays($days))
                ->where('results_count', '=', 0)
                ->count(),
        ];
    }
}
```

## SearchLogService

```php
<?php

namespace App\Services;

use App\Models\SearchLog;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class SearchLogService
{
    /**
     * Записать лог поиска
     */
    public function logSearch(string $query, int $resultsCount, ?Request $request = null): void
    {
        try {
            SearchLog::create([
                'query' => $query,
                'user_id' => auth()->id(),
                'ip' => $request ? $request->ip() : request()->ip(),
                'user_agent' => $request ? $request->userAgent() : request()->userAgent(),
                'results_count' => $resultsCount,
            ]);
        } catch (\Exception $e) {
            Log::error('Ошибка записи лога поиска', [
                'query' => $query,
                'error' => $e->getMessage(),
            ]);
        }
    }

    /**
     * Получить популярные запросы
     */
    public function getPopularQueries(int $limit = 10, int $days = 30): \Illuminate\Support\Collection
    {
        return SearchLog::getPopularQueries($limit, $days);
    }

    /**
     * Получить запросы без результатов
     */
    public function getNoResultQueries(int $limit = 20, int $days = 7): \Illuminate\Support\Collection
    {
        return SearchLog::getNoResultQueries($limit, $days);
    }

    /**
     * Получить статистику поиска
     */
    public function getSearchStats(int $days = 30): array
    {
        return SearchLog::getSearchStats($days);
    }

    /**
     * Получить историю поиска пользователя
     */
    public function getUserHistory(int $userId, int $limit = 20): \Illuminate\Support\Collection
    {
        return SearchLog::where('user_id', $userId)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get()
            ->pluck('query')
            ->unique();
    }

    /**
     * Предложения автозаполнения на основе популярности
     */
    public function getSuggestions(string $query, int $limit = 5): array
    {
        return SearchLog::select('query')
            ->where('query', 'like', "{$query}%")
            ->where('results_count', '>', 0)
            ->groupBy('query')
            ->orderByRaw('COUNT(*) DESC')
            ->limit($limit)
            ->pluck('query')
            ->toArray();
    }

    /**
     * Очистить старые логи
     */
    public function clearOldLogs(int $days = 90): int
    {
        return SearchLog::where('created_at', '<', now()->subDays($days))->delete();
    }
}
```

## Обновление SearchController

```php
<?php

namespace App\Http\Controllers;

use App\Services\SearchService;
use App\Services\SearchLogService;
use Illuminate\Http\Request;

class SearchController extends Controller
{
    protected SearchService $searchService;
    protected SearchLogService $logService;

    public function __construct(
        SearchService $searchService,
        SearchLogService $logService
    ) {
        $this->searchService = $searchService;
        $this->logService = $logService;
    }

    public function search(Request $request)
    {
        $query = trim($request->get('q', ''));
        $page = $request->get('page', 1);

        if (empty($query)) {
            return redirect()->route('catalog');
        }

        // Выполняем поиск
        $results = $this->searchService->search($query, $page);

        // Записываем лог поиска
        $this->logService->logSearch($query, $results['total'], $request);

        // Получаем предложения
        $suggestions = $this->logService->getSuggestions($query);

        return view('search.results', compact('query', 'results', 'suggestions'));
    }

    /**
     * API для автозаполнения
     */
    public function autocomplete(Request $request)
    {
        $query = trim($request->get('q', ''));

        if (strlen($query) < 2) {
            return response()->json([]);
        }

        $suggestions = $this->logService->getSuggestions($query, 8);

        return response()->json($suggestions);
    }
}
```

## Маршруты

```php
// routes/web.php
Route::get('/search', [SearchController::class, 'search'])->name('search');
Route::get('/search/autocomplete', [SearchController::class, 'autocomplete'])->name('search.autocomplete');
```

## Маршрут для админки

```php
// routes/web.php
Route::middleware(['auth', 'admin'])->group(function () {
    Route::prefix('admin')->name('admin.')->group(function () {
        Route::get('/search-stats', [AdminSearchController::class, 'stats'])->name('search.stats');
        Route::get('/search-logs', [AdminSearchController::class, 'logs'])->name('search.logs');
    });
});
```

## AdminSearchController

```php
<?php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Services\SearchLogService;
use Illuminate\Http\Request;

class AdminSearchController extends Controller
{
    protected SearchLogService $logService;

    public function __construct(SearchLogService $logService)
    {
        $this->logService = $logService;
    }

    /**
     * Статистика поиска
     */
    public function stats(Request $request)
    {
        $days = $request->get('days', 30);

        $stats = $this->logService->getSearchStats($days);
        $popularQueries = $this->logService->getPopularQueries(20, $days);
        $noResultQueries = $this->logService->getNoResultQueries(30, 7);

        return view('admin.search.stats', compact('stats', 'popularQueries', 'noResultQueries', 'days'));
    }

    /**
     * Логи поиска
     */
    public function logs(Request $request)
    {
        $query = $request->get('query');
        $fromDate = $request->get('from_date');
        $toDate = $request->get('to_date');

        $logs = \App\Models\SearchLog::with('user')
            ->when($query, function ($q) use ($query) {
                $q->where('query', 'like', "%{$query}%");
            })
            ->when($fromDate, function ($q) use ($fromDate) {
                $q->where('created_at', '>=', $fromDate);
            })
            ->when($toDate, function ($q) use ($toDate) {
                $q->where('created_at', '<=', $toDate . ' 23:59:59');
            })
            ->orderBy('created_at', 'desc')
            ->paginate(50);

        return view('admin.search.logs', compact('logs'));
    }
}
```

## Blade шаблоны

### suggestions.blade.php (автозаполнение)

```blade
<div class="search-suggestions" id="search-suggestions" style="display: none;">
    <ul class="suggestions-list">
        {{-- Заполняется через JavaScript --}}
    </ul>
</div>

<script>
const searchInput = document.querySelector('input[name="q"]');
const suggestionsContainer = document.getElementById('search-suggestions');
const suggestionsList = suggestionsContainer.querySelector('.suggestions-list');

let timeout;

searchInput.addEventListener('input', function() {
    const query = this.value.trim();

    clearTimeout(timeout);

    if (query.length < 2) {
        suggestionsContainer.style.display = 'none';
        return;
    }

    timeout = setTimeout(() => {
        fetch(`{{ route('search.autocomplete') }}?q=${encodeURIComponent(query)}`)
            .then(response => response.json())
            .then(data => {
                if (data.length > 0) {
                    suggestionsList.innerHTML = data.map(suggestion =>
                        `<li><a href="{{ route('search') }}?q=${encodeURIComponent(suggestion)}">${suggestion}</a></li>`
                    ).join('');
                    suggestionsContainer.style.display = 'block';
                } else {
                    suggestionsContainer.style.display = 'none';
                }
            });
    }, 300);
});

// Скрывать при клике вне
document.addEventListener('click', function(e) {
    if (!suggestionsContainer.contains(e.target) && e.target !== searchInput) {
        suggestionsContainer.style.display = 'none';
    }
});
</script>
```

### admin/search/stats.blade.php

```blade
@extends('layouts.admin')

@section('title', 'Статистика поиска')

@section('content')
<div class="admin-page">
    <div class="container">
        <h1>Статистика поиска</h1>

        <!-- Фильтр по периоду -->
        <div class="stats-filters">
            <form method="GET" class="inline-form">
                <label for="days">Период:</label>
                <select id="days" name="days">
                    <option value="7" {{ $days == 7 ? 'selected' : '' }}>7 дней</option>
                    <option value="30" {{ $days == 30 ? 'selected' : '' }}>30 дней</option>
                    <option value="90" {{ $days == 90 ? 'selected' : '' }}>90 дней</option>
                </select>
                <button type="submit" class="btn btn-secondary">Обновить</button>
            </form>
        </div>

        <!-- Основная статистика -->
        <div class="stats-cards">
            <div class="stats-card">
                <div class="stats-value">{{ number_format($stats['total_searches'], 0, '', ' ') }}</div>
                <div class="stats-label">Всего поисков</div>
            </div>
            <div class="stats-card">
                <div class="stats-value">{{ number_format($stats['unique_users'], 0, '', ' ') }}</div>
                <div class="stats-label">Уникальных пользователей</div>
            </div>
            <div class="stats-card">
                <div class="stats-value">{{ number_format($stats['avg_results'], 1) }}</div>
                <div class="stats-label">Среднее результатов</div>
            </div>
            <div class="stats-card">
                <div class="stats-value">{{ number_format($stats['no_results_count'], 0, '', ' ') }}</div>
                <div class="stats-label">Без результатов</div>
            </div>
        </div>

        <!-- Популярные запросы -->
        <div class="stats-section">
            <h2>Популярные запросы</h2>
            <table class="data-table">
                <thead>
                    <tr>
                        <th>Запрос</th>
                        <th>Количество</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach($popularQueries as $item)
                    <tr>
                        <td><a href="{{ route('search') }}?q={{ urlencode($item->query) }}">{{ $item->query }}</a></td>
                        <td>{{ number_format($item->count, 0, '', ' ') }}</td>
                    </tr>
                    @endforeach
                </tbody>
            </table>
        </div>

        <!-- Запросы без результатов -->
        <div class="stats-section">
            <h2>Запросы без результатов</h2>
            <table class="data-table">
                <thead>
                    <tr>
                        <th>Запрос</th>
                        <th>Количество</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach($noResultQueries as $item)
                    <tr>
                        <td><a href="{{ route('search') }}?q={{ urlencode($item->query) }}">{{ $item->query }}</a></td>
                        <td>{{ number_format($item->count, 0, '', ' ') }}</td>
                    </tr>
                    @endforeach
                </tbody>
            </table>
        </div>
    </div>
</div>
@endsection
```

## Artisan команда для очистки старых логов

```php
<?php

namespace App\Console\Commands;

use App\Services\SearchLogService;
use Illuminate\Console\Command;

class ClearSearchLogsCommand extends Command
{
    protected $signature = 'search:clear-logs {--days=90 : Удалять логи старее указанного количества дней}';
    protected $description = 'Очистить старые логи поиска';

    public function __construct(
        protected SearchLogService $logService
    ) {
        parent::__construct();
    }

    public function handle()
    {
        $days = (int) $this->option('days');

        $deleted = $this->logService->clearOldLogs($days);

        $this->info("Удалено логов: {$deleted}");

        return 0;
    }
}
```

## Cron задача для автоматической очистки

```bash
# Добавить в crontab
crontab -e

# Команда для очистки логов старше 90 дней каждый день в 3 часа ночи
0 3 * * * cd /var/www/autoshop && php artisan search:clear-logs --days=90 >> /var/log/autoshop/cleanup.log 2>&1
```

## Консольные команды

```bash
# Создать миграцию
php artisan make:migration create_search_logs_table

# Запустить миграцию
php artisan migrate

# Создать команду
php artisan make:command ClearSearchLogsCommand

# Очистить старые логи
php artisan search:clear-logs --days=90

# Проверить логи поиска в базе
mysql -u root -p autoshop
SELECT query, COUNT(*) as count, MAX(created_at) as last_search
FROM search_logs
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY query
ORDER BY count DESC
LIMIT 20;
```

## Git

```bash
git add .
git commit -m "feat: добавлено логирование поисковых запросов с аналитикой"
git push
```

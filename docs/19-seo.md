# ЭТАП 19. SEO

## Таблица seo_pages

```sql
CREATE TABLE seo_pages (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    page_type VARCHAR(50) NOT NULL COMMENT 'Тип страницы: home, catalog, category, part, search',
    page_key VARCHAR(255) NULL COMMENT 'Идентификатор страницы: category slug, part id и т.д.',
    title VARCHAR(255) NOT NULL,
    description TEXT NULL,
    keywords TEXT NULL,
    h1 VARCHAR(255) NULL,
    og_title VARCHAR(255) NULL,
    og_description TEXT NULL,
    og_image VARCHAR(255) NULL,
    canonical VARCHAR(255) NULL,
    robots VARCHAR(50) NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_page_type (page_type),
    INDEX idx_page_key (page_key),
    UNIQUE KEY unique_page (page_type, page_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Модель SeoPage

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class SeoPage extends Model
{
    protected $fillable = [
        'page_type',
        'page_key',
        'title',
        'description',
        'keywords',
        'h1',
        'og_title',
        'og_description',
        'og_image',
        'canonical',
        'robots',
    ];

    protected $casts = [
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
    ];

    /**
     * Получить SEO для страницы
     */
    public static function forPage(string $type, ?string $key = null): ?self
    {
        return static::where('page_type', $type)
            ->where('page_key', $key ?? '')
            ->first();
    }
}
```

## SeoService

```php
<?php

namespace App\Services;

use App\Models\SeoPage;
use Illuminate\Support\Facades\Cache;

class SeoService
{
    /**
     * Получить SEO данные для страницы
     */
    public function getSeoData(string $type, ?string $key = null, array $variables = []): array
    {
        $seo = Cache::remember(
            "seo:{$type}:{$key}",
            86400,
            function () use ($type, $key) {
                return SeoPage::forPage($type, $key);
            }
        );

        return [
            'title' => $this->replaceVariables($seo->title ?? $this->getDefaultTitle($type), $variables),
            'description' => $this->replaceVariables($seo->description ?? $this->getDefaultDescription($type), $variables),
            'keywords' => $seo->keywords ?? null,
            'h1' => $this->replaceVariables($seo->h1 ?? null, $variables),
            'og:title' => $this->replaceVariables($seo->og_title ?? null, $variables),
            'og:description' => $this->replaceVariables($seo->og_description ?? null, $variables),
            'og:image' => $seo->og_image ?? null,
            'canonical' => $seo->canonical ?? null,
            'robots' => $seo->robots ?? 'index, follow',
        ];
    }

    /**
     * Замена переменных в SEO текстах
     */
    protected function replaceVariables(?string $text, array $variables): ?string
    {
        if ($text === null) {
            return null;
        }

        foreach ($variables as $key => $value) {
            $text = str_replace('{' . $key . '}', $value, $text);
        }

        return $text;
    }

    /**
     * Заголовок по умолчанию
     */
    protected function getDefaultTitle(string $type): string
    {
        return match ($type) {
            'home' => 'АвтоШоп - Запчасти для иномарок',
            'catalog' => 'Каталог автозапчастей',
            'category' => 'Автозапчасти {category}',
            'part' => '{name} {brand} - купить по выгодной цене',
            'search' => 'Результаты поиска по запросу {query}',
            'cart' => 'Корзина покупок',
            'checkout' => 'Оформление заказа',
            default => config('app.name', 'АвтоШоп'),
        };
    }

    /**
     * Описание по умолчанию
     */
    protected function getDefaultDescription(string $type): string
    {
        return match ($type) {
            'home' => 'Купить автозапчасти для иномарок по низким ценам. Доставка по всей России. Гарантия качества.',
            'catalog' => 'Большой выбор автозапчастей для всех марок автомобилей. Низкие цены, быстрая доставка.',
            'category' => 'Купить {category} в интернет-магазине АвтоШоп. Огромный выбор, низкие цены, гарантия.',
            'part' => '{name} {brand} с доставкой. Оригинальные запчасти и качественные аналоги.',
            default => 'Интернет-магазин автозапчастей АвтоШоп',
        };
    }

    /**
     * Сохранить или обновить SEO данные
     */
    public function saveSeoData(string $type, ?string $key, array $data): SeoPage
    {
        return SeoPage::updateOrCreate(
            ['page_type' => $type, 'page_key' => $key ?? ''],
            $data
        );
    }

    /**
     * Очистить кеширование SEO
     */
    public function clearCache(string $type, ?string $key = null): void
    {
        if ($key) {
            Cache::forget("seo:{$type}:{$key}");
        } else {
            $redis = Cache::getStore();
            $keys = $redis->connection('cache')->keys("seo:{$type}:*");
            foreach ($keys as $cacheKey) {
                Cache::forget($cacheKey);
            }
        }
    }
}
```

## Middleware для SEO

```php
<?php

namespace App\Http\Middleware;

use App\Services\SeoService;
use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\View;

class SeoMiddleware
{
    protected SeoService $seoService;

    public function __construct(SeoService $seoService)
    {
        $this->seoService = $seoService;
    }

    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        $route = $request->route();

        if ($route) {
            $seoData = $this->getSeoDataForRoute($route, $request);
            View::share('seo', $seoData);
        }

        return $response;
    }

    /**
     * Получить SEO данные для маршрута
     */
    protected function getSeoDataForRoute($route, Request $request): array
    {
        $routeName = $route->getName();

        return match ($routeName) {
            'home' => $this->seoService->getSeoData('home'),
            'catalog' => $this->seoService->getSeoData('catalog'),
            'catalog.category' => $this->seoService->getSeoData(
                'category',
                $request->route('category'),
                ['category' => $request->route('category')]
            ),
            'parts.show' => $this->seoService->getSeoData(
                'part',
                $request->route('part')->id,
                [
                    'name' => $request->route('part')->name,
                    'brand' => $request->route('part')->brand,
                ]
            ),
            'search' => $this->seoService->getSeoData(
                'search',
                $request->get('q'),
                ['query' => $request->get('q', '')]
            ),
            default => [],
        };
    }
}
```

## Blade шаблон для SEO мета-тегов

### Файл resources/views/layouts/seo.blade.php

```blade
{{-- SEO мета-теги --}}
@if(isset($seo))
    <title>{{ $seo['title'] }}</title>
    <meta name="description" content="{{ $seo['description'] }}">
    @if($seo['keywords'])
        <meta name="keywords" content="{{ $seo['keywords'] }}">
    @endif
    <meta name="robots" content="{{ $seo['robots'] }}">

    {{-- Open Graph --}}
    <meta property="og:title" content="{{ $seo['og:title'] ?? $seo['title'] }}">
    <meta property="og:description" content="{{ $seo['og:description'] ?? $seo['description'] }}">
    <meta property="og:type" content="website">
    <meta property="og:url" content="{{ request()->fullUrl() }}">
    @if($seo['og:image'])
        <meta property="og:image" content="{{ $seo['og:image'] }}">
    @elseif(isset($part) && $part->main_image)
        <meta property="og:image" content="{{ asset('storage/' . $part->main_image) }}">
    @else
        <meta property="og:image" content="{{ asset('images/og-default.jpg') }}">
    @endif

    {{-- Twitter Card --}}
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:title" content="{{ $seo['og:title'] ?? $seo['title'] }}">
    <meta name="twitter:description" content="{{ $seo['og:description'] ?? $seo['description'] }}">
    @if($seo['og:image'])
        <meta name="twitter:image" content="{{ $seo['og:image'] }}">
    @endif

    {{-- Canonical --}}
    @if($seo['canonical'])
        <link rel="canonical" href="{{ $seo['canonical'] }}">
    @else
        <link rel="canonical" href="{{ request()->fullUrl() }}">
    @endif
@endif
```

## Примеры SEO данных

```sql
INSERT INTO seo_pages (page_type, page_key, title, description, h1, keywords) VALUES
('home', NULL,
 'АвтоШоп - Запчасти для иномарок в Москве',
 'Купить автозапчасти для иномарок по низким ценам в Москве. Доставка по всей России. Гарантия качества.',
 'Автозапчасти для иномарок',
 'автозапчасти, иномарки, запчасти москва, автозапчасти онлайн'),

('catalog', NULL,
 'Каталог автозапчастей - АвтоШоп',
 'Большой выбор автозапчастей для всех марок автомобилей. Низкие цены, быстрая доставка по России.',
 'Каталог автозапчастей',
 'каталог запчастей, автозапчасти, автозапчасти цена'),

('category', 'filters',
 'Фильтры для автомобилей - купить в АвтоШоп',
 'Купить масляные, воздушные, топливные и салонные фильтры для автомобилей. Низкие цены, гарантия качества.',
 'Фильтры для автомобилей',
 'фильтры масляные, фильтры воздушные, фильтры топливные, фильтры салонные'),

('category', 'brakes',
 'Тормозная система - купить тормозные колодки и диски',
 'Тормозные колодки, диски, барабаны и другие детали тормозной системы. Доставка по всей России.',
 'Тормозная система',
 'тормозные колодки, тормозные диски, тормозная система'),

('search', NULL,
 'Результаты поиска по запросу {query} - АвтоШоп',
 'Найдены автозапчасти по запросу {query}. Большой выбор, низкие цены.',
 'Результаты поиска: {query}', NULL);
```

## SeoController

```php
<?php

namespace App\Http\Controllers;

use App\Services\SeoService;
use Illuminate\Http\Request;

class SeoController extends Controller
{
    protected SeoService $seoService;

    public function __construct(SeoService $seoService)
    {
        $this->seoService = $seoService;
    }

    /**
     * Редактирование SEO для страницы
     */
    public function edit(string $type, ?string $key = null)
    {
        $seo = \App\Models\SeoPage::forPage($type, $key);

        return view('seo.edit', compact('type', 'key', 'seo'));
    }

    /**
     * Сохранение SEO для страницы
     */
    public function update(Request $request, string $type, ?string $key = null)
    {
        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'description' => 'nullable|string|max:500',
            'keywords' => 'nullable|string|max:255',
            'h1' => 'nullable|string|max:255',
            'og_title' => 'nullable|string|max:255',
            'og_description' => 'nullable|string|max:500',
            'og_image' => 'nullable|string|max:255',
            'canonical' => 'nullable|string|max:255',
            'robots' => 'nullable|string|max:50',
        ]);

        $this->seoService->saveSeoData($type, $key, $validated);
        $this->seoService->clearCache($type, $key);

        return back()->with('success', 'SEO данные сохранены');
    }
}
```

## Генерация Sitemap

```php
<?php

namespace App\Http\Controllers;

use App\Models\Category;
use App\Models\Part;
use Illuminate\Support\Facades\Response;
use Spatie\Sitemap\Sitemap;
use Spatie\Sitemap\Tags\Url;

class SitemapController extends Controller
{
    public function index()
    {
        $sitemap = Sitemap::create()
            ->add(Url::create('/')
                ->setChangeFrequency(Url::CHANGE_FREQUENCY_DAILY)
                ->setPriority(1.0))
            ->add(Url::create('/catalog')
                ->setChangeFrequency(Url::CHANGE_FREQUENCY_DAILY)
                ->setPriority(0.9));

        // Категории
        Category::all()->each(function ($category) use ($sitemap) {
            $sitemap->add(Url::create("/catalog/{$category->slug}")
                ->setChangeFrequency(Url::CHANGE_FREQUENCY_WEEKLY)
                ->setPriority(0.8));
        });

        // Товары
        Part::where('is_active', true)
            ->where('price', '>', 0)
            ->each(function ($part) use ($sitemap) {
                $sitemap->add(Url::create("/parts/{$part->id}")
                    ->setChangeFrequency(Url::CHANGE_FREQUENCY_MONTHLY)
                    ->setPriority(0.7));
            });

        return $sitemap->toResponse(request());
    }
}
```

### Маршрут для sitemap

```php
// routes/web.php
Route::get('/sitemap.xml', [SitemapController::class, 'index'])
    ->name('sitemap');
```

### Установка Spatie Sitemap

```bash
composer require spatie/laravel-sitemap
```

## Robots.txt

### Файл public/robots.txt

```txt
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /cart/
Disallow: /checkout/
Disallow: /api/
Disallow: /storage/

Sitemap: https://autoshop.ru/sitemap.xml
```

## Структурированные данные (Schema.org)

### Для товара

```blade
<script type="application/ld+json">
{
  "@context": "https://schema.org/",
  "@type": "Product",
  "name": "{{ $part->name }}",
  "brand": {
    "@type": "Brand",
    "name": "{{ $part->brand }}"
  },
  "sku": "{{ $part->sku }}",
  "image": [
    @foreach($part->images as $image)
    "{{ asset('storage/' . $image->path) }}"@if(!$loop->last),@endif
    @endforeach
  ],
  "description": "{{ $part->description ?? '' }}",
  "offers": {
    "@type": "Offer",
    "url": "{{ route('parts.show', $part->id) }}",
    "priceCurrency": "RUB",
    "price": "{{ $part->price }}",
    "availability": "https://schema.org/InStock",
    "seller": {
      "@type": "Organization",
      "name": "АвтоШоп"
    }
  }
}
</script>
```

### Для организации

```blade
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "АвтоШоп",
  "url": "https://autoshop.ru",
  "logo": "https://autoshop.ru/images/logo.png",
  "contactPoint": {
    "@type": "ContactPoint",
    "telephone": "+7-800-123-45-67",
    "contactType": "customer service"
  },
  "sameAs": [
    "https://vk.com/autoshop",
    "https://t.me/autoshop"
  ]
}
</script>
```

## Консольные команды

```bash
# Создать миграцию для seo_pages
php artisan make:migration create_seo_pages_table

# Запустить миграцию
php artisan migrate

# Создать контроллер SEO
php artisan make:controller SeoController

# Создать middleware SEO
php artisan make:middleware SeoMiddleware

# Сгенерировать sitemap
php artisan sitemap:generate
```

## Git

```bash
git add .
git commit -m "feat: добавлена SEO система с мета-тегами и sitemap"
git push
```

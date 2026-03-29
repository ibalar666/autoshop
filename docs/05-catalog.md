# Категории каталога

## ЭТАП 5. Категории каталога

Создаём структуру каталога автозапчастей.

### Пример структуры

```
Фильтры
   ├ Масляные
   ├ Воздушные
   └ Топливные

Тормозная система
   ├ Колодки
   ├ Диски
   └ Барабаны

Двигатель
   ├ Поршни
   ├ Кольца
   └ Вкладыши
```

### URL структура

- `/catalog` — главная каталога
- `/catalog/filters` — категория
- `/catalog/filters/oil` — подкатегория

### Сервис категорий

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Models\Category;

class CatalogService
{
    /**
     * Получить дерево категорий
     */
    public function getTree(): array
    {
        return Category::whereNull('parent_id')
            ->with('children')
            ->where('is_active', true)
            ->orderBy('sort_order')
            ->orderBy('name')
            ->get()
            ->toArray();
    }

    /**
     * Получить категорию по slug
     */
    public function getBySlug(string $slug): ?Category
    {
        return Category::where('slug', $slug)
            ->where('is_active', true)
            ->first();
    }

    /**
     * Получить breadcrumb для категории
     */
    public function getBreadcrumb(Category $category): array
    {
        $breadcrumb = [];
        $current = $category;

        while ($current) {
            $breadcrumb[] = $current;
            $current = $current->parent;
        }

        return array_reverse($breadcrumb);
    }

    /**
     * Получить всех потомков категории
     */
    public function getAllDescendants(Category $category): array
    {
        $descendants = [];

        foreach ($category->children as $child) {
            $descendants[] = $child;
            $descendants = array_merge($descendants, $this->getAllDescendants($child));
        }

        return $descendants;
    }
}
```

### Контроллер каталога

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Models\Category;
use App\Services\CatalogService;
use Illuminate\Http\Response;
use Illuminate\View\View;

class CatalogController extends Controller
{
    public function __construct(
        private readonly CatalogService $catalogService
    ) {
    }

    /**
     * Главная страница каталога
     */
    public function index(): View
    {
        $categories = $this->catalogService->getTree();

        return view('catalog.index', compact('categories'));
    }

    /**
     * Страница категории
     */
    public function category(string $slug): View|Response
    {
        $category = $this->catalogService->getBySlug($slug);

        if (! $category) {
            abort(404);
        }

        $breadcrumb = $this->catalogService->getBreadcrumb($category);

        return view('catalog.category', compact('category', 'breadcrumb'));
    }

    /**
     * Страница подкатегории
     */
    public function subcategory(string $parentSlug, string $slug): View|Response
    {
        $parent = $this->catalogService->getBySlug($parentSlug);

        if (! $parent) {
            abort(404);
        }

        $category = $parent->children()
            ->where('slug', $slug)
            ->where('is_active', true)
            ->first();

        if (! $category) {
            abort(404);
        }

        $breadcrumb = $this->catalogService->getBreadcrumb($category);

        return view('catalog.category', compact('category', 'breadcrumb'));
    }
}
```

### Маршруты

```php
<?php

use App\Http\Controllers\CatalogController;
use Illuminate\Support\Facades\Route;

Route::prefix('catalog')->name('catalog.')->group(function () {
    Route::get('/', [CatalogController::class, 'index'])
        ->name('index');

    Route::get('/{slug}', [CatalogController::class, 'category'])
        ->name('category');

    Route::get('/{parentSlug}/{slug}', [CatalogController::class, 'subcategory'])
        ->name('subcategory');
});
```

### Blade шаблон: дерево категорий

```blade
{{-- resources/views/catalog/index.blade.php --}}
@extends('layouts.app')

@section('content')
    <div class="container py-8">
        <h1 class="text-3xl font-bold mb-8">Каталог автозапчастей</h1>

        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
            @foreach($categories as $category)
                <div class="bg-white rounded-lg shadow p-6">
                    @if($category['image'])
                        <img src="{{ asset('storage/' . $category['image']) }}"
                             alt="{{ $category['name'] }}"
                             class="w-full h-32 object-cover rounded mb-4">
                    @endif

                    <h2 class="text-xl font-semibold mb-2">
                        <a href="{{ route('catalog.category', $category['slug']) }}"
                           class="text-blue-600 hover:text-blue-800">
                            {{ $category['name'] }}
                        </a>
                    </h2>

                    @if($category['description'])
                        <p class="text-gray-600 mb-4">{{ $category['description'] }}</p>
                    @endif

                    @if(!empty($category['children']))
                        <ul class="space-y-1">
                            @foreach($category['children'] as $child)
                                <li>
                                    <a href="{{ route('catalog.subcategory', [$category['slug'], $child['slug']]) }}"
                                       class="text-sm text-gray-700 hover:text-blue-600">
                                        {{ $child['name'] }}
                                    </a>
                                </li>
                            @endforeach
                        </ul>
                    @endif
                </div>
            @endforeach
        </div>
    </div>
@endsection
```

### Компонент breadcrumb

```blade
{{-- resources/views/components/breadcrumb.blade.php --}}
<nav class="text-sm text-gray-600 mb-6">
    <ol class="flex flex-wrap items-center space-x-2">
        <li>
            <a href="{{ route('home') }}" class="hover:text-blue-600">Главная</a>
        </li>

        <li>/</li>

        <li>
            <a href="{{ route('catalog.index') }}" class="hover:text-blue-600">Каталог</a>
        </li>

        @foreach($breadcrumb as $item)
            <li>/</li>

            <li>
                @if($loop->last)
                    <span class="text-gray-900 font-medium">{{ $item->name }}</span>
                @else
                    <a href="{{ $item->parent_id
                        ? route('catalog.subcategory', [$item->parent->slug, $item->slug])
                        : route('catalog.category', $item->slug) }}"
                       class="hover:text-blue-600">
                        {{ $item->name }}
                    </a>
                @endif
            </li>
        @endforeach
    </ol>
</nav>
```

### Seeder категорий

```php
<?php

declare(strict_types=1);

namespace Database\Seeders;

use App\Models\Category;
use Illuminate\Database\Seeder;

class CategorySeeder extends Seeder
{
    public function run(): void
    {
        $categories = [
            [
                'name' => 'Фильтры',
                'slug' => 'filters',
                'description' => 'Масляные, воздушные, топливные и салонные фильтры',
                'children' => [
                    ['name' => 'Масляные', 'slug' => 'oil-filters'],
                    ['name' => 'Воздушные', 'slug' => 'air-filters'],
                    ['name' => 'Топливные', 'slug' => 'fuel-filters'],
                    ['name' => 'Салонные', 'slug' => 'cabin-filters'],
                ],
            ],
            [
                'name' => 'Тормозная система',
                'slug' => 'brakes',
                'description' => 'Тормозные колодки, диски и барабаны',
                'children' => [
                    ['name' => 'Колодки', 'slug' => 'brake-pads'],
                    ['name' => 'Диски', 'slug' => 'brake-discs'],
                    ['name' => 'Барабаны', 'slug' => 'brake-drums'],
                ],
            ],
            [
                'name' => 'Двигатель',
                'slug' => 'engine',
                'description' => 'Запчасти для двигателя',
                'children' => [
                    ['name' => 'Поршни', 'slug' => 'pistons'],
                    ['name' => 'Кольца', 'slug' => 'rings'],
                    ['name' => 'Вкладыши', 'slug' => 'bearings'],
                ],
            ],
        ];

        foreach ($categories as $data) {
            $children = $data['children'] ?? [];
            unset($data['children']);

            $category = Category::create($data);

            foreach ($children as $child) {
                $child['parent_id'] = $category->id;
                Category::create($child);
            }
        }
    }
}
```

### Рекомендации

1. **Иерархия**: Используйте self-referencing отношения для бесконечной вложенности
2. **Индексы**: Добавьте индексы на `slug`, `parent_id`, `is_active`
3. **Кэширование**: Кэшируйте дерево категорий в Redis
4. **Eager Loading**: Всегда используйте `with('children')` для избежания N+1
5. **SEO**: Используйте уникальные slug для всех уровней вложенности

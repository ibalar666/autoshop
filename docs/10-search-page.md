## ЭТАП 10. Страница поиска

URL: `/search?q=OC90`

### SearchController

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Http\Requests\SearchRequest;
use App\Services\PartSearchService;
use Illuminate\View\View;

class SearchController extends Controller
{
    public function __construct(
        private readonly PartSearchService $searchService,
    ) {}

    public function index(SearchRequest $request): View
    {
        $query = $request->get('q');
        $results = $this->searchService->search($query);
        
        return view('search.results', compact('results', 'query'));
    }
}
```

### SearchRequest (Form Request)

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class SearchRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'q' => ['required', 'string', 'min:2', 'max:50'],
        ];
    }

    public function messages(): array
    {
        return [
            'q.required' => 'Введите артикул или название запчасти',
            'q.min' => 'Запрос должен содержать минимум 2 символа',
            'q.max' => 'Запрос слишком длинный',
        ];
    }
}
```

### Blade шаблон

resources/views/search/results.blade.php:

```blade
@extends('layouts.app')

@section('title', 'Результаты поиска: ' . $query)

@section('content')
<div class="container py-5">
    <h1 class="mb-4">Результаты поиска: "{{ $query }}"</h1>
    
    @if(empty($results))
        <div class="alert alert-info">
            По запросу "{{ $query }}" ничего не найдено.
        </div>
    @else
        <div class="row">
            @foreach($results as $product)
                <div class="col-md-6 mb-4">
                    <div class="card h-100">
                        <div class="card-header d-flex justify-content-between align-items-center">
                            <h5 class="mb-0">{{ $product->name }}</h5>
                            <span class="badge bg-primary">{{ $product->brand }}</span>
                        </div>
                        <div class="card-body">
                            <p class="text-muted">Артикул: {{ $product->article }}</p>
                            
                            @if(!empty($product->offers))
                                <h6 class="mt-3">Предложения:</h6>
                                <ul class="list-group list-group-flush">
                                    @foreach($product->offers as $offer)
                                        <li class="list-group-item d-flex justify-content-between align-items-center">
                                            <div>
                                                <span class="fw-bold">{{ number_format($offer->price, 2) }} ₽</span>
                                                <small class="text-muted d-block">
                                                    {{ $offer->supplier }}
                                                    @if($offer->warehouse)
                                                        ({{ $offer->warehouse }})
                                                    @endif
                                                </small>
                                            </div>
                                            <div class="text-end">
                                                <span class="badge bg-{{ $offer->quantity > 0 ? 'success' : 'danger' }}">
                                                    {{ $offer->quantity }} шт.
                                                </span>
                                                <small class="text-muted d-block">{{ $offer->deliveryDays }} дн.</small>
                                            </div>
                                        </li>
                                    @endforeach
                                </ul>
                            @endif
                            
                            @if(!empty($product->analogs))
                                <h6 class="mt-3">Аналоги:</h6>
                                <div class="d-flex flex-wrap gap-2">
                                    @foreach($product->analogs as $analog)
                                        <a href="{{ route('search', ['q' => $analog->article]) }}" 
                                           class="badge bg-secondary text-decoration-none">
                                            {{ $analog->brand }} {{ $analog->article }}
                                        </a>
                                    @endforeach
                                </div>
                            @endif
                        </div>
                        <div class="card-footer">
                            <a href="{{ route('product.show', ['article' => $product->article, 'brand' => $product->brand]) }}" 
                               class="btn btn-primary w-100">
                                Подробнее
                            </a>
                        </div>
                    </div>
                </div>
            @endforeach
        </div>
    @endif
</div>
@endsection
```

### Роут

```php
<?php

use App\Http\Controllers\SearchController;
use Illuminate\Support\Facades\Route;

Route::get('/search', [SearchController::class, 'index'])->name('search');
```

### Структура результата поиска

```php
[
    [
        'article' => 'OC90',
        'brand' => 'MAHLE',
        'name' => 'Фильтр масляный',
        'offers' => [
            [
                'supplier' => 'Autoopt',
                'price' => 450.00,
                'quantity' => 15,
                'deliveryDays' => 2,
                'warehouse' => 'Москва',
            ],
            // ...
        ],
        'analogs' => [
            [
                'article' => 'W712/80',
                'brand' => 'MANN',
                'name' => 'Фильтр масляный',
            ],
            // ...
        ],
    ],
    // ...
]
```

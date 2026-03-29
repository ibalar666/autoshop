## ЭТАП 11. Карточка товара

URL: `/product/{brand}-{article}`

Пример: `/product/mahle-oc90`

### ProductController

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Services\PartSearchService;
use App\Services\ProductService;
use Illuminate\View\View;
use Illuminate\Http\RedirectResponse;

class ProductController extends Controller
{
    public function __construct(
        private readonly PartSearchService $searchService,
        private readonly ProductService $productService,
    ) {}

    public function show(string $brand, string $article): View|RedirectResponse
    {
        $product = $this->productService->findByBrandAndArticle($brand, $article);
        
        if (!$product) {
            return redirect()->route('search', ['q' => $article])
                ->with('error', 'Товар не найден');
        }
        
        $analogs = $this->searchService->findAnalogs($article, $brand);
        $crosses = $this->productService->getCrossReferences($article, $brand);
        
        return view('product.show', compact('product', 'analogs', 'crosses'));
    }
}
```

### ProductService

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\DTO\ProductDTO;

class ProductService
{
    public function __construct(
        private readonly PartSearchService $searchService,
        private readonly CacheService $cache,
    ) {}

    public function findByBrandAndArticle(string $brand, string $article): ?ProductDTO
    {
        $cacheKey = "product:{$brand}:{$article}";
        
        return $this->cache->remember($cacheKey, 600, function () use ($brand, $article) {
            return $this->searchService->findProduct($brand, $article);
        });
    }

    public function getCrossReferences(string $article, string $brand): array
    {
        return $this->searchService->getCrosses($article, $brand);
    }
}
```

### Blade шаблон

resources/views/product/show.blade.php:

```blade
@extends('layouts.app')

@section('title', $product->name . ' ' . $product->brand . ' ' . $product->article)

@section('content')
<div class="container py-5">
    <nav aria-label="breadcrumb">
        <ol class="breadcrumb">
            <li class="breadcrumb-item"><a href="{{ route('home') }}">Главная</a></li>
            <li class="breadcrumb-item"><a href="{{ route('search', ['q' => $product->article]) }}">Поиск</a></li>
            <li class="breadcrumb-item active">{{ $product->name }}</li>
        </ol>
    </nav>

    <div class="row">
        <div class="col-lg-8">
            <div class="card mb-4">
                <div class="card-body">
                    <div class="d-flex justify-content-between align-items-start mb-3">
                        <div>
                            <h1 class="h3 mb-2">{{ $product->name }}</h1>
                            <p class="text-muted mb-0">
                                Бренд: <span class="fw-bold">{{ $product->brand }}</span> | 
                                Артикул: <span class="fw-bold">{{ $product->article }}</span>
                            </p>
                        </div>
                        <span class="badge bg-primary fs-6">{{ $product->brand }}</span>
                    </div>
                    
                    <hr>
                    
                    @if(!empty($product->description))
                        <div class="mb-4">
                            <h5>Описание</h5>
                            <p>{{ $product->description }}</p>
                        </div>
                    @endif
                    
                    @if(!empty($product->attributes))
                        <div class="mb-4">
                            <h5>Характеристики</h5>
                            <table class="table table-sm">
                                <tbody>
                                    @foreach($product->attributes as $key => $value)
                                        <tr>
                                            <td class="text-muted">{{ $key }}</td>
                                            <td class="fw-bold">{{ $value }}</td>
                                        </tr>
                                    @endforeach
                                </tbody>
                            </table>
                        </div>
                    @endif
                </div>
            </div>
            
            @if(!empty($analogs))
                <div class="card mb-4">
                    <div class="card-header">
                        <h5 class="mb-0">Аналоги</h5>
                    </div>
                    <div class="card-body">
                        <div class="table-responsive">
                            <table class="table table-hover">
                                <thead>
                                    <tr>
                                        <th>Бренд</th>
                                        <th>Артикул</th>
                                        <th>Наименование</th>
                                        <th></th>
                                    </tr>
                                </thead>
                                <tbody>
                                    @foreach($analogs as $analog)
                                        <tr>
                                            <td><span class="badge bg-secondary">{{ $analog->brand }}</span></td>
                                            <td>{{ $analog->article }}</td>
                                            <td>{{ $analog->name }}</td>
                                            <td>
                                                <a href="{{ route('product.show', ['brand' => $analog->brand, 'article' => $analog->article]) }}" 
                                                   class="btn btn-sm btn-outline-primary">
                                                    Подробнее
                                                </a>
                                            </td>
                                        </tr>
                                    @endforeach
                                </tbody>
                            </table>
                        </div>
                    </div>
                </div>
            @endif
        </div>
        
        <div class="col-lg-4">
            <div class="card sticky-top" style="top: 20px;">
                <div class="card-header bg-primary text-white">
                    <h5 class="mb-0">Предложения поставщиков</h5>
                </div>
                <div class="card-body p-0">
                    @if(!empty($product->offers))
                        <div class="list-group list-group-flush">
                            @foreach($product->offers as $offer)
                                <div class="list-group-item">
                                    <div class="d-flex justify-content-between align-items-center mb-2">
                                        <span class="fw-bold fs-5 text-primary">
                                            {{ number_format($offer->price, 2) }} ₽
                                        </span>
                                        <span class="badge bg-{{ $offer->quantity > 5 ? 'success' : ($offer->quantity > 0 ? 'warning' : 'danger') }}">
                                            {{ $offer->quantity }} шт.
                                        </span>
                                    </div>
                                    <div class="d-flex justify-content-between align-items-center text-muted small">
                                        <span>{{ $offer->supplier }}</span>
                                        <span>⏱ {{ $offer->deliveryDays }} дн.</span>
                                    </div>
                                    @if($offer->warehouse)
                                        <small class="text-muted">Склад: {{ $offer->warehouse }}</small>
                                    @endif
                                    <button class="btn btn-success btn-sm w-100 mt-2 add-to-cart" 
                                            data-article="{{ $product->article }}"
                                            data-brand="{{ $product->brand }}"
                                            data-name="{{ $product->name }}"
                                            data-price="{{ $offer->price }}"
                                            data-supplier="{{ $offer->supplier }}">
                                        В корзину
                                    </button>
                                </div>
                            @endforeach
                        </div>
                    @else
                        <div class="p-3 text-center text-muted">
                            Нет доступных предложений
                        </div>
                    @endif
                </div>
            </div>
        </div>
    </div>
</div>
@endsection

@push('scripts')
<script>
document.querySelectorAll('.add-to-cart').forEach(button => {
    button.addEventListener('click', function() {
        const data = {
            article: this.dataset.article,
            brand: this.dataset.brand,
            name: this.dataset.name,
            price: this.dataset.price,
            supplier: this.dataset.supplier,
            quantity: 1,
            _token: '{{ csrf_token() }}'
        };
        
        fetch('{{ route('cart.add') }}', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-Requested-With': 'XMLHttpRequest'
            },
            body: JSON.stringify(data)
        })
        .then(response => response.json())
        .then(result => {
            if (result.success) {
                alert('Товар добавлен в корзину');
                updateCartCounter(result.cartCount);
            } else {
                alert(result.message || 'Ошибка добавления в корзину');
            }
        })
        .catch(error => {
            console.error('Error:', error);
            alert('Произошла ошибка');
        });
    });
});

function updateCartCounter(count) {
    const counter = document.querySelector('.cart-counter');
    if (counter) {
        counter.textContent = count;
        counter.classList.add('animate-pulse');
        setTimeout(() => counter.classList.remove('animate-pulse'), 500);
    }
}
</script>
@endpush
```

### Роут

```php
<?php

use App\Http\Controllers\ProductController;
use Illuminate\Support\Facades\Route;

Route::get('/product/{brand}-{article}', [ProductController::class, 'show'])
    ->name('product.show')
    ->where(['brand' => '[a-zA-Z0-9_-]+', 'article' => '[a-zA-Z0-9_-]+']);
```

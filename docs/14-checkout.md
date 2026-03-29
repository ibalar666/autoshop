## ЭТАП 14. Оформление заказа

URL: `/checkout`

### CheckoutRequest (Form Request)

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class CheckoutRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:100'],
            'phone' => ['required', 'string', 'regex:/^[\+\d\s\-\(\)]{10,20}$/'],
            'email' => ['nullable', 'email', 'max:255'],
            'comment' => ['nullable', 'string', 'max:1000'],
        ];
    }

    public function messages(): array
    {
        return [
            'name.required' => 'Введите ваше имя',
            'name.max' => 'Имя слишком длинное',
            'phone.required' => 'Введите номер телефона',
            'phone.regex' => 'Некорректный формат телефона',
            'email.email' => 'Некорректный email адрес',
            'comment.max' => 'Комментарий слишком длинный (макс. 1000 символов)',
        ];
    }

    public function prepareForValidation(): void
    {
        if ($this->has('phone')) {
            $this->merge([
                'phone' => preg_replace('/[^\d+]/', '', $this->phone),
            ]);
        }
    }
}
```

### CheckoutController

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Http\Requests\CheckoutRequest;
use App\Services\CartService;
use App\Services\OrderService;
use Illuminate\View\View;
use Illuminate\Http\RedirectResponse;

class CheckoutController extends Controller
{
    public function __construct(
        private readonly CartService $cartService,
        private readonly OrderService $orderService,
    ) {}

    public function index(): View|RedirectResponse
    {
        $cart = $this->cartService->getCurrentCart();
        
        if ($cart->items->isEmpty()) {
            return redirect()->route('cart.index')
                ->with('error', 'Ваша корзина пуста');
        }

        return view('checkout.index', compact('cart'));
    }

    public function store(CheckoutRequest $request): RedirectResponse
    {
        $cart = $this->cartService->getCurrentCart();
        
        if ($cart->items->isEmpty()) {
            return redirect()->route('cart.index')
                ->with('error', 'Ваша корзина пуста');
        }

        $validated = $request->validated();
        
        try {
            $order = $this->orderService->createFromCart($cart, $validated);
            
            // Clear cart after successful order
            $this->cartService->clearCart($cart);
            
            return redirect()->route('checkout.success', ['order' => $order->id])
                ->with('success', 'Заказ успешно оформлен');
                
        } catch (\Exception $e) {
            return redirect()->back()
                ->with('error', 'Ошибка оформления заказа: ' . $e->getMessage())
                ->withInput();
        }
    }

    public function success(int $orderId): View
    {
        $order = $this->orderService->findOrder($orderId);
        
        return view('checkout.success', compact('order'));
    }
}
```

### Blade шаблон

resources/views/checkout/index.blade.php:

```blade
@extends('layouts.app')

@section('title', 'Оформление заказа')

@section('content')
<div class="container py-5">
    <h1 class="mb-4">Оформление заказа</h1>
    
    <div class="row">
        <div class="col-lg-8">
            <div class="card mb-4">
                <div class="card-header">
                    <h5 class="mb-0">Контактная информация</h5>
                </div>
                <div class="card-body">
                    <form action="{{ route('checkout.store') }}" method="POST" id="checkout-form">
                        @csrf
                        
                        <div class="mb-3">
                            <label for="name" class="form-label">Имя *</label>
                            <input type="text" 
                                   class="form-control @error('name') is-invalid @enderror" 
                                   id="name" 
                                   name="name" 
                                   value="{{ old('name') }}"
                                   placeholder="Иван Иванов"
                                   required>
                            @error('name')
                                <div class="invalid-feedback">{{ $message }}</div>
                            @enderror
                        </div>
                        
                        <div class="mb-3">
                            <label for="phone" class="form-label">Телефон *</label>
                            <input type="tel" 
                                   class="form-control @error('phone') is-invalid @enderror" 
                                   id="phone" 
                                   name="phone" 
                                   value="{{ old('phone') }}"
                                   placeholder="+7 (999) 123-45-67"
                                   required>
                            @error('phone')
                                <div class="invalid-feedback">{{ $message }}</div>
                            @enderror
                        </div>
                        
                        <div class="mb-3">
                            <label for="email" class="form-label">Email</label>
                            <input type="email" 
                                   class="form-control @error('email') is-invalid @enderror" 
                                   id="email" 
                                   name="email" 
                                   value="{{ old('email') }}"
                                   placeholder="email@example.com">
                            <div class="form-text">Для отправки копии заказа</div>
                            @error('email')
                                <div class="invalid-feedback">{{ $message }}</div>
                            @enderror
                        </div>
                        
                        <div class="mb-3">
                            <label for="comment" class="form-label">Комментарий</label>
                            <textarea class="form-control @error('comment') is-invalid @enderror" 
                                      id="comment" 
                                      name="comment" 
                                      rows="4"
                                      placeholder="Дополнительная информация по заказу">{{ old('comment') }}</textarea>
                            @error('comment')
                                <div class="invalid-feedback">{{ $message }}</div>
                            @enderror
                        </div>
                        
                        <div class="d-grid">
                            <button type="submit" class="btn btn-success btn-lg">
                                Подтвердить заказ
                            </button>
                        </div>
                    </form>
                </div>
            </div>
        </div>
        
        <div class="col-lg-4">
            <div class="card sticky-top" style="top: 20px;">
                <div class="card-header">
                    <h5 class="mb-0">Ваш заказ</h5>
                </div>
                <div class="card-body p-0">
                    <ul class="list-group list-group-flush">
                        @foreach($cart->items as $item)
                            <li class="list-group-item">
                                <div class="d-flex justify-content-between">
                                    <div>
                                        <div class="fw-bold">{{ $item->name }}</div>
                                        <small class="text-muted">
                                            {{ $item->brand }} {{ $item->article }} × {{ $item->quantity }}
                                        </small>
                                    </div>
                                    <div class="text-end">
                                        {{ number_format($item->subtotal, 2) }} ₽
                                    </div>
                                </div>
                            </li>
                        @endforeach
                    </ul>
                    <div class="card-footer">
                        <div class="d-flex justify-content-between align-items-center">
                            <span class="h5 mb-0">Итого:</span>
                            <span class="h3 mb-0 text-primary">
                                {{ number_format($cart->total, 2) }} ₽
                            </span>
                        </div>
                    </div>
                </div>
            </div>
            
            <div class="mt-3">
                <a href="{{ route('cart.index') }}" class="btn btn-outline-secondary w-100">
                    ← Вернуться в корзину
                </a>
            </div>
        </div>
    </div>
</div>
@endsection

@push('scripts')
<script>
// Phone mask
const phoneInput = document.getElementById('phone');

phoneInput.addEventListener('input', function(e) {
    let value = e.target.value.replace(/\D/g, '');
    
    if (value.length > 0) {
        if (value[0] === '7' || value[0] === '8') {
            value = value.substring(1);
        }
        
        let formattedValue = '+7';
        
        if (value.length > 0) {
            formattedValue += ' (' + value.substring(0, 3);
        }
        if (value.length >= 3) {
            formattedValue += ') ' + value.substring(3, 6);
        }
        if (value.length >= 6) {
            formattedValue += '-' + value.substring(6, 8);
        }
        if (value.length >= 8) {
            formattedValue += '-' + value.substring(8, 10);
        }
        
        e.target.value = formattedValue;
    }
});

// Form validation
document.getElementById('checkout-form').addEventListener('submit', function(e) {
    const submitBtn = this.querySelector('button[type="submit"]');
    submitBtn.disabled = true;
    submitBtn.innerHTML = '<span class="spinner-border spinner-border-sm"></span> Оформление...';
});
</script>
@endpush
```

### Страница успешного заказа

resources/views/checkout/success.blade.php:

```blade
@extends('layouts.app')

@section('title', 'Заказ оформлен')

@section('content')
<div class="container py-5">
    <div class="row justify-content-center">
        <div class="col-md-8 text-center">
            <div class="display-1 text-success mb-3">✓</div>
            <h1 class="mb-3">Заказ успешно оформлен!</h1>
            <p class="lead text-muted mb-4">
                Номер вашего заказа: <strong>#{{ $order->id }}</strong>
            </p>
            
            <div class="card text-start mb-4">
                <div class="card-header">
                    <h5 class="mb-0">Информация о заказе</h5>
                </div>
                <div class="card-body">
                    <div class="row">
                        <div class="col-sm-6">
                            <p class="mb-1 text-muted">Получатель:</p>
                            <p class="fw-bold">{{ $order->name }}</p>
                        </div>
                        <div class="col-sm-6">
                            <p class="mb-1 text-muted">Телефон:</p>
                            <p class="fw-bold">{{ $order->phone }}</p>
                        </div>
                    </div>
                    <div class="row">
                        <div class="col-sm-6">
                            <p class="mb-1 text-muted">Сумма заказа:</p>
                            <p class="fw-bold text-primary">{{ number_format($order->total, 2) }} ₽</p>
                        </div>
                        <div class="col-sm-6">
                            <p class="mb-1 text-muted">Статус:</p>
                            <span class="badge bg-warning">{{ $order->status_label }}</span>
                        </div>
                    </div>
                </div>
            </div>
            
            <p class="text-muted mb-4">
                Мы свяжемся с вами в ближайшее время для подтверждения заказа.
            </p>
            
            <a href="{{ route('home') }}" class="btn btn-primary">
                Вернуться на главную
            </a>
        </div>
    </div>
</div>
@endsection
```

### Роуты

```php
<?php

use App\Http\Controllers\CheckoutController;
use Illuminate\Support\Facades\Route;

Route::prefix('checkout')->name('checkout.')->group(function () {
    Route::get('/', [CheckoutController::class, 'index'])->name('index');
    Route::post('/', [CheckoutController::class, 'store'])->name('store');
    Route::get('/success/{order}', [CheckoutController::class, 'success'])->name('success');
});
```

### Flash сообщения в layout

resources/views/layouts/app.blade.php:

```blade
<!DOCTYPE html>
<html lang="ru">
<head>
    {{-- ... --}}
</head>
<body>
    @include('layouts.partials.header')
    
    <main>
        @if(session('success'))
            <div class="container mt-3">
                <div class="alert alert-success alert-dismissible fade show">
                    {{ session('success') }}
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            </div>
        @endif
        
        @if(session('error'))
            <div class="container mt-3">
                <div class="alert alert-danger alert-dismissible fade show">
                    {{ session('error') }}
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            </div>
        @endif
        
        @yield('content')
    </main>
    
    @include('layouts.partials.footer')
    
    @stack('scripts')
</body>
</html>
```

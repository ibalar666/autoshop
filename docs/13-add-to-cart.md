## ЭТАП 13. Добавление в корзину

### AJAX добавление через fetch API

Кнопка "Добавить в корзину" отправляет асинхронный запрос без перезагрузки страницы.

### AddToCartRequest (Form Request)

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class AddToCartRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'article' => ['required', 'string', 'max:50'],
            'brand' => ['required', 'string', 'max:50'],
            'name' => ['required', 'string', 'max:255'],
            'price' => ['required', 'numeric', 'min:0'],
            'supplier' => ['required', 'string', 'max:50'],
            'quantity' => ['sometimes', 'integer', 'min:1', 'max:100'],
            'metadata' => ['sometimes', 'array'],
        ];
    }

    public function messages(): array
    {
        return [
            'article.required' => 'Артикул обязателен',
            'brand.required' => 'Бренд обязателен',
            'price.required' => 'Цена обязательна',
            'price.min' => 'Цена не может быть отрицательной',
            'quantity.min' => 'Минимальное количество: 1',
            'quantity.max' => 'Максимальное количество: 100',
        ];
    }
}
```

### CartController (метод add)

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Http\Requests\AddToCartRequest;
use App\Services\CartService;
use Illuminate\Http\JsonResponse;

class CartController extends Controller
{
    public function __construct(
        private readonly CartService $cartService,
    ) {}

    public function add(AddToCartRequest $request): JsonResponse
    {
        $validated = $request->validated();
        
        $cart = $this->cartService->getCurrentCart();
        $item = $this->cartService->addItem($cart, $validated);

        return response()->json([
            'success' => true,
            'message' => 'Товар добавлен в корзину',
            'cartCount' => $cart->fresh()->items_count,
            'cartTotal' => $cart->fresh()->total,
            'item' => [
                'id' => $item->id,
                'name' => $item->name,
                'brand' => $item->brand,
                'article' => $item->article,
                'price' => $item->price,
                'quantity' => $item->quantity,
                'subtotal' => $item->subtotal,
            ],
        ]);
    }
}
```

### JavaScript для AJAX

resources/js/cart.js:

```javascript
/**
 * Cart functionality
 */
class CartManager {
    constructor() {
        this.csrfToken = document.querySelector('meta[name="csrf-token"]')?.content;
        this.init();
    }

    init() {
        document.querySelectorAll('.add-to-cart').forEach(button => {
            button.addEventListener('click', (e) => this.handleAddToCart(e));
        });
    }

    async handleAddToCart(event) {
        const button = event.currentTarget;
        
        // Prevent double-click
        if (button.disabled) return;
        button.disabled = true;
        
        const data = {
            article: button.dataset.article,
            brand: button.dataset.brand,
            name: button.dataset.name,
            price: parseFloat(button.dataset.price),
            supplier: button.dataset.supplier,
            quantity: parseInt(button.dataset.quantity) || 1,
        };

        try {
            const response = await fetch('/cart/add', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'X-CSRF-TOKEN': this.csrfToken,
                    'X-Requested-With': 'XMLHttpRequest',
                    'Accept': 'application/json',
                },
                body: JSON.stringify(data),
            });

            const result = await response.json();

            if (result.success) {
                this.showNotification('success', result.message);
                this.updateCartCounter(result.cartCount);
                this.updateCartTotal(result.cartTotal);
                this.animateButton(button);
            } else {
                this.showNotification('error', result.message || 'Ошибка добавления');
            }
        } catch (error) {
            console.error('Cart error:', error);
            this.showNotification('error', 'Произошла ошибка. Попробуйте позже.');
        } finally {
            button.disabled = false;
        }
    }

    updateCartCounter(count) {
        const counters = document.querySelectorAll('.cart-counter');
        counters.forEach(counter => {
            counter.textContent = count;
            counter.classList.add('pulse');
            setTimeout(() => counter.classList.remove('pulse'), 500);
        });
    }

    updateCartTotal(total) {
        const totalElements = document.querySelectorAll('.cart-total');
        const formatter = new Intl.NumberFormat('ru-RU', {
            style: 'currency',
            currency: 'RUB',
        });
        totalElements.forEach(el => {
            el.textContent = formatter.format(total);
        });
    }

    showNotification(type, message) {
        // Simple notification implementation
        const notification = document.createElement('div');
        notification.className = `alert alert-${type === 'success' ? 'success' : 'danger'} position-fixed`;
        notification.style.cssText = 'top: 20px; right: 20px; z-index: 9999; min-width: 300px;';
        notification.textContent = message;
        
        document.body.appendChild(notification);
        
        setTimeout(() => {
            notification.remove();
        }, 3000);
    }

    animateButton(button) {
        button.classList.add('btn-success');
        button.classList.remove('btn-primary');
        button.innerHTML = '✓ Добавлено';
        
        setTimeout(() => {
            button.classList.add('btn-primary');
            button.classList.remove('btn-success');
            button.innerHTML = 'В корзину';
        }, 1500);
    }
}

// Initialize on DOM ready
document.addEventListener('DOMContentLoaded', () => {
    window.cartManager = new CartManager();
});
```

### HTML кнопка добавления

```blade
{{-- В карточке товара --}}
<button 
    class="btn btn-primary add-to-cart"
    data-article="{{ $product->article }}"
    data-brand="{{ $product->brand }}"
    data-name="{{ $product->name }}"
    data-price="{{ $offer->price }}"
    data-supplier="{{ $offer->supplier }}"
    data-quantity="1"
>
    В корзину
</button>

{{-- В списке товаров --}}
<button 
    class="btn btn-sm btn-outline-primary add-to-cart"
    data-article="OC90"
    data-brand="MAHLE"
    data-name="Фильтр масляный"
    data-price="450.00"
    data-supplier="Autoopt"
>
    В корзину
</button>
```

### Индикатор корзины в шапке

```blade
{{-- resources/views/layouts/partials/header.blade.php --}}
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <div class="container">
        <a class="navbar-brand" href="{{ route('home') }}">Autoshop</a>
        
        <div class="navbar-nav ms-auto">
            <a href="{{ route('cart.index') }}" class="nav-link position-relative">
                🛒 Корзина
                @php
                    $cartCount = app(\App\Services\CartService::class)->getCurrentCart()->items_count;
                @endphp
                <span class="cart-counter badge bg-danger position-absolute top-0 start-100 translate-middle {{ $cartCount > 0 ? '' : 'd-none' }}"
                      id="cart-counter">
                    {{ $cartCount }}
                </span>
            </a>
        </div>
    </div>
</nav>
```

### CSS анимация

```css
/* resources/css/cart.css */

.cart-counter {
    transition: transform 0.2s ease;
}

.cart-counter.pulse {
    animation: pulse 0.5s ease;
}

@keyframes pulse {
    0%, 100% {
        transform: translate(-50%, -50%) scale(1);
    }
    50% {
        transform: translate(-50%, -50%) scale(1.3);
    }
}

.add-to-cart:disabled {
    opacity: 0.7;
    cursor: not-allowed;
}
```

### Vite настройка

vite.config.js:

```javascript
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: [
                'resources/css/app.css',
                'resources/js/app.js',
                'resources/js/cart.js',
            ],
            refresh: true,
        }),
    ],
});
```

resources/js/app.js:

```javascript
import './cart';
```

### Роут

```php
<?php

use App\Http\Controllers\CartController;
use Illuminate\Support\Facades\Route;

Route::post('/cart/add', [CartController::class, 'add'])
    ->name('cart.add')
    ->middleware('web');
```

### Обработка ошибок

```javascript
// Расширенная обработка ошибок
async handleAddToCart(event) {
    const button = event.currentTarget;
    
    if (button.disabled) return;
    button.disabled = true;

    try {
        const response = await fetch('/cart/add', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-TOKEN': this.csrfToken,
                'X-Requested-With': 'XMLHttpRequest',
            },
            body: JSON.stringify(data),
        });

        // Handle HTTP errors
        if (!response.ok) {
            if (response.status === 419) {
                throw new Error('Сессия истекла. Обновите страницу.');
            }
            if (response.status === 422) {
                const errors = await response.json();
                throw new Error(Object.values(errors.errors).flat().join(', '));
            }
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }

        const result = await response.json();
        // ... success handling
        
    } catch (error) {
        this.showNotification('error', error.message);
    } finally {
        button.disabled = false;
    }
}
```

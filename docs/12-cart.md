## ЭТАП 12. Корзина

### Миграции

Таблица `carts`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('carts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->nullable()->constrained()->onDelete('cascade');
            $table->string('session_id')->nullable()->index();
            $table->timestamps();
            
            $table->index(['user_id', 'session_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('carts');
    }
};
```

Таблица `cart_items`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('cart_items', function (Blueprint $table) {
            $table->id();
            $table->foreignId('cart_id')->constrained()->onDelete('cascade');
            $table->string('article');
            $table->string('brand');
            $table->string('name');
            $table->string('supplier');
            $table->decimal('price', 10, 2);
            $table->unsignedInteger('quantity')->default(1);
            $table->json('metadata')->nullable();
            $table->timestamps();
            
            $table->unique(['cart_id', 'article', 'brand', 'supplier']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('cart_items');
    }
};
```

### Модели

Cart.php:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Cart extends Model
{
    use HasFactory;

    protected $fillable = ['user_id', 'session_id'];

    public function items(): HasMany
    {
        return $this->hasMany(CartItem::class);
    }

    public function getTotalAttribute(): float
    {
        return $this->items->sum(fn($item) => $item->price * $item->quantity);
    }

    public function getItemsCountAttribute(): int
    {
        return $this->items->sum('quantity');
    }
}
```

CartItem.php:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class CartItem extends Model
{
    use HasFactory;

    protected $fillable = [
        'cart_id', 'article', 'brand', 'name', 
        'supplier', 'price', 'quantity', 'metadata'
    ];

    protected $casts = [
        'price' => 'float',
        'metadata' => 'array',
    ];

    public function cart(): BelongsTo
    {
        return $this->belongsTo(Cart::class);
    }

    public function getSubtotalAttribute(): float
    {
        return $this->price * $this->quantity;
    }
}
```

### CartController

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Services\CartService;
use Illuminate\View\View;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\RedirectResponse;

class CartController extends Controller
{
    public function __construct(
        private readonly CartService $cartService,
    ) {}

    public function index(): View
    {
        $cart = $this->cartService->getCurrentCart();
        
        return view('cart.index', compact('cart'));
    }

    public function add(\Illuminate\Http\Request $request): JsonResponse
    {
        $validated = $request->validate([
            'article' => 'required|string',
            'brand' => 'required|string',
            'name' => 'required|string',
            'price' => 'required|numeric|min:0',
            'supplier' => 'required|string',
            'quantity' => 'integer|min:1|max:100',
        ]);

        $cart = $this->cartService->getCurrentCart();
        $item = $this->cartService->addItem($cart, $validated);

        return response()->json([
            'success' => true,
            'message' => 'Товар добавлен в корзину',
            'cartCount' => $cart->fresh()->items_count,
            'item' => $item,
        ]);
    }

    public function update(\Illuminate\Http\Request $request, int $itemId): JsonResponse
    {
        $validated = $request->validate([
            'quantity' => 'required|integer|min:0|max:100',
        ]);

        $cart = $this->cartService->getCurrentCart();
        $this->cartService->updateQuantity($cart, $itemId, $validated['quantity']);

        return response()->json([
            'success' => true,
            'cartCount' => $cart->fresh()->items_count,
            'cartTotal' => $cart->fresh()->total,
        ]);
    }

    public function remove(int $itemId): RedirectResponse
    {
        $cart = $this->cartService->getCurrentCart();
        $this->cartService->removeItem($cart, $itemId);

        return redirect()->route('cart.index')
            ->with('success', 'Товар удалён из корзины');
    }

    public function clear(): RedirectResponse
    {
        $cart = $this->cartService->getCurrentCart();
        $this->cartService->clearCart($cart);

        return redirect()->route('cart.index')
            ->with('success', 'Корзина очищена');
    }
}
```

### CartService

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Models\Cart;
use App\Models\CartItem;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Session;

class CartService
{
    public function getCurrentCart(): Cart
    {
        $userId = Auth::id();
        $sessionId = Session::getId();

        if ($userId) {
            $cart = Cart::firstOrCreate(
                ['user_id' => $userId],
                ['session_id' => $sessionId]
            );
            
            // Merge guest cart if exists
            $guestCart = Cart::where('session_id', $sessionId)
                ->whereNull('user_id')
                ->first();
                
            if ($guestCart && $guestCart->id !== $cart->id) {
                $this->mergeCarts($cart, $guestCart);
            }
        } else {
            $cart = Cart::firstOrCreate(
                ['session_id' => $sessionId],
                ['user_id' => null]
            );
        }

        return $cart->load('items');
    }

    public function addItem(Cart $cart, array $data): CartItem
    {
        $item = $cart->items()
            ->where('article', $data['article'])
            ->where('brand', $data['brand'])
            ->where('supplier', $data['supplier'])
            ->first();

        if ($item) {
            $item->increment('quantity', $data['quantity'] ?? 1);
            return $item->fresh();
        }

        return $cart->items()->create([
            'article' => $data['article'],
            'brand' => $data['brand'],
            'name' => $data['name'],
            'supplier' => $data['supplier'],
            'price' => $data['price'],
            'quantity' => $data['quantity'] ?? 1,
            'metadata' => $data['metadata'] ?? null,
        ]);
    }

    public function updateQuantity(Cart $cart, int $itemId, int $quantity): void
    {
        $item = $cart->items()->findOrFail($itemId);
        
        if ($quantity <= 0) {
            $item->delete();
        } else {
            $item->update(['quantity' => $quantity]);
        }
    }

    public function removeItem(Cart $cart, int $itemId): void
    {
        $cart->items()->where('id', $itemId)->delete();
    }

    public function clearCart(Cart $cart): void
    {
        $cart->items()->delete();
    }

    private function mergeCarts(Cart $target, Cart $source): void
    {
        foreach ($source->items as $item) {
            $this->addItem($target, $item->toArray());
        }
        
        $source->delete();
    }
}
```

### Blade шаблон

resources/views/cart/index.blade.php:

```blade
@extends('layouts.app')

@section('title', 'Корзина')

@section('content')
<div class="container py-5">
    <h1 class="mb-4">Корзина</h1>
    
    @if($cart->items->isEmpty())
        <div class="text-center py-5">
            <div class="display-1 text-muted mb-3">🛒</div>
            <h3 class="text-muted">Ваша корзина пуста</h3>
            <p class="text-muted">Добавьте товары из каталога</p>
            <a href="{{ route('home') }}" class="btn btn-primary">Продолжить покупки</a>
        </div>
    @else
        <div class="row">
            <div class="col-lg-8">
                <div class="card">
                    <div class="card-body p-0">
                        <div class="table-responsive">
                            <table class="table table-hover mb-0">
                                <thead class="table-light">
                                    <tr>
                                        <th>Товар</th>
                                        <th>Поставщик</th>
                                        <th>Цена</th>
                                        <th>Кол-во</th>
                                        <th>Сумма</th>
                                        <th></th>
                                    </tr>
                                </thead>
                                <tbody>
                                    @foreach($cart->items as $item)
                                        <tr>
                                            <td>
                                                <div>
                                                    <strong>{{ $item->name }}</strong>
                                                    <div class="text-muted small">
                                                        {{ $item->brand }} {{ $item->article }}
                                                    </div>
                                                </div>
                                            </td>
                                            <td>{{ $item->supplier }}</td>
                                            <td>{{ number_format($item->price, 2) }} ₽</td>
                                            <td>
                                                <input type="number" 
                                                       class="form-control form-control-sm quantity-input" 
                                                       value="{{ $item->quantity }}" 
                                                       min="1" max="100"
                                                       data-item-id="{{ $item->id }}"
                                                       style="width: 70px;">
                                            </td>
                                            <td class="fw-bold item-subtotal" data-item-id="{{ $item->id }}">
                                                {{ number_format($item->subtotal, 2) }} ₽
                                            </td>
                                            <td>
                                                <form action="{{ route('cart.remove', $item->id) }}" method="POST" class="d-inline">
                                                    @csrf
                                                    @method('DELETE')
                                                    <button type="submit" class="btn btn-sm btn-outline-danger">
                                                        ✕
                                                    </button>
                                                </form>
                                            </td>
                                        </tr>
                                    @endforeach
                                </tbody>
                            </table>
                        </div>
                    </div>
                </div>
                
                <div class="mt-3">
                    <form action="{{ route('cart.clear') }}" method="POST" class="d-inline">
                        @csrf
                        @method('DELETE')
                        <button type="submit" class="btn btn-outline-danger btn-sm">
                            Очистить корзину
                        </button>
                    </form>
                    <a href="{{ route('home') }}" class="btn btn-outline-secondary btn-sm">
                        Продолжить покупки
                    </a>
                </div>
            </div>
            
            <div class="col-lg-4">
                <div class="card">
                    <div class="card-header">
                        <h5 class="mb-0">Итого</h5>
                    </div>
                    <div class="card-body">
                        <div class="d-flex justify-content-between mb-2">
                            <span>Товаров:</span>
                            <span id="cart-items-count">{{ $cart->items_count }}</span>
                        </div>
                        <hr>
                        <div class="d-flex justify-content-between mb-3">
                            <span class="fw-bold">Общая сумма:</span>
                            <span class="fw-bold fs-4 text-primary" id="cart-total">
                                {{ number_format($cart->total, 2) }} ₽
                            </span>
                        </div>
                        <a href="{{ route('checkout.index') }}" class="btn btn-success btn-lg w-100">
                            Оформить заказ
                        </a>
                    </div>
                </div>
            </div>
        </div>
    @endif
</div>
@endsection

@push('scripts')
<script>
document.querySelectorAll('.quantity-input').forEach(input => {
    let timeout;
    
    input.addEventListener('change', function() {
        clearTimeout(timeout);
        const itemId = this.dataset.itemId;
        const quantity = parseInt(this.value);
        
        if (quantity < 1) {
            this.value = 1;
            return;
        }
        
        timeout = setTimeout(() => {
            fetch(`{{ route('cart.index') }}/${itemId}`, {
                method: 'PATCH',
                headers: {
                    'Content-Type': 'application/json',
                    'X-CSRF-TOKEN': '{{ csrf_token() }}',
                    'X-Requested-With': 'XMLHttpRequest'
                },
                body: JSON.stringify({ quantity })
            })
            .then(response => response.json())
            .then(result => {
                if (result.success) {
                    updateCartDisplay(result);
                    updateItemSubtotal(itemId, quantity);
                }
            })
            .catch(error => console.error('Error:', error));
        }, 500);
    });
});

function updateCartDisplay(data) {
    document.getElementById('cart-items-count').textContent = data.cartCount;
    document.getElementById('cart-total').textContent = 
        new Intl.NumberFormat('ru-RU', { style: 'currency', currency: 'RUB' })
            .format(data.cartTotal);
}

function updateItemSubtotal(itemId, quantity) {
    const row = document.querySelector(`[data-item-id="${itemId}"]`).closest('tr');
    const price = parseFloat(row.querySelector('td:nth-child(3)').textContent.replace(/[^\d.]/g, ''));
    const subtotal = price * quantity;
    
    document.querySelector(`.item-subtotal[data-item-id="${itemId}"]`).textContent = 
        new Intl.NumberFormat('ru-RU', { style: 'currency', currency: 'RUB' })
            .format(subtotal);
}
</script>
@endpush
```

### Роуты

```php
<?php

use App\Http\Controllers\CartController;
use Illuminate\Support\Facades\Route;

Route::prefix('cart')->name('cart.')->group(function () {
    Route::get('/', [CartController::class, 'index'])->name('index');
    Route::post('/add', [CartController::class, 'add'])->name('add');
    Route::patch('/{itemId}', [CartController::class, 'update'])->name('update');
    Route::delete('/{itemId}', [CartController::class, 'remove'])->name('remove');
    Route::delete('/', [CartController::class, 'clear'])->name('clear');
});
```

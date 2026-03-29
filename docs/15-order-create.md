## ЭТАП 15. Создание заказа

### Миграции

Таблица `orders`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->nullable()->constrained()->onDelete('set null');
            $table->string('name');
            $table->string('phone');
            $table->string('email')->nullable();
            $table->text('comment')->nullable();
            $table->decimal('total', 12, 2)->default(0);
            $table->enum('status', ['pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled'])
                ->default('pending');
            $table->string('external_order_id')->nullable()->index();
            $table->json('supplier_response')->nullable();
            $table->timestamps();
            
            $table->index(['status', 'created_at']);
            $table->index('phone');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};
```

Таблица `order_items`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('order_items', function (Blueprint $table) {
            $table->id();
            $table->foreignId('order_id')->constrained()->onDelete('cascade');
            $table->string('article');
            $table->string('brand');
            $table->string('name');
            $table->string('supplier');
            $table->decimal('price', 10, 2);
            $table->unsignedInteger('quantity');
            $table->decimal('subtotal', 10, 2);
            $table->string('external_item_id')->nullable();
            $table->json('metadata')->nullable();
            $table->enum('status', ['pending', 'reserved', 'shipped', 'delivered', 'cancelled'])
                ->default('pending');
            $table->timestamps();
            
            $table->index(['order_id', 'status']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('order_items');
    }
};
```

### Модели

Order.php:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Order extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id', 'name', 'phone', 'email', 'comment',
        'total', 'status', 'external_order_id', 'supplier_response',
    ];

    protected $casts = [
        'total' => 'float',
        'supplier_response' => 'array',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    public function getStatusLabelAttribute(): string
    {
        return match($this->status) {
            'pending' => 'Ожидает подтверждения',
            'confirmed' => 'Подтверждён',
            'processing' => 'В обработке',
            'shipped' => 'Отправлен',
            'delivered' => 'Доставлен',
            'cancelled' => 'Отменён',
            default => $this->status,
        };
    }

    public function getStatusColorAttribute(): string
    {
        return match($this->status) {
            'pending' => 'warning',
            'confirmed' => 'info',
            'processing' => 'primary',
            'shipped' => 'secondary',
            'delivered' => 'success',
            'cancelled' => 'danger',
            default => 'light',
        };
    }
}
```

OrderItem.php:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class OrderItem extends Model
{
    use HasFactory;

    protected $fillable = [
        'order_id', 'article', 'brand', 'name', 'supplier',
        'price', 'quantity', 'subtotal', 'external_item_id', 'metadata', 'status',
    ];

    protected $casts = [
        'price' => 'float',
        'subtotal' => 'float',
        'metadata' => 'array',
    ];

    public function order(): BelongsTo
    {
        return $this->belongsTo(Order::class);
    }
}
```

### OrderService

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\DTO\OrderDTO;
use App\Models\Cart;
use App\Models\Order;
use App\Models\OrderItem;
use App\Suppliers\Autoopt\AutooptClient;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class OrderService
{
    public function __construct(
        private readonly AutooptClient $supplierClient,
    ) {}

    public function createFromCart(Cart $cart, array $customerData): Order
    {
        return DB::transaction(function () use ($cart, $customerData) {
            // Calculate total
            $total = $cart->items->sum(fn($item) => $item->price * $item->quantity);

            // Create order
            $order = Order::create([
                'user_id' => Auth::id(),
                'name' => $customerData['name'],
                'phone' => $customerData['phone'],
                'email' => $customerData['email'] ?? null,
                'comment' => $customerData['comment'] ?? null,
                'total' => $total,
                'status' => 'pending',
            ]);

            // Create order items
            foreach ($cart->items as $cartItem) {
                OrderItem::create([
                    'order_id' => $order->id,
                    'article' => $cartItem->article,
                    'brand' => $cartItem->brand,
                    'name' => $cartItem->name,
                    'supplier' => $cartItem->supplier,
                    'price' => $cartItem->price,
                    'quantity' => $cartItem->quantity,
                    'subtotal' => $cartItem->price * $cartItem->quantity,
                    'metadata' => $cartItem->metadata,
                    'status' => 'pending',
                ]);
            }

            // Send order to supplier API
            $this->sendToSupplier($order);

            return $order->load('items');
        });
    }

    private function sendToSupplier(Order $order): void
    {
        try {
            $orderData = $this->prepareSupplierPayload($order);
            
            $response = $this->supplierClient->createOrder($orderData);
            
            $order->update([
                'external_order_id' => $response['order_id'] ?? null,
                'supplier_response' => $response,
                'status' => $response['status'] === 'confirmed' ? 'confirmed' : 'pending',
            ]);

            // Update order items with external IDs
            if (!empty($response['items'])) {
                foreach ($response['items'] as $itemData) {
                    $order->items()
                        ->where('article', $itemData['article'])
                        ->where('supplier', $itemData['supplier'])
                        ->update([
                            'external_item_id' => $itemData['item_id'] ?? null,
                            'status' => $itemData['status'] ?? 'pending',
                        ]);
                }
            }

            Log::info('Order sent to supplier', [
                'order_id' => $order->id,
                'external_order_id' => $order->external_order_id,
            ]);

        } catch (\Exception $e) {
            Log::error('Failed to send order to supplier', [
                'order_id' => $order->id,
                'error' => $e->getMessage(),
            ]);
            
            // Order remains in 'pending' status for manual processing
            // Could queue retry job here
        }
    }

    private function prepareSupplierPayload(Order $order): array
    {
        return [
            'external_id' => (string) $order->id,
            'customer' => [
                'name' => $order->name,
                'phone' => $order->phone,
                'email' => $order->email,
            ],
            'comment' => $order->comment,
            'items' => $order->items->map(fn($item) => [
                'article' => $item->article,
                'brand' => $item->brand,
                'name' => $item->name,
                'quantity' => $item->quantity,
                'price' => $item->price,
            ])->toArray(),
        ];
    }

    public function findOrder(int $id): ?Order
    {
        return Order::with('items')->find($id);
    }

    public function updateStatus(Order $order, string $status): void
    {
        $order->update(['status' => $status]);
    }

    public function cancelOrder(Order $order): void
    {
        DB::transaction(function () use ($order) {
            // Cancel at supplier if external order exists
            if ($order->external_order_id) {
                try {
                    $this->supplierClient->cancelOrder($order->external_order_id);
                } catch (\Exception $e) {
                    Log::error('Failed to cancel order at supplier', [
                        'order_id' => $order->id,
                        'error' => $e->getMessage(),
                    ]);
                }
            }

            $order->update(['status' => 'cancelled']);
            $order->items()->update(['status' => 'cancelled']);
        });
    }
}
```

### Интеграция с API поставщика

```php
<?php

declare(strict_types=1);

namespace App\Suppliers\Autoopt;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class AutooptClient
{
    private string $baseUrl;
    private string $apiKey;

    public function __construct()
    {
        $this->baseUrl = config('services.autoopt.url');
        $this->apiKey = config('services.autoopt.key');
    }

    public function createOrder(array $orderData): array
    {
        $response = Http::withHeaders([
            'Authorization' => 'Bearer ' . $this->apiKey,
            'Accept' => 'application/json',
        ])->post($this->baseUrl . '/api/v2/orders', $orderData);

        if (!$response->successful()) {
            Log::error('Autoopt API create order failed', [
                'status' => $response->status(),
                'body' => $response->body(),
            ]);
            throw new \Exception('Failed to create order: ' . $response->body());
        }

        return $response->json();
    }

    public function cancelOrder(string $externalOrderId): array
    {
        $response = Http::withHeaders([
            'Authorization' => 'Bearer ' . $this->apiKey,
        ])->post($this->baseUrl . '/api/v2/orders/' . $externalOrderId . '/cancel');

        if (!$response->successful()) {
            throw new \Exception('Failed to cancel order: ' . $response->body());
        }

        return $response->json();
    }

    public function getOrderStatus(string $externalOrderId): array
    {
        $response = Http::withHeaders([
            'Authorization' => 'Bearer ' . $this->apiKey,
        ])->get($this->baseUrl . '/api/v2/orders/' . $externalOrderId);

        if (!$response->successful()) {
            throw new \Exception('Failed to get order status: ' . $response->body());
        }

        return $response->json();
    }
}
```

### Конфигурация

config/services.php:

```php
<?php

return [
    // ...

    'autoopt' => [
        'url' => env('AUTOOPT_URL', 'https://beta.autoopt.ru'),
        'key' => env('AUTOOPT_API_KEY'),
    ],
];
```

.env:

```
AUTOOPT_URL=https://beta.autoopt.ru
AUTOOPT_API_KEY=your_api_key_here
```

### Order DTO

```php
<?php

declare(strict_types=1);

namespace App\DTO;

readonly class OrderDTO
{
    public function __construct(
        public string $supplier,
        public string $externalId,
        public string $status,
        public float $total,
        public array $items = [],
    ) {}

    public static function fromArray(array $data): self
    {
        return new self(
            supplier: $data['supplier'],
            externalId: $data['external_id'],
            status: $data['status'],
            total: (float) ($data['total'] ?? 0),
            items: $data['items'] ?? [],
        );
    }
}

readonly class OrderItemDTO
{
    public function __construct(
        public string $article,
        public string $brand,
        public string $name,
        public float $price,
        public int $quantity,
        public float $subtotal,
    ) {}
}
```

### OrderResource (MoonShine Admin)

```php
<?php

declare(strict_types=1);

namespace App\MoonShine\Resources;

use App\Models\Order;
use MoonShine\Laravel\Fields\Relationships\HasMany;
use MoonShine\Laravel\Resources\ModelResource;
use MoonShine\UI\Components\Layout\Box;
use MoonShine\UI\Fields\Date;
use MoonShine\UI\Fields\ID;
use MoonShine\UI\Fields\Number;
use MoonShine\UI\Fields\Select;
use MoonShine\UI\Fields\Text;
use MoonShine\UI\Fields\Textarea;

class OrderResource extends ModelResource
{
    protected string $model = Order::class;
    protected string $title = 'Заказы';

    public function fields(): array
    {
        return [
            ID::make()->sortable(),
            
            Box::make('Клиент', [
                Text::make('Имя', 'name'),
                Text::make('Телефон', 'phone'),
                Text::make('Email', 'email'),
            ]),
            
            Box::make('Информация о заказе', [
                Number::make('Сумма', 'total'),
                Select::make('Статус', 'status')
                    ->options([
                        'pending' => 'Ожидает',
                        'confirmed' => 'Подтверждён',
                        'processing' => 'В обработке',
                        'shipped' => 'Отправлен',
                        'delivered' => 'Доставлен',
                        'cancelled' => 'Отменён',
                    ]),
                Text::make('Внешний ID', 'external_order_id'),
                Date::make('Создан', 'created_at'),
            ]),
            
            Textarea::make('Комментарий', 'comment'),
            
            HasMany::make('Товары', 'items', OrderItemResource::class),
        ];
    }
}
```

### Синхронизация статусов (Job)

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Models\Order;
use App\Services\OrderService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class SyncOrderStatus implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        private readonly int $orderId,
    ) {}

    public function handle(OrderService $orderService): void
    {
        $order = Order::find($this->orderId);
        
        if (!$order || !$order->external_order_id) {
            return;
        }

        $status = $orderService->syncStatusFromSupplier($order);
        
        if ($status) {
            $orderService->updateStatus($order, $status);
        }
    }
}
```

### Планировщик

routes/console.php:

```php
<?php

use App\Jobs\SyncOrderStatus;
use App\Models\Order;
use Illuminate\Support\Facades\Schedule;

// Sync pending orders every 15 minutes
Schedule::call(function () {
    $orders = Order::whereIn('status', ['pending', 'confirmed', 'processing'])
        ->whereNotNull('external_order_id')
        ->get();
    
    foreach ($orders as $order) {
        SyncOrderStatus::dispatch($order->id);
    }
})->everyFifteenMinutes();
```

# Модели Eloquent

## ЭТАП 3. Создание моделей

Команды для генерации моделей с миграциями:

```bash
php artisan make:model Category -m
php artisan make:model Brand -m
php artisan make:model Cart -m
php artisan make:model CartItem -m
php artisan make:model Order -m
php artisan make:model OrderItem -m
php artisan make:model PartCache -m
php artisan make:model Markup -m
php artisan make:model SearchLog -m
php artisan make:model SeoPage -m
php artisan make:model ApiSetting -m
```

## Модель Category

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Category extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'name',
        'slug',
        'description',
        'image',
        'parent_id',
        'is_active',
    ];

    protected $casts = [
        'is_active' => 'boolean',
    ];

    public function parent(): BelongsTo
    {
        return $this->belongsTo(self::class, 'parent_id');
    }

    public function children(): HasMany
    {
        return $this->hasMany(self::class, 'parent_id');
    }
}
```

## Модель Brand

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Brand extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'name',
        'slug',
        'logo',
        'is_active',
    ];

    protected $casts = [
        'is_active' => 'boolean',
    ];

    public function partCaches(): HasMany
    {
        return $this->hasMany(PartCache::class, 'brand');
    }

    public function markups(): HasMany
    {
        return $this->hasMany(Markup::class, 'brand');
    }
}
```

## Модель Cart

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Cart extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'user_id',
        'session_id',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(CartItem::class);
    }
}
```

## Модель CartItem

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\SoftDeletes;

class CartItem extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'cart_id',
        'article',
        'brand',
        'price',
        'quantity',
        'supplier',
        'delivery_days',
    ];

    protected $casts = [
        'price' => 'decimal:2',
        'quantity' => 'integer',
        'delivery_days' => 'integer',
    ];

    public function cart(): BelongsTo
    {
        return $this->belongsTo(Cart::class);
    }
}
```

## Модель Order

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Order extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'user_id',
        'status',
        'total',
        'customer_name',
        'phone',
        'email',
        'comment',
    ];

    protected $casts = [
        'total' => 'decimal:2',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }
}
```

## Модель OrderItem

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\SoftDeletes;

class OrderItem extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'order_id',
        'article',
        'brand',
        'name',
        'price',
        'quantity',
        'supplier',
    ];

    protected $casts = [
        'price' => 'decimal:2',
        'quantity' => 'integer',
    ];

    public function order(): BelongsTo
    {
        return $this->belongsTo(Order::class);
    }
}
```

## Модель PartCache

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class PartCache extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'article',
        'brand',
        'name',
        'data_json',
        'expires_at',
    ];

    protected $casts = [
        'data_json' => 'array',
        'expires_at' => 'datetime',
    ];

    public function isExpired(): bool
    {
        return $this->expires_at->isPast();
    }
}
```

## Модель Markup

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Markup extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'brand',
        'category',
        'percent',
        'is_active',
    ];

    protected $casts = [
        'percent' => 'decimal:2',
        'is_active' => 'boolean',
    ];

    public function calculatePrice(float $basePrice): float
    {
        return $basePrice * (1 + $this->percent / 100);
    }
}
```

## Модель SearchLog

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class SearchLog extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'query',
        'results_count',
        'ip',
        'user_agent',
    ];

    protected $casts = [
        'results_count' => 'integer',
    ];
}
```

## Модель SeoPage

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class SeoPage extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'url',
        'title',
        'description',
        'h1',
        'content',
    ];

    public function scopeByUrl(string $url)
    {
        return $this->where('url', $url);
    }
}
```

## Модель ApiSetting

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class ApiSetting extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'supplier',
        'api_key',
        'api_url',
        'is_active',
    ];

    protected $casts = [
        'is_active' => 'boolean',
    ];

    protected $hidden = [
        'api_key',
    ];

    public function isActive(): bool
    {
        return $this->is_active;
    }

    public function getApiKey(): string
    {
        return decrypt($this->api_key);
    }

    public function setApiKey(string $key): void
    {
        $this->api_key = encrypt($key);
    }
}
```

## Общие рекомендации

1. **Typed Properties**: Все свойства имеют строгую типизацию
2. **SoftDeletes**: Все модели используют мягкое удаление
3. **Casts**: Правильное приведение типов для полей (boolean, decimal:2, array, datetime)
4. **Relations**: Явно типизированные методы отношений (BelongsTo, HasMany и др.)
5. **Declare Strict Types**: Используйте `declare(strict_types=1);` в начале файла
6. **Fillable**: Все поля, которые должны быть массово назначаемы, указаны в $fillable

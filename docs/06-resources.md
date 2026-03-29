# MoonShine ресурсы

## ЭТАП 6. MoonShine ресурсы

Создаём CRUD ресурсы для админ-панели.

### Команды генерации

```bash
php artisan moonshine:resource Category
php artisan moonshine:resource Brand
php artisan moonshine:resource Order
php artisan moonshine:resource User
php artisan moonshine:resource Markup
php artisan moonshine:resource SeoPage
php artisan moonshine:resource ApiSetting
```

### CategoryResource

```php
<?php

declare(strict_types=1);

namespace App\MoonShine\Resources;

use App\Models\Category;
use MoonShine\Attributes\Icon;
use MoonShine\Attributes\Resource;
use MoonShine\Components\Tabs;
use MoonShine\Components\Tabs\Tab;
use MoonShine\Fields\Checkbox;
use MoonShine\Fields\ID;
use MoonShine\Fields\Image;
use MoonShine\Fields\Number;
use MoonShine\Fields\Relationships\BelongsTo;
use MoonShine\Fields\Relationships\HasMany;
use MoonShine\Fields\Slug;
use MoonShine\Fields\Text;
use MoonShine\Fields\Textarea;
use MoonShine\Pages\Crud\DetailPage;
use MoonShine\Pages\Crud\FormPage;
use MoonShine\Pages\Crud\IndexPage;
use MoonShine\Resources\ModelResource;

#[Resource(
    alias: 'category',
    model: Category::class,
    title: 'Категории',
    icon: 'heroicons.outline.folder'
)]
#[Icon('heroicons.outline.folder')]
class CategoryResource extends ModelResource
{
    protected string $model = Category::class;

    protected string $title = 'Категории';

    protected array $with = ['parent', 'children'];

    protected bool $withPolicy = false;

    public function fields(): array
    {
        return [
            Tabs::make([
                Tab::make('Основное', [
                    ID::make()->sortable(),

                    BelongsTo::make('Родительская категория', 'parent', 'name')
                        ->nullable()
                        ->searchable(),

                    Text::make('Название', 'name')
                        ->required()
                        ->sortable(),

                    Slug::make('Slug', 'slug')
                        ->from('name')
                        ->unique()
                        ->required(),

                    Textarea::make('Описание', 'description')
                        ->nullable(),

                    Image::make('Изображение', 'image')
                        ->nullable()
                        ->disk('public')
                        ->dir('categories'),
                ]),

                Tab::make('Настройки', [
                    Number::make('Порядок сортировки', 'sort_order')
                        ->default(0)
                        ->sortable(),

                    Checkbox::make('Активна', 'is_active')
                        ->default(true),
                ]),

                Tab::make('Подкатегории', [
                    HasMany::make('Подкатегории', 'children', resource: new self())
                        ->creatable()
                        ->asyncSearch(),
                ]),
            ]),
        ];
    }

    public function rules($item): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'slug' => ['required', 'string', 'max:255', 'unique:categories,slug,' . ($item?->id ?? 'NULL')],
            'description' => ['nullable', 'string', 'max:5000'],
            'parent_id' => ['nullable', 'exists:categories,id'],
            'sort_order' => ['integer', 'min:0'],
            'is_active' => ['boolean'],
        ];
    }

    public function filters(): array
    {
        return [
            BelongsTo::make('Родительская категория', 'parent', 'name')
                ->nullable()
                ->searchable(),

            Checkbox::make('Только активные', 'is_active'),
        ];
    }

    public function search(): array
    {
        return ['id', 'name', 'slug'];
    }
}
```

### BrandResource

```php
<?php

declare(strict_types=1);

namespace App\MoonShine\Resources;

use App\Models\Brand;
use MoonShine\Attributes\Icon;
use MoonShine\Attributes\Resource;
use MoonShine\Fields\Checkbox;
use MoonShine\Fields\ID;
use MoonShine\Fields\Image;
use MoonShine\Fields\Slug;
use MoonShine\Fields\Text;
use MoonShine\Resources\ModelResource;

#[Resource(
    alias: 'brand',
    model: Brand::class,
    title: 'Бренды',
    icon: 'heroicons.outline.tag'
)]
#[Icon('heroicons.outline.tag')]
class BrandResource extends ModelResource
{
    protected string $model = Brand::class;

    protected string $title = 'Бренды';

    public function fields(): array
    {
        return [
            ID::make()->sortable(),

            Text::make('Название', 'name')
                ->required()
                ->sortable(),

            Slug::make('Slug', 'slug')
                ->from('name')
                ->unique()
                ->required(),

            Image::make('Логотип', 'logo')
                ->nullable()
                ->disk('public')
                ->dir('brands'),

            Checkbox::make('Активен', 'is_active')
                ->default(true),
        ];
    }

    public function rules($item): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'slug' => ['required', 'string', 'max:255', 'unique:brands,slug,' . ($item?->id ?? 'NULL')],
            'logo' => ['nullable', 'image', 'max:2048'],
            'is_active' => ['boolean'],
        ];
    }

    public function filters(): array
    {
        return [
            Text::make('Название', 'name'),

            Checkbox::make('Только активные', 'is_active'),
        ];
    }

    public function search(): array
    {
        return ['id', 'name', 'slug'];
    }
}
```

### OrderResource

```php
<?php

declare(strict_types=1);

namespace App\MoonShine\Resources;

use App\Models\Order;
use MoonShine\Attributes\Icon;
use MoonShine\Attributes\Resource;
use MoonShine\Components\Tabs;
use MoonShine\Components\Tabs\Tab;
use MoonShine\Fields\Date;
use MoonShine\Fields\Enum;
use MoonShine\Fields\ID;
use MoonShine\Fields\Number;
use MoonShine\Fields\Relationships\BelongsTo;
use MoonShine\Fields\Relationships\HasMany;
use MoonShine\Fields\Text;
use MoonShine\Fields\Textarea;
use MoonShine\Resources\ModelResource;
use App\Enums\OrderStatus;

#[Resource(
    alias: 'order',
    model: Order::class,
    title: 'Заказы',
    icon: 'heroicons.outline.shopping-cart'
)]
#[Icon('heroicons.outline.shopping-cart')]
class OrderResource extends ModelResource
{
    protected string $model = Order::class;

    protected string $title = 'Заказы';

    protected array $with = ['user', 'items'];

    public function fields(): array
    {
        return [
            Tabs::make([
                Tab::make('Основное', [
                    ID::make()->sortable(),

                    Enum::make('Статус', 'status')
                        ->options([
                            'pending' => 'Новый',
                            'processing' => 'В обработке',
                            'completed' => 'Выполнен',
                            'cancelled' => 'Отменён',
                        ])
                        ->sortable(),

                    Number::make('Сумма', 'total')
                        ->readonly()
                        ->sortable(),

                    BelongsTo::make('Пользователь', 'user', 'name')
                        ->nullable()
                        ->searchable(),

                    Date::make('Дата создания', 'created_at')
                        ->readonly()
                        ->sortable(),

                    Date::make('Дата обновления', 'updated_at')
                        ->readonly()
                        ->sortable(),
                ]),

                Tab::make('Клиент', [
                    Text::make('Имя клиента', 'customer_name')
                        ->required(),

                    Text::make('Телефон', 'phone')
                        ->required(),

                    Text::make('Email', 'email')
                        ->nullable()
                        ->mask('email'),

                    Textarea::make('Комментарий', 'comment')
                        ->nullable(),
                ]),

                Tab::make('Товары', [
                    HasMany::make('Товары заказа', 'items', resource: OrderItemResource::class)
                        ->creatable()
                        ->fields([
                            ID::make()->sortable(),
                            Text::make('Артикул', 'article'),
                            Text::make('Бренд', 'brand'),
                            Text::make('Название', 'name'),
                            Number::make('Цена', 'price'),
                            Number::make('Количество', 'quantity'),
                        ]),
                ]),
            ]),
        ];
    }

    public function rules($item): array
    {
        return [
            'status' => ['required', 'in:pending,processing,completed,cancelled'],
            'customer_name' => ['required', 'string', 'max:255'],
            'phone' => ['required', 'string', 'max:20'],
            'email' => ['nullable', 'email', 'max:255'],
            'comment' => ['nullable', 'string', 'max:5000'],
        ];
    }

    public function filters(): array
    {
        return [
            Enum::make('Статус', 'status')
                ->options([
                    'pending' => 'Новый',
                    'processing' => 'В обработке',
                    'completed' => 'Выполнен',
                    'cancelled' => 'Отменён',
                ]),

            BelongsTo::make('Пользователь', 'user', 'name')
                ->nullable()
                ->searchable(),

            Date::make('От', 'created_at')
                ->placeholder('Дата от'),

            Date::make('До', 'created_at')
                ->placeholder('Дата до'),
        ];
    }

    public function search(): array
    {
        return ['id', 'customer_name', 'phone', 'email'];
    }
}
```

### UserResource

```php
<?php

declare(strict_types=1);

namespace App\MoonShine\Resources;

use App\Models\User;
use MoonShine\Attributes\Icon;
use MoonShine\Attributes\Resource;
use MoonShine\Fields\Checkbox;
use MoonShine\Fields\Date;
use MoonShine\Fields\Email;
use MoonShine\Fields\ID;
use MoonShine\Fields\Password;
use MoonShine\Fields\Text;
use MoonShine\Resources\ModelResource;

#[Resource(
    alias: 'user',
    model: User::class,
    title: 'Пользователи',
    icon: 'heroicons.outline.users'
)]
#[Icon('heroicons.outline.users')]
class UserResource extends ModelResource
{
    protected string $model = User::class;

    protected string $title = 'Пользователи';

    public function fields(): array
    {
        return [
            ID::make()->sortable(),

            Text::make('Имя', 'name')
                ->required()
                ->sortable(),

            Email::make('Email', 'email')
                ->required()
                ->sortable(),

            Password::make('Пароль', 'password')
                ->when(fn($data) => $data === null)
                ->required(),

            Checkbox::make('Администратор', 'is_admin')
                ->default(false),

            Date::make('Email подтверждён', 'email_verified_at')
                ->nullable()
                ->readonly()
                ->sortable(),

            Date::make('Создан', 'created_at')
                ->readonly()
                ->sortable(),
        ];
    }

    public function rules($item): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'max:255', 'unique:users,email,' . ($item?->id ?? 'NULL')],
            'password' => [$item === null ? 'required' : 'nullable', 'string', 'min:8'],
            'is_admin' => ['boolean'],
        ];
    }

    public function filters(): array
    {
        return [
            Text::make('Имя', 'name'),
            Text::make('Email', 'email'),
            Checkbox::make('Только администраторы', 'is_admin'),
        ];
    }

    public function search(): array
    {
        return ['id', 'name', 'email'];
    }
}
```

### MarkupResource

```php
<?php

declare(strict_types=1);

namespace App\MoonShine\Resources;

use App\Models\Markup;
use MoonShine\Attributes\Icon;
use MoonShine\Attributes\Resource;
use MoonShine\Fields\Checkbox;
use MoonShine\Fields\ID;
use MoonShine\Fields\Number;
use MoonShine\Fields\Text;
use MoonShine\Resources\ModelResource;

#[Resource(
    alias: 'markup',
    model: Markup::class,
    title: 'Наценки',
    icon: 'heroicons.outline.percent-badge'
)]
#[Icon('heroicons.outline.percent-badge')]
class MarkupResource extends ModelResource
{
    protected string $model = Markup::class;

    protected string $title = 'Наценки';

    public function fields(): array
    {
        return [
            ID::make()->sortable(),

            Text::make('Бренд', 'brand')
                ->nullable()
                ->sortable(),

            Text::make('Категория', 'category')
                ->nullable()
                ->sortable(),

            Number::make('Процент', 'percent')
                ->required()
                ->min(0)
                ->max(1000)
                ->step(0.01)
                ->suffix('%')
                ->sortable(),

            Checkbox::make('Активна', 'is_active')
                ->default(true),
        ];
    }

    public function rules($item): array
    {
        return [
            'brand' => ['nullable', 'string', 'max:255'],
            'category' => ['nullable', 'string', 'max:255'],
            'percent' => ['required', 'numeric', 'min:0', 'max:1000'],
            'is_active' => ['boolean'],
        ];
    }

    public function filters(): array
    {
        return [
            Text::make('Бренд', 'brand'),
            Text::make('Категория', 'category'),
            Checkbox::make('Только активные', 'is_active'),
        ];
    }

    public function search(): array
    {
        return ['id', 'brand', 'category'];
    }
}
```

### SeoPageResource

```php
<?php

declare(strict_types=1);

namespace App\MoonShine\Resources;

use App\Models\SeoPage;
use MoonShine\Attributes\Icon;
use MoonShine\Attributes\Resource;
use MoonShine\Fields\ID;
use MoonShine\Fields\Slug;
use MoonShine\Fields\Text;
use MoonShine\Fields\Textarea;
use MoonShine\Resources\ModelResource;

#[Resource(
    alias: 'seo-page',
    model: SeoPage::class,
    title: 'SEO страницы',
    icon: 'heroicons.outline.magnifying-glass'
)]
#[Icon('heroicons.outline.magnifying-glass')]
class SeoPageResource extends ModelResource
{
    protected string $model = SeoPage::class;

    protected string $title = 'SEO страницы';

    public function fields(): array
    {
        return [
            ID::make()->sortable(),

            Text::make('URL', 'url')
                ->required()
                ->placeholder('/catalog/filters')
                ->sortable(),

            Text::make('Title', 'title')
                ->required(),

            Textarea::make('Description', 'description')
                ->nullable(),

            Text::make('H1', 'h1')
                ->nullable(),

            Textarea::make('Контент', 'content')
                ->nullable(),
        ];
    }

    public function rules($item): array
    {
        return [
            'url' => ['required', 'string', 'max:500', 'unique:seo_pages,url,' . ($item?->id ?? 'NULL')],
            'title' => ['required', 'string', 'max:255'],
            'description' => ['nullable', 'string', 'max:1000'],
            'h1' => ['nullable', 'string', 'max:255'],
            'content' => ['nullable', 'string'],
        ];
    }

    public function filters(): array
    {
        return [
            Text::make('URL', 'url'),
            Text::make('Title', 'title'),
        ];
    }

    public function search(): array
    {
        return ['id', 'url', 'title', 'h1'];
    }
}
```

### ApiSettingResource

```php
<?php

declare(strict_types=1);

namespace App\MoonShine\Resources;

use App\Models\ApiSetting;
use MoonShine\Attributes\Icon;
use MoonShine\Attributes\Resource;
use MoonShine\Fields\Checkbox;
use MoonShine\Fields\ID;
use MoonShine\Fields\Password;
use MoonShine\Fields\Text;
use MoonShine\Fields\Url;
use MoonShine\Resources\ModelResource;

#[Resource(
    alias: 'api-setting',
    model: ApiSetting::class,
    title: 'Настройки API',
    icon: 'heroicons.outline.cog-6-tooth'
)]
#[Icon('heroicons.outline.cog-6-tooth')]
class ApiSettingResource extends ModelResource
{
    protected string $model = ApiSetting::class;

    protected string $title = 'Настройки API';

    public function fields(): array
    {
        return [
            ID::make()->sortable(),

            Text::make('Поставщик', 'supplier')
                ->required()
                ->sortable(),

            Password::make('API Key', 'api_key')
                ->required(),

            Url::make('API URL', 'api_url')
                ->required(),

            Checkbox::make('Активен', 'is_active')
                ->default(true),
        ];
    }

    public function rules($item): array
    {
        return [
            'supplier' => ['required', 'string', 'max:255'],
            'api_key' => ['required', 'string'],
            'api_url' => ['required', 'url', 'max:500'],
            'is_active' => ['boolean'],
        ];
    }

    public function filters(): array
    {
        return [
            Text::make('Поставщик', 'supplier'),
            Checkbox::make('Только активные', 'is_active'),
        ];
    }

    public function search(): array
    {
        return ['id', 'supplier', 'api_url'];
    }
}
```

### Регистрация ресурсов

```php
<?php

// config/moonshine.php

return [
    // ...

    'resources' => [
        \App\MoonShine\Resources\CategoryResource::class,
        \App\MoonShine\Resources\BrandResource::class,
        \App\MoonShine\Resources\OrderResource::class,
        \App\MoonShine\Resources\UserResource::class,
        \App\MoonShine\Resources\MarkupResource::class,
        \App\MoonShine\Resources\SeoPageResource::class,
        \App\MoonShine\Resources\ApiSettingResource::class,
    ],

    // ...
];
```

### Рекомендации

1. **Иконки**: Используйте Heroicons для единообразия интерфейса
2. **Валидация**: Всегда определяйте `rules()` для защиты данных
3. **Фильтры**: Добавляйте фильтры для удобного поиска записей
4. **Русские названия**: Все поля должны иметь понятные русские названия
5. **Табы**: Группируйте поля по смыслу в табы для лучшей организации
6. **Relations**: Используйте `with()` для eager loading связанных данных
7. **Поиск**: Определяйте поля для поиска через метод `search()`

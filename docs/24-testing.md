# ЭТАП 24. Тестирование

## Установка PHPUnit

```bash
composer require --dev phpunit/phpunit

# PHPUnit уже включён в Laravel
```

## Файл phpunit.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true"
>
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="BCRYPT_ROUNDS" value="4"/>
        <env name="CACHE_DRIVER" value="array"/>
        <env name="DB_CONNECTION" value="sqlite"/>
        <env name="DB_DATABASE" value=":memory:"/>
        <env name="MAIL_MAILER" value="array"/>
        <env name="QUEUE_CONNECTION" value="sync"/>
        <env name="SESSION_DRIVER" value="array"/>
        <env name="TELESCOPE_ENABLED" value="false"/>
    </php>
</phpunit>
```

## Тестирование поиска

```php
<?php

namespace Tests\Feature;

use App\Models\Part;
use App\Services\SearchService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class SearchTest extends TestCase
{
    use RefreshDatabase;

    public function test_search_by_name()
    {
        // Создаем тестовые данные
        Part::factory()->create([
            'name' => 'Масляный фильтр',
            'brand' => 'Bosch',
            'sku' => 'F001',
            'is_active' => true,
        ]);

        Part::factory()->create([
            'name' => 'Тормозная колодка',
            'brand' => 'Bosch',
            'sku' => 'B001',
            'is_active' => true,
        ]);

        // Выполняем поиск
        $response = $this->get('/search?q=фильтр');

        // Проверяем результат
        $response->assertStatus(200);
        $response->assertSee('Масляный фильтр');
        $response->assertDontSee('Тормозная колодка');
    }

    public function test_search_by_brand()
    {
        Part::factory()->create([
            'name' => 'Масляный фильтр',
            'brand' => 'Bosch',
            'is_active' => true,
        ]);

        Part::factory()->create([
            'name' => 'Воздушный фильтр',
            'brand' => 'Mann',
            'is_active' => true,
        ]);

        $response = $this->get('/search?q=Bosch');

        $response->assertStatus(200);
        $response->assertSee('Bosch');
    }

    public function test_search_by_sku()
    {
        Part::factory()->create([
            'name' => 'Масляный фильтр',
            'brand' => 'Bosch',
            'sku' => 'F001',
            'is_active' => true,
        ]);

        $response = $this->get('/search?q=F001');

        $response->assertStatus(200);
        $response->assertSee('F001');
    }

    public function test_search_empty_query_redirects_to_catalog()
    {
        $response = $this->get('/search?q=');

        $response->assertRedirect('/catalog');
    }

    public function test_search_only_active_parts()
    {
        Part::factory()->create([
            'name' => 'Активный товар',
            'is_active' => true,
        ]);

        Part::factory()->create([
            'name' => 'Неактивный товар',
            'is_active' => false,
        ]);

        $response = $this->get('/search?q=товар');

        $response->assertSee('Активный товар');
        $response->assertDontSee('Неактивный товар');
    }
}
```

## Тестирование корзины

```php
<?php

namespace Tests\Feature;

use App\Models\Part;
use App\Models\User;
use App\Services\CartService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class CartTest extends TestCase
{
    use RefreshDatabase;

    protected CartService $cartService;

    protected function setUp(): void
    {
        parent::setUp();
        $this->cartService = app(CartService::class);
    }

    public function test_add_to_cart()
    {
        $part = Part::factory()->create([
            'price' => 1000,
            'stock' => 10,
            'is_active' => true,
        ]);

        $response = $this->post('/cart/add', [
            'part_id' => $part->id,
            'quantity' => 2,
        ]);

        $response->assertRedirect();

        $cart = session('cart', []);
        $this->assertArrayHasKey($part->id, $cart);
        $this->assertEquals(2, $cart[$part->id]['quantity']);
    }

    public function test_remove_from_cart()
    {
        $part = Part::factory()->create();

        // Добавляем в корзину
        session(['cart' => [$part->id => ['id' => $part->id, 'quantity' => 1]]]);

        $response = $this->post('/cart/remove', [
            'part_id' => $part->id,
        ]);

        $response->assertRedirect();

        $cart = session('cart', []);
        $this->assertArrayNotHasKey($part->id, $cart);
    }

    public function test_update_quantity()
    {
        $part = Part::factory()->create([
            'stock' => 10,
        ]);

        session(['cart' => [$part->id => ['id' => $part->id, 'quantity' => 1]]]);

        $response = $this->post('/cart/update', [
            'part_id' => $part->id,
            'quantity' => 5,
        ]);

        $response->assertRedirect();

        $cart = session('cart');
        $this->assertEquals(5, $cart[$part->id]['quantity']);
    }

    public function test_cannot_add_more_than_stock()
    {
        $part = Part::factory()->create([
            'price' => 1000,
            'stock' => 5,
            'is_active' => true,
        ]);

        $response = $this->post('/cart/add', [
            'part_id' => $part->id,
            'quantity' => 10,
        ]);

        $response->assertSessionHasErrors();

        $cart = session('cart', []);
        $this->assertArrayNotHasKey($part->id, $cart);
    }

    public function test_calculate_total()
    {
        $part1 = Part::factory()->create(['price' => 1000]);
        $part2 = Part::factory()->create(['price' => 500]);

        $this->cartService->add($part1->id, 2);
        $this->cartService->add($part2->id, 1);

        $total = $this->cartService->total();
        $this->assertEquals(2500, $total);
    }

    public function test_clear_cart()
    {
        $part = Part::factory()->create();
        session(['cart' => [$part->id => ['id' => $part->id, 'quantity' => 1]]]);

        $response = $this->post('/cart/clear');

        $response->assertRedirect();

        $cart = session('cart');
        $this->assertEmpty($cart);
    }
}
```

## Тестирование заказов

```php
<?php

namespace Tests\Feature;

use App\Models\Part;
use App\Models\User;
use App\Models\Order;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class OrderTest extends TestCase
{
    use RefreshDatabase;

    public function test_create_order()
    {
        $user = User::factory()->create();
        $part = Part::factory()->create([
            'price' => 1000,
            'stock' => 10,
        ]);

        // Добавляем товар в корзину
        session([
            'cart' => [
                $part->id => [
                    'id' => $part->id,
                    'quantity' => 2,
                    'price' => $part->price,
                ]
            ]
        ]);

        $response = $this->actingAs($user)
            ->post('/checkout', [
                'name' => 'Иван Иванов',
                'phone' => '+7 999 123 45 67',
                'email' => 'test@example.com',
                'delivery_method' => 'courier',
                'address' => 'г. Москва, ул. Тестовая, д. 1',
            ]);

        $response->assertRedirect('/checkout/success');

        $this->assertDatabaseHas('orders', [
            'user_id' => $user->id,
            'status' => 'pending',
            'total' => 2000,
        ]);
    }

    public function test_order_reduces_stock()
    {
        $user = User::factory()->create();
        $part = Part::factory()->create([
            'price' => 1000,
            'stock' => 10,
        ]);

        session([
            'cart' => [
                $part->id => [
                    'id' => $part->id,
                    'quantity' => 3,
                    'price' => $part->price,
                ]
            ]
        ]);

        $this->actingAs($user)
            ->post('/checkout', [
                'name' => 'Иван Иванов',
                'phone' => '+7 999 123 45 67',
                'email' => 'test@example.com',
                'delivery_method' => 'courier',
            ]);

        $part->refresh();
        $this->assertEquals(7, $part->stock);
    }

    public function test_order_validation()
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->post('/checkout', [
                'name' => '',
                'phone' => '',
                'email' => '',
            ]);

        $response->assertSessionHasErrors(['name', 'phone', 'email']);
    }

    public function test_cannot_create_order_with_empty_cart()
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->post('/checkout', [
                'name' => 'Иван Иванов',
                'phone' => '+7 999 123 45 67',
                'email' => 'test@example.com',
            ]);

        $response->assertRedirect('/cart');
    }

    public function test_order_status_update()
    {
        $admin = User::factory()->admin()->create();
        $order = Order::factory()->create(['status' => 'pending']);

        $response = $this->actingAs($admin)
            ->put("/admin/orders/{$order->id}", [
                'status' => 'processing',
                'tracking_number' => 'TRK123456',
            ]);

        $response->assertRedirect();

        $order->refresh();
        $this->assertEquals('processing', $order->status);
        $this->assertEquals('TRK123456', $order->tracking_number);
    }

    public function test_guest_can_create_order()
    {
        $part = Part::factory()->create([
            'price' => 1000,
            'stock' => 10,
        ]);

        session([
            'cart' => [
                $part->id => [
                    'id' => $part->id,
                    'quantity' => 1,
                    'price' => $part->price,
                ]
            ]
        ]);

        $response = $this->post('/checkout', [
            'name' => 'Иван Иванов',
            'phone' => '+7 999 123 45 67',
            'email' => 'guest@example.com',
            'delivery_method' => 'courier',
        ]);

        $response->assertRedirect('/checkout/success');

        $this->assertDatabaseHas('orders', [
            'name' => 'Иван Иванов',
            'email' => 'guest@example.com',
        ]);
    }
}
```

## Тестирование API

```php
<?php

namespace Tests\Feature;

use App\Models\Part;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ApiTest extends TestCase
{
    use RefreshDatabase;

    public function test_get_parts_list()
    {
        Part::factory()->count(20)->create();

        $response = $this->getJson('/api/parts');

        $response->assertStatus(200)
            ->assertJsonCount(15, 'data'); // 15 per page by default
    }

    public function test_get_single_part()
    {
        $part = Part::factory()->create();

        $response = $this->getJson("/api/parts/{$part->id}");

        $response->assertStatus(200)
            ->assertJson([
                'data' => [
                    'id' => $part->id,
                    'name' => $part->name,
                ]
            ]);
    }

    public function test_search_api()
    {
        Part::factory()->create(['name' => 'Масляный фильтр']);
        Part::factory()->create(['name' => 'Тормозная колодка']);

        $response = $this->getJson('/api/search?q=фильтр');

        $response->assertStatus(200)
            ->assertJsonCount(1, 'data');
    }

    public function test_authenticated_can_add_to_cart()
    {
        $user = User::factory()->create();
        $part = Part::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/cart/add', [
                'part_id' => $part->id,
                'quantity' => 1,
            ]);

        $response->assertStatus(200);
    }

    public function test_unauthenticated_cannot_create_order()
    {
        $response = $this->postJson('/api/orders', [
            'name' => 'Иван Иванов',
            'phone' => '+7 999 123 45 67',
        ]);

        $response->assertStatus(401);
    }
}
```

## Unit тесты

### Тест PriceService

```php
<?php

namespace Tests\Unit;

use App\Services\PriceService;
use App\Models\Markup;
use App\Models\Part;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PriceServiceTest extends TestCase
{
    use RefreshDatabase;

    protected PriceService $priceService;

    protected function setUp(): void
    {
        parent::setUp();
        $this->priceService = app(PriceService::class);
    }

    public function test_calculate_price_with_specific_markup()
    {
        Markup::factory()->create([
            'brand' => 'Bosch',
            'category' => 'Фильтры',
            'percent' => 20,
        ]);

        $price = $this->priceService->getPriceWithMarkup(1000, 'Bosch', 'Фильтры');

        $this->assertEquals(1200, $price);
    }

    public function test_calculate_price_with_default_markup()
    {
        Markup::factory()->create([
            'brand' => '*',
            'category' => '*',
            'percent' => 15,
        ]);

        $price = $this->priceService->getPriceWithMarkup(1000, 'Unknown', 'Unknown');

        $this->assertEquals(1150, $price);
    }

    public function test_recalculate_part_price()
    {
        $part = Part::factory()->create([
            'base_price' => 1000,
            'price' => 1000,
        ]);

        Markup::factory()->create([
            'brand' => '*',
            'category' => '*',
            'percent' => 20,
        ]);

        $this->priceService->recalculatePrice($part);

        $part->refresh();
        $this->assertEquals(1200, $part->price);
    }
}
```

### Тест SearchService

```php
<?php

namespace Tests\Unit;

use App\Services\SearchService;
use App\Models\Part;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class SearchServiceTest extends TestCase
{
    use RefreshDatabase;

    protected SearchService $searchService;

    protected function setUp(): void
    {
        parent::setUp();
        $this->searchService = app(SearchService::class);
    }

    public function test_search_returns_results()
    {
        Part::factory()->create([
            'name' => 'Масляный фильтр',
            'brand' => 'Bosch',
            'is_active' => true,
        ]);

        $results = $this->searchService->search('фильтр');

        $this->assertCount(1, $results['data']);
        $this->assertEquals(1, $results['total']);
    }

    public function test_search_uses_cache()
    {
        $part = Part::factory()->create([
            'name' => 'Масляный фильтр',
            'is_active' => true,
        ]);

        // Первый вызов
        $results1 = $this->searchService->search('фильтр');

        // Второй вызов должен использовать кеширование
        $results2 = $this->searchService->search('фильтр');

        $this->assertEquals($results1, $results2);
    }
}
```

## Запуск тестов

```bash
# Запустить все тесты
./vendor/bin/phpunit

# Запустить определенный тест
./vendor/bin/phpunit tests/Feature/SearchTest.php

# Запустить определенный тестовый метод
./vendor/bin/phpunit --filter test_search_by_name

# Запустить с выводом информации
./vendor/bin/phpunit --testdox

# Запустить с покрытием кода
./vendor/bin/phpunit --coverage-html coverage

# Запустить только feature тесты
./vendor/bin/phpunit --testsuite=Feature

# Запустить только unit тесты
./vendor/bin/phpunit --testsuite=Unit

# Показать список всех тестов
./vendor/bin/phpunit --list-tests
```

## Factory для тестов

```php
<?php

namespace Database\Factories;

use App\Models\Part;
use Illuminate\Database\Eloquent\Factories\Factory;

class PartFactory extends Factory
{
    protected $model = Part::class;

    public function definition()
    {
        $brands = ['Bosch', 'Mann', 'Valeo', 'Sachs', 'Lemforder'];
        $categories = ['Фильтры', 'Тормоза', 'Амортизаторы', 'Рулевое управление'];

        return [
            'name' => fake()->words(3, true),
            'sku' => strtoupper(fake()->bothify('???###')),
            'brand' => fake()->randomElement($brands),
            'category' => fake()->randomElement($categories),
            'price' => fake()->randomFloat(2, 500, 15000),
            'discount' => fake()->randomFloat(2, 0, 30),
            'stock' => fake()->numberBetween(0, 100),
            'is_active' => true,
            'views' => fake()->numberBetween(0, 1000),
            'sales_count' => fake()->numberBetween(0, 100),
            'description' => fake()->paragraph(),
        ];
    }
}
```

## Консольные команды

```bash
# Создать тест
php artisan make:test SearchTest

# Создать feature тест
php artisan make:test CartTest

# Создать unit тест
php artisan make:test PriceServiceTest --unit

# Создать factory
php artisan make:model Part -mf

# Запустить тесты
php artisan test
```

## Git

```bash
git add .
git commit -m "feat: добавлены тесты для поиска, корзины и заказов"
git push
```

# Архитектура проекта

Описание архитектурных решений и принципов организации кода проекта Autoshop.

## Схема архитектуры

```
┌─────────────────────────────────────────────────────────────┐
│                        HTTP Layer                          │
│  Routes → Middleware → Controllers → Form Requests         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Business Logic Layer                     │
│                      Services                               │
│  - ProductService                                           │
│  - OrderService                                             │
│  - SearchService                                            │
│  - CacheService                                             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Data Transfer Layer                       │
│                       DTO                                   │
│  - ProductDTO                                               │
│  - SearchRequestDTO                                         │
│  - SupplierResponseDTO                                      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  External Integration                       │
│                   Suppliers API                             │
│  - AutooptApiClient                                         │
│  - SupplierClientInterface                                  │
│  - ResponseMapper                                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     Data Storage                            │
│              Cache / Database / Files                       │
│  - Redis (кэш, сессии, очереди)                            │
│  - MySQL (постоянные данные)                               │
└─────────────────────────────────────────────────────────────┘
```

## Описание 5 слоёв

### 1. Controllers (HTTP Layer)

Отвечают за:
- Приём HTTP-запросов
- Валидацию входных данных через Form Requests
- Вызов сервисов
- Формирование HTTP-ответов

```php
class ProductController extends Controller
{
    public function __construct(
        private readonly ProductService $productService
    ) {}

    public function search(SearchRequest $request): JsonResponse
    {
        $products = $this->productService->search(
            SearchRequestDTO::fromRequest($request)
        );
        
        return response()->json($products);
    }
}
```

### 2. Services (Business Logic)

Содержат бизнес-логику:
- Координация между слоями
- Правила обработки данных
- Кэширование
- Обработка ошибок

```php
class ProductService
{
    public function __construct(
        private readonly AutooptApiClient $autooptClient,
        private readonly CacheService $cache
    ) {}

    public function search(SearchRequestDTO $dto): array
    {
        $cacheKey = "search:{$dto->article}";
        
        return $this->cache->remember($cacheKey, 300, function () use ($dto) {
            $response = $this->autooptClient->search($dto);
            return ProductDTO::collectionFromResponse($response);
        });
    }
}
```

### 3. DTO (Data Transfer Objects)

Обеспечивают типобезопасность передачи данных:
- Неизменяемые объекты
- Валидация данных
- Преобразование форматов

```php
readonly class ProductDTO
{
    public function __construct(
        public string $id,
        public string $name,
        public string $article,
        public float $price,
        public int $stock,
        public string $supplier
    ) {}

    public static function fromArray(array $data): self
    {
        return new self(
            id: $data['id'],
            name: $data['name'],
            article: $data['article'],
            price: $data['price'],
            stock: $data['stock'],
            supplier: $data['supplier']
        );
    }
}
```

### 4. Suppliers API

Абстракция над внешними API:
- Единый интерфейс для всех поставщиков
- Обработка аутентификации
- Маппинг ответов
- Retry-логика

```php
interface SupplierClientInterface
{
    public function search(SearchRequestDTO $dto): array;
    public function getProduct(string $id): ?ProductDTO;
}

class AutooptApiClient implements SupplierClientInterface
{
    public function __construct(
        private readonly HttpClient $http,
        private readonly string $apiKey
    ) {}
}
```

### 5. Cache/DB

Хранение данных:
- **Redis** — кэш, сессии, очереди
- **MySQL** — постоянные данные

## Структура папок

```
app/
├── Http/
│   ├── Controllers/           # HTTP-контроллеры
│   │   ├── Api/              # API-контроллеры
│   │   └── Admin/            # Контроллеры админки MoonShine
│   ├── Requests/             # Form Request валидация
│   └── Resources/            # API Resources
├── Services/                 # Бизнес-логика
│   ├── ProductService.php
│   ├── SearchService.php
│   └── CacheService.php
├── DTO/                      # Data Transfer Objects
│   ├── ProductDTO.php
│   ├── SearchRequestDTO.php
│   └── SupplierResponseDTO.php
├── Clients/                  # HTTP-клиенты
│   ├── Contracts/
│   │   └── SupplierClientInterface.php
│   └── AutooptApiClient.php
├── Models/                   # Eloquent модели
│   ├── Product.php
│   ├── Order.php
│   └── Supplier.php
├── Providers/                # Service Providers
└── Exceptions/               # Кастомные исключения
```

## Поток поиска запчасти

```
┌──────────┐     ┌─────────────┐     ┌───────────┐     ┌─────────┐     ┌─────────────┐     ┌────────────┐
│   User   │────→│  Controller │────→│  Service  │────→│   DTO   │────→│ Supplier API │────→│  Cache/DB  │
└──────────┘     └─────────────┘     └───────────┘     └─────────┘     └─────────────┘     └────────────┘
                      │                    │                 ↑                │
                      │                    │                 │                │
                      │                    └─────────────────┘                │
                      │                                                       │
                      └───────────────────────────────────────────────────────┘
                                           (Response Flow)
```

### Пример потока поиска запчасти

1. **User** отправляет запрос `GET /api/search?article=OC90`
2. **Controller** (`SearchController@index`) валидирует параметры
3. **Service** (`SearchService`) координирует поиск по поставщикам
4. **DTO** (`SearchRequestDTO`, `ProductDTO`) передают структурированные данные
5. **Supplier API** (`AutooptApiClient`) выполняет HTTP-запрос к внешнему API
6. **Cache** сохраняет результат на 5 минут
7. **Response** возвращается пользователю в едином формате

## Поток оформления заказа

```
User Request
     ↓
OrderController (валидация данных заказа)
     ↓
OrderService (создание заказа, расчёт стоимости)
     ↓
ProductDTO / OrderDTO (передача данных)
     ↓
AutooptApiClient (резервирование товара у поставщика)
     ↓
Database (сохранение заказа)
     ↓
Response (подтверждение заказа пользователю)
```

## Преимущества архитектуры

### Масштабируемость

- **Новые поставщики**: добавление нового API требует только реализации `SupplierClientInterface`
- **Новые endpoints**: создание нового контроллера и сервиса без изменения существующего кода
- **Горизонтальное масштабирование**: сервисы можно выносить в микросервисы

### Тестируемость

- **Unit-тесты**: каждый слой тестируется изолированно через моки
- **Интеграционные тесты**: подмена реальных клиентов на фейковые реализации
- **Contract-тесты**: проверка соответствия DTO спецификациям API

### Поддерживаемость

- **Чёткие границы**: изменения в одном слое минимально влияют на другие
- **Принцип единственной ответственности**: каждый класс решает одну задачу
- **Dependency Injection**: легко заменять реализации

### Пример добавления нового поставщика

```php
// 1. Создать клиент
class NewSupplierClient implements SupplierClientInterface
{
    // Реализация методов
}

// 2. Зарегистрировать в сервисе
class SearchService
{
    public function __construct(
        private readonly array $suppliers // [AutooptApiClient, NewSupplierClient]
    ) {}
}

// 3. Контроллеры остаются без изменений
```

Никаких изменений в контроллерах или API endpoints не требуется.

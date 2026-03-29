## ЭТАП 8. DTO слой

DTO защищает систему от изменений API поставщика.

```php
<?php

declare(strict_types=1);

namespace App\DTO;

readonly class ProductDTO
{
    public function __construct(
        public string $article,
        public string $brand,
        public string $name,
        public array $offers = [],
        public array $analogs = [],
    ) {}
}

readonly class OfferDTO
{
    public function __construct(
        public string $supplier,
        public float $price,
        public int $quantity,
        public int $deliveryDays,
        public ?string $warehouse = null,
    ) {}
}

readonly class PartDTO
{
    public function __construct(
        public string $article,
        public string $name,
        public ?string $brand = null,
    ) {}
}

readonly class OrderDTO
{
    public function __construct(
        public string $supplier,
        public string $externalId,
        public string $status,
        public array $items = [],
    ) {}
}
```

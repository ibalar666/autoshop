## ЭТАП 9. Сервис поиска

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\DTO\ProductDTO;
use App\Suppliers\Autoopt\AutooptClient;

class PartSearchService
{
    public function __construct(
        private AutooptClient $client,
        private CacheService $cache,
    ) {}

    public function search(string $article): array
    {
        // 1. Нормализация артикула
        $normalizedArticle = $this->normalizeArticle($article);
        
        // 2. Проверка кэша
        
        // 3. Запрос к API
        
        // 4. Маппинг в DTO
        
        // 5. Группировка по брендам
        
        // 6. Сортировка по цене
        
        // 7. Сохранение в кэш
    }

    private function normalizeArticle(string $article): string
    {
        // ...
    }

    private function findAnalogs(string $article, string $brand): array
    {
        // ...
    }

    private function groupByBrand(array $products): array
    {
        // ...
    }

    private function sortByPrice(array $offers): array
    {
        // ...
    }
}
```

## ЭТАП 7. Интеграция API поставщика Autoopt

Спецификация: https://beta.autoopt.ru/api.html#/paths/~1api~1v2~1parts~1search~1%7Barticle%7D/get

### Структура

```
app/Suppliers/
├── Interfaces/
│   └── SupplierInterface.php
└── Autoopt/
    ├── AutooptClient.php
    └── AutooptMapper.php
```

### SupplierInterface

```php
<?php

declare(strict_types=1);

namespace App\Suppliers\Interfaces;

interface SupplierInterface
{
    public function searchByArticle(string $article): array;
    public function searchByName(string $name): array;
    public function getAnalogs(string $article, string $brand): array;
    public function createOrder(array $items): array;
}
```

### AutooptClient

HTTP-клиент для API Autoopt с методами:
- searchByArticle($article)
- searchByName($name)
- getAnalogs($article, $brand)
- createOrder($items)

Настройки API хранить в config/services.php или ApiSetting модели.

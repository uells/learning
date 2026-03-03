## 1) Как “подключиться” к инфоблокам и получить IBLOCK_ID по CODE

```php
<?php
require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php'; // если это отдельный скрипт

use Bitrix\Main\Loader;
use Bitrix\Iblock\IblockTable;

Loader::includeModule('iblock'); // подключаем модуль

$iblockId = (int)(IblockTable::getList([
    'select' => ['ID'],
    'filter' => ['=CODE' => 'docs', '=IBLOCK_TYPE_ID' => 'content', '=ACTIVE' => 'Y'],
    'limit'  => 1,
])->fetch()['ID'] ?? 0);

if ($iblockId <= 0) die('IBlock not found');
```

**Пояснение (кратко):**

* В D7-подходе модуль подключаем через `Loader::includeModule`.
* `IblockTable` — нормальный “современный” способ найти инфоблок по `CODE/TYPE`.

---

## 2) Как добавить элемент в инфоблок (как обычно делают сейчас)

```php
<?php
use Bitrix\Main\Loader;

Loader::includeModule('iblock');

$el = new CIBlockElement();

$elementId = (int)$el->Add([
    'IBLOCK_ID' => $iblockId,
    'NAME'      => 'Заголовок элемента',
    'ACTIVE'    => 'Y',
]);

if ($elementId <= 0) {
    die($el->LAST_ERROR); // ошибка добавления
}
```

**Пояснение (кратко):**

* Для **добавления/обновления элементов со свойствами** в Bitrix по-прежнему **чаще всего используют `CIBlockElement`**: это самый стабильный и привычный API в продакшене.
* D7/ORM чаще применяют для выборок, фильтров, связей и “чтения”.

---

## 3) Как добавить элемент со свойством типа “Документ” (свойство Файл)

### 3.1 Файл с диска

```php
<?php
Loader::includeModule('iblock');

$docFile = CFile::MakeFileArray($_SERVER['DOCUMENT_ROOT'].'/upload/sample.pdf'); // абсолютный путь

$el = new CIBlockElement();
$elementId = (int)$el->Add([
    'IBLOCK_ID' => $iblockId,
    'NAME'      => 'Элемент с документом',
    'ACTIVE'    => 'Y',
    'PROPERTY_VALUES' => [
        'DOCUMENT' => $docFile, // CODE свойства типа "Файл"
    ],
]);

if ($elementId <= 0) die($el->LAST_ERROR);
```

### 3.2 Файл из формы (`$_FILES`)

```php
<?php
Loader::includeModule('iblock');

$el = new CIBlockElement();
$elementId = (int)$el->Add([
    'IBLOCK_ID' => $iblockId,
    'NAME'      => 'Элемент из формы',
    'ACTIVE'    => 'Y',
    'PROPERTY_VALUES' => [
        'DOCUMENT' => $_FILES['DOCUMENT'], // <input type="file" name="DOCUMENT">
    ],
]);

if ($elementId <= 0) die($el->LAST_ERROR);
```

**Пояснение (кратко):**

* В `PROPERTY_VALUES` указываешь **CODE свойства** (`DOCUMENT`).
* Для “Файл” можно передавать:

  * `CFile::MakeFileArray(path)`
  * или массив `$_FILES[...]`
* Если свойство множественное: ` 'DOCUMENT' => [$file1, $file2]`.

---

## 4) Как вывести информацию о документе (имя, размер, ссылка)

### Вариант A — “обычно так делают” (быстро и понятно)

```php
<?php
use Bitrix\Main\FileTable;

Loader::includeModule('iblock');

// 1) достаем ID файла из свойства DOCUMENT
$propRes = CIBlockElement::GetProperty($iblockId, $elementId, [], ['CODE' => 'DOCUMENT']);

$fileIds = [];
while ($p = $propRes->Fetch()) {
    if (!empty($p['VALUE'])) $fileIds[] = (int)$p['VALUE']; // для множественного соберет все
}

// 2) по каждому файлу получаем мету + ссылку
foreach ($fileIds as $fileId) {
    $file = FileTable::getById($fileId)->fetch();
    if (!$file) continue;

    echo $file['ORIGINAL_NAME']."\n";       // имя
    echo $file['FILE_SIZE']." bytes\n";     // размер
    echo CFile::GetPath($fileId)."\n\n";    // публичная ссылка
}
```

**Пояснение (кратко):**

* `GetProperty` — простой путь получить значение свойства (file_id).
* `FileTable` (D7) — современно взять метаданные файла.
* `CFile::GetPath` — получить публичный URL.

---

## 5) Как вывести документ “по-новому” (D7): получить file_id через PropertyTable + ElementPropertyTable

```php
<?php
use Bitrix\Iblock\PropertyTable;
use Bitrix\Iblock\ElementPropertyTable;
use Bitrix\Main\FileTable;

Loader::includeModule('iblock');

// 1) находим ID свойства DOCUMENT
$propId = (int)(PropertyTable::getList([
    'select' => ['ID'],
    'filter' => ['=IBLOCK_ID' => $iblockId, '=CODE' => 'DOCUMENT', '=ACTIVE' => 'Y'],
    'limit'  => 1,
])->fetch()['ID'] ?? 0);

if ($propId <= 0) die('Property DOCUMENT not found');

// 2) берем значения свойства (для множественного придет несколько строк)
$propVals = ElementPropertyTable::getList([
    'select' => ['VALUE'],
    'filter' => ['=IBLOCK_ELEMENT_ID' => $elementId, '=IBLOCK_PROPERTY_ID' => $propId],
]);

while ($row = $propVals->fetch()) {
    $fileId = (int)$row['VALUE'];
    if ($fileId <= 0) continue;

    $file = FileTable::getById($fileId)->fetch();
    if (!$file) continue;

    echo $file['ORIGINAL_NAME']."\n";
    echo $file['FILE_SIZE']." bytes\n";
    echo CFile::GetPath($fileId)."\n\n";
}
```

**Пояснение (кратко):**

* Это уже D7-цепочка: `PropertyTable` → `ElementPropertyTable` → `FileTable`.
* Удобно, когда ты строишь выборки “в ORM стиле” и не хочешь `GetProperty`.

---

Если хочешь, следующим блоком добавлю ещё 2 “шпаргалочных” пункта в таком же стиле:

* **Как обновить элемент + заменить документ** (Update + PROPERTY_VALUES)
* **Как выбрать список элементов с фильтром (D7 ElementTable) и потом подтянуть документ** (паттерн для списка/каталога)

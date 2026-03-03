# Работа с инфоблоками
## Получение списка
### Получение списка элементов из инфоблока. Стандартный (legacy) подход

`CIBlockElement` — “legacy”-класс Битрикса для работы с элементами инфоблоков:
- `GetList` - выборка
- `Add` - добавление
- `Update` - обновление
- `GetProperty` - получение свойств

Несмотря на возраст, в проектах до сих пор широко используется, особенно когда нужны свойства и “битриксовая магия” (URL, форматирование, ~-поля).

`CIBlockElement::GetList(...)` выполняет запрос к элементам инфоблока и возвращает объект результата типа CDBResult (по сути “курсор” по строкам выборки). Самих данных сразу массивом он не отдаёт — ты читаешь их по одной записи в цикле. С результатом можно работать несколькими способами:
- **`$res->GetNext()`** возвращает массив следующей записи с обработкой Битрикса. Часто:
  - добавляет “сгенерированные” поля вроде `DETAIL_PAGE_URL` (если оно формируется)
  - может возвращать пары `FIELD` и `~FIELD `(где `~FIELD `— “сырое” значение без обработки)
  - в целом удобнее для вывода на страницу/шаблон.
- **`$res->Fetch()`** возвращает **массив следующей записи** в более “сыром” виде (минимум обработок):
- **`$res->GetNextElement()`** Возвращает **объект элемента**, у которого есть:
  - `$ob->GetFields()` — поля элемента
  - `$ob->GetProperties()` — свойства (структурировано, удобно для множественных/HTML)

**Общее**
- на каждой итерации возвращают **одну строку** (один элемент),
- когда записи кончились — возвращают `false`.

**Как получить элементы инфоблока по фильтру, сразу с полями и свойствами?**
```php
<?php
$iblockId = 12;

$res = CIBlockElement::GetList(
  ['ID' => 'DESC'],                 // сортировка
  [
    'IBLOCK_ID' => $iblockId,
    '>=DATE_ACTIVE_FROM' => '29.11.2023',
    '=ACTIVE' => 'Y',
  ],
  false,                            // группировка
  ['nTopCount' => 20],              // навигация
  ['*', 'PROPERTY_*']               // поля + все свойства (плоско в массиве)
);

while ($row = $res->GetNext()) {    // GetNext() удобнее, чем Fetch()
  // ПОЛЯ
  $id   = (int)$row['ID'];
  $name = $row['NAME'];

  // СВОЙСТВА (плоские ключи)
  // Для свойства с CODE=FIO будет: PROPERTY_FIO_VALUE
  $fio = $row['PROPERTY_FIO_VALUE'] ?? null;

  // Для свойства типа "файл" (CODE=PHOTO): VALUE = file_id
  $photoId = (int)($row['PROPERTY_PHOTO_VALUE'] ?? 0);
  $photoSrc = $photoId ? CFile::GetPath($photoId) : null;

  // Для HTML/текст (USER_TYPE=HTML) часто приходит _VALUE['TEXT'] через GetProperties,
  // но в режиме PROPERTY_* обычно будет строка в PROPERTY_CODE_VALUE
  $songStory = $row['PROPERTY_SONG_STORY_VALUE'] ?? null;

  // ...
}
```
**Как получать поля + только нужные свойства (быстрее и стабильнее)?**
```php
<?php
$res = CIBlockElement::GetList(
  ['ID' => 'DESC'],
  ['IBLOCK_ID' => $iblockId, '=ACTIVE' => 'Y'],
  false,
  ['nTopCount' => 50],
  [
    'ID','NAME','DATE_CREATE',       // нужные поля
    'PROPERTY_FIO',                  // нужные свойства
    'PROPERTY_EMAIL',
    'PROPERTY_PHOTO',
    'PROPERTY_SONG_TITLE',
  ]
);

while ($row = $res->GetNext()) {
  $fio   = $row['PROPERTY_FIO_VALUE'] ?? null;
  $email = $row['PROPERTY_EMAIL_VALUE'] ?? null;

  $photoId  = (int)($row['PROPERTY_PHOTO_VALUE'] ?? 0);
  $photoSrc = $photoId ? CFile::GetPath($photoId) : null;
}
```
**Как получить элемент(ы) с полями + всеми свойствами (множественное)?**
```php
<?php
$res = CIBlockElement::GetList(
  ['ID' => 'DESC'],
  ['IBLOCK_ID' => $iblockId, '=ACTIVE' => 'Y'],
  false,
  ['nTopCount' => 20],
  ['ID','NAME','DETAIL_TEXT'] // только поля
);

while ($ob = $res->GetNextElement()) {
  $fields = $ob->GetFields();      // поля
  $props  = $ob->GetProperties();  // ВСЕ свойства (по CODE)

  $fio = $props['FIO']['VALUE'] ?? null;

  // Файл
  $photoId = (int)($props['PHOTO']['VALUE'] ?? 0);
  $photoSrc = $photoId ? CFile::GetPath($photoId) : null;

  // HTML/текст (user type HTML) — здесь уже корректная структура
  $storyText = $props['SONG_STORY']['VALUE']['TEXT'] ?? $props['SONG_STORY']['VALUE'] ?? null;
}
```

### Получение списка элементов из инфоблока. D7 подход. (через `\Bitrix\Iblock\Elements\Element*****Table`)

1) Подготовка в админке

В настройках инфоблока заполнить поле «Символьный код API» (например `statehistorys1`). Тогда Битрикс сгенерирует ORM-класс вида:

`\Bitrix\Iblock\Elements\Element<API_CODE>Table`

где <**`API_CODE`**> — твой “Символьный код API” (`StateHistoryS1`).

```php
<?php
use Bitrix\Main\Loader;

Loader::includeModule('iblock');

// Сформированный ORM-класс по "Символьному коду API"
$entityClass = \Bitrix\Iblock\Elements\ElementStateHistoryS1Table::class;

$collection = $entityClass::getList([
  'filter' => [
    '>=ACTIVE_FROM' => '29.11.2023',
  ],
  'select' => [
    'ID',
    'NAME',
    'STATE',          // свойство инфоблока (по CODE)
    'IBLOCK_SECTION', // связь на раздел
  ],
])->fetchCollection(); // <-- важно: коллекция объектов
```
**Как читать результат?**

**`fetchCollection()`** - возвращает коллекцию объектов, а не массив.

У каждого объекта есть:
- геттеры `getId()`, `getName()`, `get('ID')`
- геттер свойства по **CamelCase**: `STATE → getState()`

```php
foreach ($collection as $element) {
  $id   = $element->getId();           // или $element['ID'] / $element->get('ID')
  $name = $element->getName();         // или $element['NAME']

  // Свойство STATE: доступ через ключ или геттер
  // (структура зависит от типа свойства: одиночное/множественное, список/строка и т.п.)
  $stateValue = $element->getState()->getValue(); // типичный вариант для одиночного

  // Раздел: получаем объект раздела и его имя
  $sectionName = $element->getIblockSection()?->getName();
}
```

Помимо этого метода существует и `fetchAll()` - он возвращает массивы, и свойства/связи часто выглядят менее удобно (ключи могут быть неочевидными)




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

#Источники
1. [1С-Битрикс. Ядро d7 в работе с элементами инфоблоков](https://habr.com/ru/articles/778052/)

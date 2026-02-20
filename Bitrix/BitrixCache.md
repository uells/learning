# Кеширование в 1С-Битрикс

## 1) Что такое кеш в Битриксе и где хранится

Кеш — это сохранение результатов “дорогих” операций (SQL/ORM/HL/API/сбор массивов/HTML), чтобы повторно не выполнять вычисления.

Типовые директории:

* **`/bitrix/cache/`** — обычный файловый кеш (компонентный + low-level).
* **`/bitrix/managed_cache/`** — служебная часть “управляемого кеша” (зависимости, индексы для тегов и т.п.).
* Композит (если включён) хранит HTML страниц отдельно (в зависимости от версии/настроек).

Кеш всегда определяется тремя вещами:

* **TTL** (время жизни)
* **cache key** (ключ: уникально описывает результат)
* **инвалидация** (TTL или сброс по тегам/событиям)

---

## 2) Виды кеша

### A) Компонентный кеш (самый частый)

Используется при `IncludeComponent()`. Управляется параметрами:

* `CACHE_TYPE`: `A|Y|N`
* `CACHE_TIME`: секунды
* `CACHE_GROUPS`: `Y|N` (учёт прав групп)

**Как работает:** компонент рассчитывает ключ → если кеш валиден, отдаёт результат без тяжёлых запросов.

**Мини-пример (стандартный компонент):**

```php
$APPLICATION->IncludeComponent(
  "bitrix:news.list",
  "my_tpl",
  [
    "IBLOCK_ID" => 5,
    "NEWS_COUNT" => 10,
    "CACHE_TYPE" => "A",
    "CACHE_TIME" => 3600,
    "CACHE_GROUPS" => "Y",
  ]
);
```

**Кастомный компонент — ключевые методы (в `component.php`):**

* `$this->StartResultCache($ttl, $additionalCacheId = false)` — читать/создать кеш.
* `$this->AbortResultCache()` — отменить кеширование (ошибка/пусто).
* `$this->IncludeComponentTemplate()` — вывести шаблон.
* `$this->getCachePath()` — путь кеша компонента (для тегов).

**Мини-пример (кастомный компонент):**

```php
if ($this->StartResultCache(3600)) {
  $arResult['ITEMS'] = heavyQuery();
  if (!$arResult['ITEMS']) { $this->AbortResultCache(); return; }
  $this->IncludeComponentTemplate();
}
```

---

### B) Low-level кеш D7 (когда ты пишешь “просто PHP”)

Класс: **`Bitrix\Main\Data\Cache`**

**Ключевые методы:**

* `Cache::createInstance()`
* `initCache($ttl, $cacheId, $cacheDir)` → проверка валидного кеша
* `getVars()` → получить данные
* `startDataCache()` → начать создание кеша
* `endDataCache($data)` → записать данные
* `abortDataCache()` → отменить запись (если ошибка)

**Мини-пример:**

```php
use Bitrix\Main\Data\Cache;

$cache = Cache::createInstance();
$ttl = 600;
$cacheId  = 'dict_colors_v1';
$cacheDir = '/dict/colors';

if ($cache->initCache($ttl, $cacheId, $cacheDir)) {
  $data = $cache->getVars();
} elseif ($cache->startDataCache()) {
  $data = loadFromHL(); // тяжелая логика
  $cache->endDataCache($data);
}
```

---

### C) Тегированный кеш (TaggedCache) — “современная” инвалидция

Дает “обновлять сразу после изменения данных”, а не ждать TTL.

**Ключевые методы:**

* `Application::getInstance()->getTaggedCache()`
* `startTagCache($cacheDir)`
* `registerTag($tag)`
* `endTagCache()`
* `clearByTag($tag)` — очистить все кеши, где зарегистрирован тег

**Мини-пример (D7 Cache + tags):**

```php
use Bitrix\Main\Data\Cache;
use Bitrix\Main\Application;

$cache = Cache::createInstance();
$ttl = 86400;
$cacheId  = 'home_services_v1';
$cacheDir = '/home/services';

if ($cache->initCache($ttl, $cacheId, $cacheDir)) {
  $data = $cache->getVars();
} elseif ($cache->startDataCache()) {

  $tagged = Application::getInstance()->getTaggedCache();
  $tagged->startTagCache($cacheDir);
  $tagged->registerTag('iblock_id_5');      // зависим от ИБ 5
  $tagged->registerTag('home_services');    // кастомный тег
  $tagged->endTagCache();

  $data = loadFromIblock();
  $cache->endDataCache($data);
}
```

**Сброс по тегу (например после импорта/события):**

```php
\Bitrix\Main\Application::getInstance()
  ->getTaggedCache()
  ->clearByTag('home_services');
```

> Для инфоблоков `iblock_id_X` часто сбрасывается ядром автоматически.
> Для HL/кастомных источников обычно нужен явный `clearByTag()` через события/процедуры обновления.

---

### D) Управляемый кеш (Managed cache)

Это режим/инфраструктура, благодаря которой ядро и tagged cache умеют **точечно инвалидировать** кеш при изменениях данных.
Хранилище служебных зависимостей — обычно в **`/bitrix/managed_cache/`**.

Практически: если managed cache включён, стандартные сущности (меню/ИБ и т.п.) чаще сбрасывают кеш корректно, а твой tagged cache работает предсказуемо.

---

### E) Композит (Composite cache)

Кеширует **HTML страницы** целиком и отдаёт быстро; динамические части выводятся отдельными “областями”.
Это слой “сверху”; компонентный/low-level кеш всё равно нужен.

---

## 3) Правила мидла (самое важное)

1. **Компоненты** → `CACHE_TYPE/CACHE_TIME` + при сложных зависимостях добавляй теги/доп. ключ.
2. **Своя логика в index.php/сервисе** → `Bitrix\Main\Data\Cache` (+ теги при необходимости).
3. **Ключ кеша должен учитывать всё, что влияет на результат**: сайт/язык/фильтр/пагинацию/группы.
4. **Персональное не кешируй общим кешем** (корзина/личное) — делай AJAX/динамические области.
5. Для “обновлять сразу” → **TaggedCache + clearByTag()** (обычно через события изменения данных).

---

## 4) Мини-пример “реальной жизни” (в одном блоке)

Кешируем справочник из HL (редко меняется), обновляем сразу после изменения:

```php
use Bitrix\Main\Data\Cache;
use Bitrix\Main\Application;

$hlId = 7;
$ttl = 86400;
$cacheDir = "/dict/hl{$hlId}";
$cacheId  = "dict_v1";

$cache = Cache::createInstance();

if ($cache->initCache($ttl, $cacheId, $cacheDir)) {
  $dict = $cache->getVars();
} elseif ($cache->startDataCache()) {

  $tagged = Application::getInstance()->getTaggedCache();
  $tagged->startTagCache($cacheDir);
  $tagged->registerTag("hlblock_{$hlId}"); // свой тег зависимости
  $tagged->endTagCache();

  $dict = loadDictFromHl($hlId);
  $cache->endDataCache($dict);
}
```

Сброс (в обработчике изменения HL/после импорта):

```php
\Bitrix\Main\Application::getInstance()
  ->getTaggedCache()
  ->clearByTag('hlblock_7');
```
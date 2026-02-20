# Компонеты в 1С-Bitrix. Кастомные компоненты.
## Файл class.php 
Поддержка классов компонентов реализована в виде файла `/component_name/class.php`. Имя [`class.php`](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&LESSON_ID=2028&LESSON_PATH=3913.3516.4790.2028) - зарезервированно. Этот файл автоматически подключается при вызове:
```php
$APPLICATION->IncludeComponent()
```
При этом происходит вызов `final` метода `initComponent` в котором и подключается `class.php` (если он есть) и из него берется самый последний класс наследник от `CBitrixComponent`.

Действия вида:
```php
class CDemoTest extends CBitrixComponent{}
class CDemoTestDecorator1 extends CDemoTest {}
class CDemoTestDecorator2 extends CDemoTest {}
```
не будут иметь успеха. В итоге будет использоваться CDemoTestDecorator2.

Учтите что при изменении базового класса компонента нужно будет учитывать поведение всех его потомков (других компонентов).

### Пример

Рассмотрим простейший компонент возводящий параметр в квадрат.

Файл `/bitrix/components/demo/sqr/component.php`:
```php
<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true) die();
$arParams["X"] = intval($arParams["X"]);
if($this->startResultCache()) //startResultCache используется не для кеширования html, а для кеширования arResult
{
	$arResult["Y"] = $arParams["X"] * $arParams["X"];
}
$this->includeComponentTemplate();
?>
```
Файл `/bitrix/components/demo/sqr/templates/.default/template.php`:
```php
<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true)die();?>
<div class="equation">
<?echo $arParams["X"];?> в квадрате равно <?echo $arResult["Y"];?>
</div>
```
В реальных компонентах вместо операции умножения может быть три десятка строк и таких операций может быть 5-6. В результате файл `component.php` превращается в тяжело понимаемую "вещь в себе".
Выделяем логику компонента в класс.

Файл `/bitrix/components/demo/sqr/class.php`:
```php
<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true) die();
class CDemoSqr extends CBitrixComponent
{
	// Подготавливаем параметры компонента
	public function onPrepareComponentParams($arParams)
	{
		$result = array(
			"CACHE_TYPE" => $arParams["CACHE_TYPE"],
			"CACHE_TIME" => isset($arParams["CACHE_TIME"]) ?$arParams["CACHE_TIME"]: 36000000,
			"X" => intval($arParams["X"]),
		);
		return $result;
	}
	public function sqr($x)
	{
		return $x * $x;
	}
}?>
```

Файл `/bitrix/components/demo/sqr/component.php`:

```php
<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true) die();
if($this->startResultCache())//startResultCache используется не для кеширования html, а для кеширования arResult
{
	//$this - экземпляр CDemoSqr
	$arResult["Y"] = $this->sqr($arParams["X"]);
}
$this->includeComponentTemplate();
?>
```
Теперь код в файле `component.php` стал управляемым.


## Файл component.php
`component.php` — основной файл выполнения компонента в «классическом» стиле. В нём обычно:

- приводят и нормализуют параметры (`$arParams`);
- формируют результат (`$arResult`) — данные, которые пойдут в шаблон;
- управляют кешированием (`startResultCache()`, `abortResultCache()`, `endResultCache()`);
- подключают шаблон через `$this->includeComponentTemplate()`.

### Если в компоненте есть class.php — что тогда выполняется?
Когда у компонента есть `class.php`, ядро создаёт объект вашего класса (наследника `CBitrixComponent`) и вызывает `executeComponent()`. Если вы **не переопределяли** `executeComponent()`, то дефолтная реализация просто подключает `component.php` внутри себя. То есть `class.php` в таком режиме — это «обвязка» (методы, подготовка параметров, приватные функции), а точка входа логики остаётся `component.php`.

Если вы **переопределяете** `executeComponent()` и не вызываете `parent::executeComponent()`, тогда `component.php` можно вообще не использовать: вся логика будет внутри класса.

### Типовой каркас component.php (с кешем)
```php
<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true) die();

$arParams = $this->onPrepareComponentParams($arParams); // если используешь class.php

if ($this->startResultCache($arParams["CACHE_TIME"])) {
    // 1) тяжелая логика: запросы к БД / API / сложные вычисления
    $arResult = [
        "Y" => $arParams["X"] * $arParams["X"],
    ];

    // 2) ключи, которые нужно сохранить отдельно, чтобы они были доступны в component_epilog.php
    $this->setResultCacheKeys(["Y"]);
}

// 3) вывод шаблона (либо выполняется, либо отдается из кеша)
$this->includeComponentTemplate();
?>
```

**Важный нюанс по кешу.** Встроенный кеш компонента — это не только «кеш данных». При валидном кеше `startResultCache()` может **сразу вывести закешированный вывод** и вернуть `false`, а код внутри `if(...) {}` не выполнится. При невалидном кеше возвращается `true`, код внутри выполняется, и кеш сохраняется при подключении шаблона (`includeComponentTemplate()`).  
Отдельно есть `endResultCache()` — он позволяет «завершить кеширование данных», а вывод (шаблон) вызвать уже **вне кешируемого блока**, если тебе нужно.

### abortResultCache() — зачем
`abortResultCache()` применяется, когда ты понял, что кешировать нечего/нельзя (например, объект не найден, нет прав, ошибка входных данных). Иначе можно «наделать мусорного кеша» под произвольные значения параметров.

---

## Файл template.php
`template.php` — файл шаблона компонента (представление). Его задача — **сгенерировать HTML** на основе `$arResult` и `$arParams`.

Типовые правила, которые держат шаблон «здоровым»:

- минимум вычислений: циклы/условия ок, но «бизнес-логика» должна жить в `component.php`/`class.php` или `result_modifier.php`;
- никаких запросов к БД из шаблона (почти всегда плохая идея);
- если используется кеш компонента, то PHP-код `template.php` будет выполняться **только на промахе кеша**, а на хите вернётся готовый HTML.

### Где лежит template.php и как выбирается шаблон
Шаблон может быть:
1) системным — внутри компонента:  
`/bitrix/components/<namespace>/<component>/templates/<template_name>/template.php`  
или в `/local/components/...` для кастомных компонентов;

2) переопределённым в шаблоне сайта (на практике это основной сценарий кастомизации стандартных компонентов):  
`/bitrix/templates/<site_template>/components/<namespace>/<component>/<template_name>/template.php`  
или в `/local/templates/<site_template>/components/...` (в современных проектах чаще так).

Когда ты указываешь второй параметр в `$APPLICATION->IncludeComponent("ns:comp", "my_tpl", ...)`, ты задаёшь `<template_name>`. Если имя не указано (или пустая строка), используется `.default`.

### Что такое $this в template.php
Внутри `template.php` `$this` — это объект шаблона (`CBitrixComponentTemplate`), а не объект компонента. Через него можно, например:

- включить поддержку композитного режима для шаблона: `$this->setFrameMode(true);`
- подключить внешние CSS/JS: `$this->addExternalCss(...)`, `$this->addExternalJs(...)`.

---

## Файл result_modifier.php
`result_modifier.php` — **промежуточный файл** шаблона, который подключается **перед** `template.php`, если он существует в папке шаблона.

Задача: подготовить `$arResult` к выводу так, чтобы `template.php` оставался максимально простым.

### Когда использовать
- нужно «пересобрать» структуру `$arResult` под удобный вывод (группировки, индексы, приведение форматов);
- нужно добавить «косметические» вычисления (форматирование дат/цен, подготовка ссылок, разбор сложных структур);
- нужно доточить стандартный компонент, **не правя** его `component.php` (важно для обновлений).

### Кеширование result_modifier.php
Если компонент работает с встроенным кешем, то `result_modifier.php` обычно исполняется в той же логике кеширования, что и `template.php`: при хите кеша он не выполняется (потому что шаблон не выполняется), а отдаётся готовый HTML.

### Передача данных в component_epilog.php (SetResultCacheKeys)
Если в `component_epilog.php` тебе нужен какой-то кусок `$arResult`, то этот ключ нужно явно сохранить в кеш:  
```php
<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true)die();

$cp = $this->__component;          // объект компонента
$cp->SetResultCacheKeys(["Y"]);    // "Y" попадёт в кеш и будет доступен в component_epilog.php

$arResult["Y_FORMATTED"] = number_format($arResult["Y"], 0, ".", " ");
```

---

## Файл component_epilog.php
`component_epilog.php` — файл, который подключается **после** исполнения шаблона компонента и выполняется **на каждом хите**, независимо от того, есть кеш или нет.

Типовые применения:

- действия, которые должны выполняться всегда: `SetTitle`, `SetPageProperty`, хлебные крошки (`AddChainItem`), метрики;
- «некешируемые вставки» внутри кешируемого компонента (например, вложенный компонент/блок, который должен обновляться на каждом запросе);
- пост-обработка вывода/буферов (реже, но встречается).

### Важный нюанс: component_epilog.php выполняется *до* кода, который идёт после IncludeComponentTemplate()
То есть если в `component.php` у тебя после `$this->IncludeComponentTemplate()` стоит код, он выполнится **после** `component_epilog.php`. Если ты, например, выставляешь заголовок страницы и там, и там — «победит» последний.

### Мини-пример для demo:sqr
Файл `/bitrix/components/demo/sqr/templates/.default/component_epilog.php`:
```php
<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true)die();
global $APPLICATION;

// На хите кеша $arResult может содержать только те ключи, которые сохранены через SetResultCacheKeys()
$APPLICATION->SetTitle("Квадрат: ".$arResult["Y"]);
```

---

## Пролог и эпилог страницы
В терминах Битрикса «страница» (обычно это `/something/index.php`) состоит из:

- **пролога**: подключение ядра, модулей, событий, начало буферизации;
- **рабочей области**: основной код страницы (в том числе вызовы компонентов);
- **эпилога**: вывод накопленного, финальные события, завершение буферизации.

На практике это выглядит так:
```php
require($_SERVER["DOCUMENT_ROOT"]."/bitrix/header.php"); // пролог (визуальная часть)
    // workarea: здесь твой код и компоненты
require($_SERVER["DOCUMENT_ROOT"]."/bitrix/footer.php"); // эпилог (визуальная часть)
```

Внутри пролога и эпилога ядро также подключает «служебные» части (`prolog_before.php`, `prolog_after.php`, `epilog_before.php`, `epilog_after.php`), где происходят события уровня ядра (например, `OnProlog`/`OnEpilog`), подключение шаблона сайта и т.д.

---

## Как это работает в системе
Разберём цепочку исполнения на примере вызова компонента `demo:sqr` на странице.

### 1) Страница
```php
<?require($_SERVER["DOCUMENT_ROOT"]."/bitrix/header.php");

$APPLICATION->IncludeComponent(
    "demo:sqr",
    ".default",
    [
        "X" => 7,
        "CACHE_TYPE" => "A",
        "CACHE_TIME" => 3600,
    ]
);

require($_SERVER["DOCUMENT_ROOT"]."/bitrix/footer.php");?>
```

### 2) Что происходит при IncludeComponent()
Упрощённо (без внутренних деталей редактора/панели/композита) процесс такой:

1. Ядро ищет компонент по имени `demo:sqr` (в первую очередь в `/local/components`, затем в `/bitrix/components`).
2. Инициализирует компонент (`initComponent()`), подключает `class.php` если есть, создаёт объект компонента.
3. Готовит параметры: вызывается `onPrepareComponentParams()` (если компонент классовый и метод определён).
4. Запускает выполнение:
   - если ты **переопределил** `executeComponent()` в `class.php` — выполнится он;
   - иначе дефолтный `executeComponent()` подключит `component.php`.
5. В `component.php` обычно вызывается `startResultCache()`:
   - **кеш валиден** → возвращается `false`, формирование `$arResult` пропускается, ядро отдаёт кешированный вывод;
   - **кеша нет/протух** → возвращается `true`, формируется `$arResult`, при необходимости вызывается `setResultCacheKeys()`.
6. Вызывается `$this->includeComponentTemplate()`:
   - выбирается папка шаблона (по имени шаблона и правилам переопределения);
   - если есть `result_modifier.php` — он подключается и может изменить `$arResult`;
   - подключается `template.php` и формирует HTML (или отдаётся из кеша).
7. После шаблона подключается `component_epilog.php` (если он есть) — **всегда**, даже при хите кеша.
8. Управление возвращается в код страницы, затем выполняется эпилог страницы (`footer.php`).


### 3) Мини-пример: добавляем result_modifier + component_epilog к demo:sqr
1) `/bitrix/components/demo/sqr/templates/.default/result_modifier.php`  
```php
<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true)die();

$arResult["Y_FORMATTED"] = number_format($arResult["Y"], 0, ".", " ");

// Делаем Y доступным в component_epilog.php даже на хите кеша:
$this->__component->SetResultCacheKeys(["Y"]);
```

2) `/bitrix/components/demo/sqr/templates/.default/template.php`  
(ничего не меняем, но можно вывести `$arResult["Y_FORMATTED"]` вместо `Y`, если ты решишь расширять шаблон дальше)

3) `/bitrix/components/demo/sqr/templates/.default/component_epilog.php`  
```php
<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true)die();
global $APPLICATION;

$APPLICATION->SetTitle("Квадрат числа: ".$arResult["Y"]);
```

---

## Про комплексные компоненты (самая простая модель)
Комплексный компонент — это «роутер»: он не столько собирает данные, сколько выбирает, **какую страницу шаблона** подключить (например, `list.php` или `detail.php`) в зависимости от URL и настроек SEF.

У таких компонентов шаблон обычно состоит не из одного `template.php`, а из набора файлов-страниц:
- `templates/.default/sections.php`
- `templates/.default/detail.php`
- и т.д.

А в коде компонента выбирается `$templatePage`, который передаётся в:
```php
$this->IncludeComponentTemplate($templatePage);
```

Внутри страниц комплексного компонента часто подключаются **простые** компоненты (дочерние), и тогда появляется важная связка: родительский компонент + `$component` (ссылка на родителя) для корректной работы кеша/эпилогов вложенных компонентов.



## Справка
`final` в PHP — это модификатор, который запрещает переопределение. Если метод объявлен как `final`, то в дочернем классе нельзя написать метод с таким же именем, чтобы заменить его поведение.

```php
class A {
    final public function initComponent() {}
}

class B extends A {
    public function initComponent() {} // ❌ Fatal error: Cannot override final method
}
```
Если класс `final`, то от него нельзя наследоваться.
```php
final class A {}
class B extends A {} // ❌ нельзя
```

[`CBitrixComponent`](https://dev.1c-bitrix.ru/api_help/main/reference/cbitrixcomponent/index.php) — это базовый (родительский) класс ядра Битрикса для всех компонентов.

`$arParams` — массив входных параметров компонента. Формируется из параметров вызова `$APPLICATION->IncludeComponent(...)` и нормализуется в `onPrepareComponentParams()` (если он есть).

`$arResult` — массив результатов работы компонента. Его наполняют в `component.php` / `executeComponent()` и (опционально) дорабатывают в `result_modifier.php`. Именно этот массив использует шаблон для вывода.

`startResultCache()` — метод встроенного кеширования компонента. Если кеш валиден, метод может вывести сохранённый вывод и вернуть `false`. Если кеш невалиден — возвращает `true`, и код внутри блока формирует данные/вывод, которые будут сохранены.

`endResultCache()` — завершает кеширование данных, позволяя вынести вызов `includeComponentTemplate()` **за пределы** кешируемого блока (когда нужно кешировать только данные, но не кешировать HTML).

`abortResultCache()` — прерывает кеширование (например, при отсутствии данных/ошибке/нет прав), чтобы не сохранить «плохой» кеш.

`SetResultCacheKeys([...])` — задаёт список ключей `$arResult`, которые нужно сохранить в кеше так, чтобы они были доступны на хите кеша в `component_epilog.php` (и иногда — родительскому компоненту).

`includeComponentTemplate($templatePage = "")` — подключает шаблон компонента. Для обычного компонента `$templatePage` пустой. Для комплексного компонента туда передают имя «страницы шаблона» (например, `detail`).

`$this`:
- в `component.php` (и в методах компонента) — это объект компонента (`CBitrixComponent` или твой наследник);
- в `template.php`/`result_modifier.php` — это объект шаблона (`CBitrixComponentTemplate`), а объект компонента доступен как `$this->__component`.

`$component` — ссылка на объект родительского компонента. Её часто передают 4-м параметром в `IncludeComponent(...)` при вызове вложенных компонентов, чтобы родитель мог корректно учитывать кеш/эпилоги дочерних компонентов.

Комплексный компонент — компонент, который выбирает, какую «страницу шаблона» подключить, обычно в зависимости от URL. Как правило, он выступает «роутером» и подключает простые компоненты для конкретных страниц.

SEF (Search Engine Friendly URLs) — режим формирования человеко-понятных URL (например, `/catalog/iphone-15/` вместо `?ID=123`). В комплексных компонентах SEF влияет на то, какая страница шаблона будет выбрана.



# Конспект: AddViewContent / ShowViewContent / ob_* в Битрикс

## 1) [CMain::AddViewContent](https://dev.1c-bitrix.ru/api_help/main/reference/cmain/addviewcontent.php)

**Определение:** метод **складывает** (накапливает) HTML/текст в именованный “слот” (область) страницы, чтобы потом вывести его в нужном месте шаблона.

### Параметры
- **$view** *(string)* — имя слота (например: `afterTitle`, `head`, `sidebar`).
- **$content** *(string)* — добавляемый контент (обычно HTML).
- **$pos** *(int, опционально)* — **порядок внутри слота**: чем меньше число, тем раньше окажется этот кусок при выводе.  
  Полезно, когда в один слот добавляют несколько компонентов/фрагментов, а выводить нужно в стабильном порядке.

### Короткий пример
```php
global $APPLICATION;

$APPLICATION->AddViewContent('afterTitle', '<div class="badge">Акция</div>', 100);
$APPLICATION->AddViewContent('afterTitle', '<div class="note">Важно</div>', 10); // выведется раньше
```

## 2) [CMain::ShowViewContent](https://dev.1c-bitrix.ru/api_help/main/reference/cmain/showviewcontent.php)

**Определение:** метод **выводит** всё, что было **сложено** в указанный слот через `AddViewContent` (с учётом `$pos`).

### Параметры

* **$view** *(string)* — имя слота, который нужно вывести (то же самое имя, что в `AddViewContent`).
* *(встречаются доп. параметры в разных примерах/версиях, но для базового сценария обычно достаточно имени слота)*

### Короткий пример

```php
global $APPLICATION;

$APPLICATION->ShowViewContent('afterTitle');
```

## 3) PHP output buffering (ob_*)

> Буфер вывода — это “копилка” для `echo`/HTML: вместо мгновенного вывода на страницу контент можно собрать в строку и дальше куда-то положить (например, в `AddViewContent`).

### ob_start

**Что делает:** включает буферизацию вывода (всё, что “печатается”, копится в буфере).

```php
ob_start();
echo "<b>Привет</b>";
```

### ob_get_clean

**Что делает:** забирает содержимое буфера **в строку** и **очищает/закрывает** буфер.

```php
ob_start();
echo "<div>Собрали HTML</div>";
$html = ob_get_clean(); // $html содержит строку, на страницу ничего не вывелось
```

### ob_get_contents

**Что делает:** возвращает текущий буфер **в строку**, **не очищая** и не закрывая его.

```php
ob_start();
echo "A";
$tmp = ob_get_contents(); // "A"
echo "B";
$all = ob_get_contents(); // "AB"
ob_end_clean();
```

### ob_end_clean

**Что делает:** очищает буфер и выключает буферизацию, **ничего не возвращает**.

```php
ob_start();
echo "<div>Это выбросим</div>";
ob_end_clean(); // контент не попадёт на страницу
```

## 4) Применение (связка ob_* + AddViewContent + ShowViewContent) — слот `afterTitle`

### Шаблон (где выводим)

```php
<?php global $APPLICATION; ?>

<h1><?= $APPLICATION->ShowTitle(false) ?></h1>

<?php
// Здесь выведется всё, что компоненты/логика сложили в afterTitle
$APPLICATION->ShowViewContent('afterTitle');
?>
```

### Компонент / include (где добавляем)

```php
<?php
global $APPLICATION;

ob_start();

echo '<div class="after-title">';
echo '  <span class="tag">Новинка</span>';
echo '  <a href="/promo/">Смотреть промо</a>';
echo '</div>';

$html = ob_get_clean();

// Складываем в слот afterTitle (pos управляет порядком среди других добавлений)
$APPLICATION->AddViewContent('afterTitle', $html, 50);
```

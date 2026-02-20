# Добавление CSS и JS методами 1С-Битрикс
В 1С-Битрикс “по-современному” (и правильно с точки зрения кеширования/объединения/Композита) CSS/JS подключают **через менеджер ассетов**, а не руками через `<link>`/`<script>` в шаблоне.

## 1) Базовый и самый распространённый способ: `Bitrix\Main\Page\Asset`

Обычно кладёте файлы в шаблон, например:

* `/bitrix/templates/<template>/assets/css/app.min.css`
* `/bitrix/templates/<template>/assets/js/app.min.js`

И подключаете в `header.php` (или `footer.php`, если скрипты внизу):

```php
<?php
use Bitrix\Main\Page\Asset;

Asset::getInstance()->addCss(SITE_TEMPLATE_PATH . '/assets/css/app.min.css');
Asset::getInstance()->addJs(SITE_TEMPLATE_PATH . '/assets/js/app.min.js');
?>
```

Важно:

* этот код должен выполниться **до** `$APPLICATION->ShowHead()` (для CSS/скриптов, которые должны попасть в `<head>`).
* Если скрипт добавили после `ShowHead()`, он чаще уйдёт в конец (зависит от шаблона/версии/настроек).

### Если нужно `defer/async`

У `Asset` нет универсального “кросс-версийного” API для `defer/async`, поэтому надёжный способ — добавить строкой:

```php
use Bitrix\Main\Page\Asset;

Asset::getInstance()->addString(
    '<script defer src="'.SITE_TEMPLATE_PATH.'/assets/js/app.min.js"></script>'
);
```

(Да, это “ручной” тег, но он всё равно идёт через менеджер ассетов и будет выводиться в нужном месте.)

---

## 2) Старые методы (встречаются, но лучше постепенно уходить)

Работают, но менее гибко, чем `Asset`:

```php
$APPLICATION->SetAdditionalCSS(SITE_TEMPLATE_PATH.'/assets/css/app.min.css');
$APPLICATION->AddHeadScript(SITE_TEMPLATE_PATH.'/assets/js/app.min.js');
```

---

## 3) Если подключение нужно **только внутри компонента**

Чтобы не тащить скрипты на весь сайт, а подключать только там, где компонент используется:

В `component.php` или `class.php` компонента:

```php
$this->addExternalCss(SITE_TEMPLATE_PATH . '/assets/css/component.css');
$this->addExternalJs(SITE_TEMPLATE_PATH . '/assets/js/component.js');
```

(Методы доступны у компонента; это хороший способ держать зависимости рядом с компонентом.)

---

## 4) “Современный” Bitrix-подход для JS: **Extensions** (`\Bitrix\Main\UI\Extension`)

Это удобно, когда вы хотите:

* описывать зависимости (core, ui, другие расширения),
* грузить пакетом,
* подключать по месту.

Пример подключения:

```php
\Bitrix\Main\UI\Extension::load('main.core');
\Bitrix\Main\UI\Extension::load('ui.alerts'); // пример
// или ваше расширение:
\Bitrix\Main\UI\Extension::load('vendor.project');
```

А своё расширение регистрируют через `config.php` в `/bitrix/js/...` (или через модуль). Это уже чуть “архитектурнее”, но на больших проектах сильно удобнее.

---

## Рекомендованная практика “в реальном проекте”

* **Глобальные стили/скрипты шаблона** → `Asset::addCss/addJs` в `header.php`.
* **Страничные/компонентные зависимости** → `addExternalCss/addExternalJs` внутри компонента или `Extension::load`.
* `defer` для тяжёлых скриптов → через `Asset::addString('<script defer …>')`.
* Не плодить подключения прямо HTML-тегами в шаблоне без нужды — потом сложно контролировать порядок и кеширование.

Если скажешь, у тебя сейчас шаблон на **старом ядре** (с `CJSCore::Init(['jquery'])`) или уже используете `main.core/ui`, и куда ты хочешь подключать (в шаблон, в компонент, только на главной) — подскажу самый чистый вариант под твой кейс.

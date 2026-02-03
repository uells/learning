# ORM подход в 1С-Bitrix
## Как получить список пользователей?
[**UserTable**](https://dev.1c-bitrix.ru/api_d7/bitrix/main/usertable/index.php) - класс для работы с пользователями.
Является наследником класса [`Bitrix\Main\ORM\Data\DataManager`](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&CHAPTER_ID=05748&LESSON_PATH=3913.5062.5748) (до версии 18.0.2 модуля Main - класса [`Bitrix\Main\Entity\DataManager`](https://dev.1c-bitrix.ru/api_d7/bitrix/main/entity/datamanager/index.php)).

В методы класса входит: 
  - [`getMap`](https://dev.1c-bitrix.ru/api_d7/bitrix/main/usertable/getmap.php) - метод возвращает список полей для таблицы.
  - унаследованные от DataManager [классы](https://dev.1c-bitrix.ru/api_d7/bitrix/main/entity/datamanager/index.php)

### Пример получения всех пользователей
```php
use Bitrix\Main\UserTable;

$users = UserTable::getList([
    'select' => ['ID', 'LOGIN', 'NAME', 'LAST_NAME', 'EMAIL'],
    'order'  => ['ID' => 'ASC'],
])->fetchAll();

foreach ($users as $user) {
    echo $user['ID'] . ' ' . $user['LOGIN'] . PHP_EOL;
}
```
### Только активные + фильтр по email домену + лимит
```php
use Bitrix\Main\UserTable;

$result = UserTable::getList(array(
    'select' => array('ID', 'LOGIN', 'EMAIL', 'ACTIVE'),
    'filter' => array(
        '=ACTIVE' => 'Y',
        '%EMAIL'  => '@example.com', // LIKE %...%
    ),
    'order' => array('ID' => 'DESC'),
    'limit' => 100,
));

$users = array();
while ($row = $result->fetch()) {
    $users[] = $row;
}
```
### Таблица сравнений подходов от GPT
| Критерий                                       | ORM (`\Bitrix\Main\UserTable`)                           | `CUser`                                                          |
| ---------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------- |
| Подход                                         | D7 ORM, единый стиль с остальными `*Table`               | Классический API Битрикса (легаси-стиль)                         |
| Чтение данных                                  | ✅ Очень удобно (select/filter/order/limit/offset)        | ✅ Удобно, но стиль отличается от ORM                             |
| JOIN’ы / сложные выборки                       | ✅ Лучше ложится на сложные запросы и                     | Обычно сложнее и менее единообразно
| интеграцию с другими таблицами (в т.ч. D7 ORM) | ⚠️ Обычно сложнее и менее единообразно                    | Часто проще/привычнее через SELECT => array("UF_*")
| Пользовательские поля `UF_*`                   | ✅ Можно, но надо явно указывать `UF_*`                   | ✅ Часто проще/привычнее через `SELECT => array("UF_*")`          |
| Легаси-код / компоненты                        | ⚠️ Иногда нужно “подружить” со старым кодом               | ✅ Максимальная совместимость, много примеров в сети              |
| Создание/обновление пользователя               | ✅ Возможно (ORM)                                         | ✅ Привычно, много готовых паттернов/обработчиков                 |
| События/хуки вокруг пользователей              | ✅ Есть, но в легаси чаще ожидают `CUser`                 | ✅ Обычно “как принято” в старых проектах                         |
| Читаемость в новой кодовой базе                | ✅ Обычно выше (единый стиль ORM)                         | ⚠️ Зависит от проекта, часто “разнобой”                           |
| Когда выбирать                                 | Новая разработка, выборки, права/профили/HL, архитектура | Быстрая правка в легаси, привычные сценарии, много готового кода |

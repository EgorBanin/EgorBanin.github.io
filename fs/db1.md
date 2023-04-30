# Работа с базой данных

Для работы с базами данных PHP предлагает несколько стандартных библиотек,
наиболее популярная из которых [PDO](https://www.php.net/PDO) (надеюсь вы с ней знакомы).
А наиболее популярной базой данных среди новичков веб-программирования является MySQL.
В этой статье покажу, как можно работать с MySQL с помощью PDO.

> Если вы познакомились с работой с базами данных в PHP, изучая программирование
> по каким-нибудь сильно устаревшим учебникам и статьям в интеренте, то возможно
> имели дело с библиотекой [mysql](https://www.php.net/manual/ru/book.mysql.php).
> Обратите внимание, что это расширение [устарело](https://www.php.net/manual/ru/intro.mysql.php)!

Из-за плохого понимания принципов взаимодействия приложения и базы данных,
начинающие программисты часто пишут довольно уродливый и уязвимый код. Ситуация усугубляется
не самым интуитивным и удобным интерфейсом PDO. Попробуем разобраться, как же использовать
PDO правильно.

## Подключение к базе данных

Чтобы начать делать запросы в базу данных, к ней надо подключиться. Это как позвонить
по телефону. Вы набираете номер друга, и только когда он поднимет трубку вы можете начать общаться.
Очевидно, что нет смысла класть трубку, после каждой фразы и перезванивать ему снова, чтобы
продолжить разговор. С базой данных точно так же. В большинстве случаев вам надо подключиться
к ней один раз за всё время взаимодействия (выполнения вашего скрипта).

Кроме этого, в вашем приложении могут быть сценарии, когда взаимодействие с базой данных не требуется.
Подключаться на всякий случай не очень-то умно. Было бы здорово подключаться к базе при
первом обращении к ней, а при последующих использовать уже созданное подключение.
Стандартная библиотека PDO ничего такого не предлагает. Этот код надо написать самому.

```php
<?php

class MySQL {
    
    private $pdoConstructorArgs;

    private $pdo;

    public function __construct(
        string $username,
        string $password,
        string $dbName,
        string $host = 'localhost',
        int $port = 3306,
        array $options = []
    ) {
        $dsn = sprintf('mysql:host=%s;port=%d;dbname=%s', $host, $port, $dbName);
        $this->pdoConstructorArgs = [
            $dsn,
            $username,
            $password,
            $options,
        ];
    }

    public function getPdo(): \PDO {
        if ($this->pdo === null) {
            // подключаемся только один раз
            $this->pdo = new \PDO(...$this->pdoConstructorArgs);
            $this->pdo->setAttribute(\PDO::ATTR_ERRMODE, \PDO::ERRMODE_EXCEPTION);
        }
    
        return $this->pdo;
    }

}

```

Теперь если создать экземпляр нашего класса, то подключение к базе данных не будет создано
до тех пор, пока мы не вызовем метод `getPdo`. При этом повторный вызов `getPdo` вернёт уже
существующее подключение.

Вы могли заметить, что PDO предлагает общую абстракцию для работы с разными базами данных
(не только MySQL). Но на самом деле, работа с разными базами довольно сильно отличается.
Разные базы данных поддерживают разный синтаксис запросов, разные типы данных и разные
возможности. Тут я рассказываю именно про MySQL, поэтому назвал класс `MySQL`.

## Доступ к подключению

Чтобы воспользоваться преимуществами, которые даёт нам наша обёртка PDO,
нам понадобится каким-то образом дать доступ к объекту класса `MySQL` во всех местах,
где мы собираемся с базой работать (таких мест может быть довольно много).
Самый простой способ сделать это -- создать глобальный (доступный отовсюду) объект.

```php
<?php

$GLOBALS['db'] = new MySQL('user', 'password', 'testdb');
```

Теперь в любом месте, где вам необходимо поработать с базой данных,
вы можете обратиться к глобальному объекту.

```php
<?php

$st = $GLOBALS['db']->getPdo()->query('select 1 as `test`');
foreach ($st as $row) {
    echo $row['test'], "\n";
}
```

Использование суперглобального массива `$GLOBALS` довольно спорное решение. Многие
более опытные программисты могут осудить этот подход. Однако, если пользоваться им аккуратно,
то на первых порах он отлично подходит для того, чтобы сделать наш объект доступным глобально.

> В некоторых учебниках по PHP вы можете встретить предложение реализовать глобальный объект
> с помощью паттерна singleton. Обратите внимание, что такой подход мало чем отличается
> от использования $GLOBALS. В любом случае предпочитайте передавать объект
> для работы с базой как аргумент (конструктора или конкретного метода),
> а не полагаться на глобальный объект.


## Выполнение запросов

Если вы потрудитесь разобраться с интерфейсом библиотеки PDO, то обнаружите
множество способов сделать запрос в базу. Это довольно печально, потому что
приводит к путанице с тем, как правильно поступить в той или иной ситуации.

### SQL-инъекции

Если ваши запросы в базу, формируются динамически из данных, которые отправляют
пользователи (а именно так работает большинство веб-приложений), то очень важно
понимать, что такое [SQL-инъекции](https://ru.wikipedia.org/wiki/Внедрение_SQL-кода)
и как их избежать.

Вот типичный запрос новичка:

```php
<?php

// ...

$st = $pdo->query("
    select *
    from `test`
    where `id` = {$_GET['id']}
");
```

Проблема заключаеся в том, что в `$_GET['id']` необязательно будет идентификатор. Данные
`$_GET` (`$_POST`, `$_REQUEST`, `$_COOKIE` и другие) это данные, которые отправил
пользователь. А раз так, то там может быть почти всё что угодно. В том числе и что-нибудь такое:

```
http://mysite/index.php?id= 1 or 1 = 1 union select * from `users` 
```

Что произойдёт, если подставить такой id в sql-запрос? Если вы уже немного разбираетесь в 
SQL, то поймёте -- запрос изменится. Теперь условием запроса будет `id = 1 or 1 = 1`
(то есть условие будет выполнено всегда). Кроме того, к результатам запроса присоединится
запрос, который добавил злоумышленник!

С помощью такой техники взломщик может логиниться на сайт под любым пользователем, читать и
изменять любые данные в базе, подсовывать пользователям вашего сайта вредоносный код.

Чтобы избежать всех этих ужасов надо разделить запрос и его параметры. В PDO для этого есть
методы `PDO::prepare` и `PDOStatement::bindValue`.

```php
<?php

// ...

$st = $pdo->prepare('
    select *
    from `test`
    where `id` = :id
');
$st->bindValue(':id', $_GET['id'], \PDO::PARAM_INT);
$st->execute();

```

### Универсальный способ сделать безопасный запрос в базу

Таким образом, в большинстве случаев запрос делается так

```php
<?php

$id = (int) ($_GET['id']?? 0);
// составляем запрос
$sql = 'select * from `foobar` where id = :id';
// подготавливаем запрос, получаем объект PDOStatement
$st = $GLOBALS['db']->getPdo()->prepare($sql);
// подставляем все переменные в запрос, указывая правильный тип
$st->bindValue(':id', $id, \PDO::PARAM_INT);
// выполняем запрос
$st->execute();
// получаем строки
$rows = $st->fetchAll();
```

Всё это надо писать всякий раз, когда вы делаете запрос в базу. Многословно и довольно утомительно.
Можно написать функцию, которая выразит ваши намерения более явно.

```php
<?php

class MySQL {
    
    private const TYPES_MAP = [
        'string' => \PDO::PARAM_STR,
        'integer' => \PDO::PARAM_INT,
        'boolean' => \PDO::PARAM_BOOL,
        'NULL' => \PDO::PARAM_NULL,
    ];

    // ...
    
    public function query(string $sql, array $params = []): \PDOStatement {
        $pdo = $this->getPdo();
        $st = $pdo->prepare($sql);
        foreach ($params as $name => $value) {
            if (isset(self::TYPES_MAP[gettype($value)])) {
                $type = self::TYPES_MAP[gettype($value)];
            } else {
                $value = (string) $value;
                $type = \PDO::PARAM_STR;
            }

            $st->bindValue($name, $value, $type);
        }
        $st->execute();

        return $st;
    }

}

```


Теперь запросы будут выглядеть как-то так:

```php
<?php

$id = (int) ($_GET['id']?? 0);
$st = $GLOBALS['db']->query('
    select *
    from `foobar`
    where id = :id
', [':id' => $id]);
// получаем строки
$rows = $st->fetchAll();
```

Ничего лишнего! Ну почти.

Для запросов `update`, `insert`, `delete` очень полезно получать количество затронутых строк,
а для `insert` ещё и последний вставленный автоинкрементарный id. Количество затронутых
строк легко получить `$st->rowCount()`. А вот последний вставленный id недоступен из
`PDOStatement`, для его получения надо вызвать метод `PDO::lastInsertId`.

На мой взгляд, было бы нагляднее и полезнее завсети класс для объекта результата выполнения
запроса, котрый содержит всё необходимое.

```php
<?php

class MySQLResult {

    public string $sql;
    
    public array $params;
    
    public array $rows;
    
    public string $affectedRows;
    
    public ?string $insertedId;
    
    public function __construct(
        string $sql,
        array  $params,
        array $rows,
        int $affectedRows,
        ?string $insertedId
    ) {
        $this->sql = $sql;
        $this->params = $params;
        $this->rows = $rows;
        $this->affectedRows = $affectedRows;
        $this->insertedId = $insertedId;
    }

}
```

Я добавил `sql` и `params`, они пригодятся для отладки, когда запрос вернул не то,
что вы ожидали.

```php
<?php

class MySQL {
    
    // ...
    
    public function query(string $sql, array $params): MySQLResult {
        // ...
        
        try {
            $rows = $st->fetchAll(\PDO::FETCH_ASSOC);
        } catch (\PDOException $e) {
            /*
                К сожалению поведение fetchAll() зависит от того, какой запрос был вызван до этого.
                Для не select запросов функция выбросит исключение.

                PDO не предоставляет возможности различать типы запросов,
                но предполагает разные методы для них -- пример плохого дизайна PDO.
            */
            $rows = [];
        }

        return new MySQLResult(
            $sql,
            $params,
            $rows,
            $st->rowCount(),
            $pdo->lastInsertId()
        );
    }
}

```

## Обработка ошибок

Если вы внимательно читали, то заметили строку
`$this->pdo->setAttribute(\PDO::ATTR_ERRMODE, \PDO::ERRMODE_EXCEPTION);`.
Она означает, что в случае ошибки (неправильно составлен запрос или разоравлось
соединение с базой, или ещё что-то пошо не так) PDO выбросит исключение `\PDOException`.
Это очень удобно, но лучше обработать такое исключение внутри класса-обёртки и выбросить своё.

```php
class MysqlException extends \Exception {
    
    const CODE_CONNECTION_ERROR = 1;
    const CODE_QUERY_ERROR = 2;
    const CODE_BINDING_ERROR = 3;
    
    public string $sql;

    public array $params;
    
    public function __construct(
        string $message,
        int $code,
        \Throwable $previous = null,
        string $sql = '',
        array $params = []
    ) {
        $this->sql = $sql;
        $this->params = $params;
        parent::__construct($message, $code, $previous);
    }
    
}
```

Тогда подключение выглядит таким образом:

```php
try {
    $this->pdo = new \PDO(...$this->pdoConstructorArgs);
} catch (\PDOException $e) {
    throw new MysqlException(
        'Не удалось подключить к базе данных. ' . $e->getMessage(),
        MysqlException::CODE_CONNECTION_ERROR,
        $e
    );
}
```

А приведение к строке переданного параметра так:

```php
try {
    $value = (string) $value;
} catch (\Throwable $t) {
    throw new MysqlException(
        "Недопустимый параметр $name. " . $t->getMessage(),
        MysqlException::CODE_BINDING_ERROR,
        $t
    );
}
```

Исключение запроса может включать в себя SQL, вызвавший проблему.

```php
try {
    $st->execute();
} catch (\PDOException $e) {
    throw new MysqlException(
        "Не удалось выполнить зарос. " . $e->getMessage(),
        MysqlException::CODE_QUERY_ERROR,
        $e,
        $sql,
        $params
    );
}
```

## Итог

В результате у нас получился заточенный на MySQL класс, с единственным методом выполнения запроса. И хотя интерфейс обёртки не такой гибкий, как PDO, зато он гораздо проще. А простота способствует большей выразительности кода и единому стилю.

```php
$pdo = new \PDO('mysql:host=localhost;port=3306;dbname=sakila', 'user', 'password');
$this->pdo->setAttribute(\PDO::ATTR_ERRMODE, \PDO::ERRMODE_EXCEPTION);
$st = $pdo->prepare('
    select *
    from `posts`
    where
        `userId` = :userId
        and `type` = :type
        and `createdAt` >= :today
');
$st->bindValue(':userId', 123, \PDO::PARAM_INT);
$st->bindValue(':type', 'article', \PDO::PARAM_STR);
$st->bindValue(':today', new \DateTime('today')->format(\DateTime::ATOM), \PDO::PARAM_STR);
$st->execute();
$rows = $st->fetchAll(\PDO::FETCH_ASSOC);
var_dump($rows);
```

```php
$db = new MySQL('user', 'password', 'sakila');
$result = $db->query('
    select *
    from `posts`
    where
        `userId` = :userId
        and `type` = :type
        and `createdAt` >= :today
', [
    ':userId' => 123,
    ':type' => 'article',
    ':today' => new \DateTime('today')->format(\DateTime::ATOM),
]);
var_dump($result->rows);
```

Кроме запроса в базу было бы полезно иметь кое-что ещё. Например, интерфейс управления транзакциями.
Об этом в [следующей части](/fs/db2.md).
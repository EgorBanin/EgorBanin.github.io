! Черновик

# Работа с базой данных

Для работы с базами данных PHP предлагает несколько стандартных библиотек,
наиболее популярная из которых [PDO](https://www.php.net/PDO) (надеюсь вы с ней знакомы).
А наиболее популярной базой данных среди новичков веб-программирования является MySQL.

Из-за плохого понимания принципов взаимодействия приложения и базы данных,
начинающие программисты часто пишут довольно уродливый и уязвимый код. Ситуация усугубляется
не самым интуитивным и удобным интерфейсом PDO. Попробуем разобраться, как использовать
PDO для работы с MySQL.

## Подключение к базе данных

Чтобы начать делать запросы в базу данных, к ней надо подключиться. Это как позвонить
по телефону. Вы набираете номер друга, и только когда он поднимет трубку вы можете начать общаться.
Очевидно, что нет смысла класть трубку, после каждой фразы и перезванивать ему снова, чтобы
продолжить разговор. С базой данных точно так же. В большинстве случаев вам надо подключимться к ней один раз
за всё время взаимодейтствия (выполнения вашего скрипта).

Кроме этого в вашем приложении могут быть сценарии, когда взаимодействие с базой данных не требуется.
Подключаться на всякий случай не очень-то умно. Было бы здорово подключаться к базе при
первом обращении к ней, а при последующих использовать уже созданное подключение.
Стандартная библиотека PDO ничего такого не предлагает. Этот код надо написать самому.

```php
<?php

class MySQL {
    
    private $pdoConstructorArgs;

    private $pdo;

    public function __construct(
        string $dsn,
        string $username = '',
        string $password = '',
        array $options = []
    ) {
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
        }
    
        return $this->pdo;
    }

}

```
Нам понаобится каким-то образом дать доступ к интерфейсу взаимодействия с базой
во всех местах, где мы собираемся с базой работать (таких мест может быть довольно много).
И самый простой способ сделать это -- создать глобальный (доступный отовсюду) объект.

```php
<?php

$GLOBALS['db'] = new MySQL('mysql:host=localhost;dbname=testdb', 'user', 'password');
```

Теперь в любом месте, где вам недобходимо поработать с базой данных,
вы можете обратится к глобальному объекту.

```php
<?php

$st = $GLOBALS['db']->getPdo()->query('select 1 as `test`');
foreach ($st as $row) {
    echo $row['test'], "\n";
}
```

Исмпользование суперглобального массива `$GLOBALS` довольно спорное решение. Многие
более опытные программисты могут осудить этот подход. Однако, если пользоваться им аккуратно,
то на первых парах он отлично подходит для того, чтобы сделать наш объект доступным глобально. 


## Выполнение запросов

Если вы потрудитесь разобраться с интерфейсом библиотеки PDO, то обнаружите
множество способов сделать запрос в базу. Это довольно печально, потому что
приводит к путанице с тем, как правильно поступить в той или иной ситуации.

На самом деле, в большинстве случаев запрос в базу делается по следующей схеме
(`insert`, `update`, `delete` делаются аналогично, есть небольшие тонкости, мы вернёмся к ним позже.).

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
if ( ! $st->execute()) {
    // обрабатываем ошибку
    throw new \Exception('Не удалось выполнить запрос ' . $sql);
}
// получаем строки
$rows = $st->fetchAll();
```

Без комменариев и не относящегося к запросу кода получается минимум 6 строк (вы можете найти
примеры более лаконичные, но они не будут досаточно универсальны). 6 строк каздый раз,
когда вы делает запрос, -- никуда не годится. Напишем простой метод и забудем про копипасту.

```php
<?php

class MySQL {
    
    // ...
    
    public function query(string $sql, array $params = []): \PDOStatement {
        $pdo = $this->getPdo();
        $st = $pdo->prepare($sql);
        foreach ($params as $name => $value) {
            switch (gettype($value)) {
                case 'string':
                    $type = \PDO::PARAM_STR;
                    break;

                case 'integer':
                    $type = \PDO::PARAM_INT;
                    break;

                case 'boolean':
                    $type = \PDO::PARAM_BOOL;
                    break;
    
                case 'NULL':
                    $type = \PDO::PARAM_NULL;
                    break;

                default:
                    try {
                        $value = (string) $value;
                    } catch (\Throwable $e) {
                        throw new \Exception('Недопустимый параметр ' . $name);
                    }
                    $type = \PDO::PARAM_STR;
            }
            $st->bindValue($name, $value, $type);
        }
        if ( ! $st->execute()) {
            throw new \Exception('Не удалось выполнить запрос ' . $sql);
        }

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

Я обещал рассказать про `update`, `insert`, `delete`.
Дело в том, что для этих запросов очень полезно получать количество затронутых строк,
а для `insert` ещё и последний вставленный автоинкрементарный id. Количество затронутых
строк легко получить `$st->rowCount()`. А вот последний вставленный id недоступен из
`PDOStatement`, для его получения надо вызвать метод `PDO::lastInsertId` (обратите внимание,
что поведение этих методов отличается для драйверов разных баз данных, именно поэтому я назвал класс MySQL).

Мне кажется, было бы нагляднее и полезнее завсети класс для объекта результата выполнения
запроса, котрый содержит всё необходимое.

```php
<?php

class MySQLResult {

    private $sql;
    
    private $params;
    
    private $rows;
    
    private $affectedRows;
    
    private $insertedId;
    
    public function __construct(
        string $sql,
        array  $params,
        array $rows,
        int $affectedRows,
        $insertedId
    ) {
        $this->sql = $sql;
        $this->params = $params;
        $this->rows = $rows;
        $this->affectedRows = $affectedRows;
        $this->insertedId = $insertedId;
    }

    public  function getSql(): string {
        return $this->sql;
    }

    public  function getParams(): array {
        return $this->params;
    }

    public  function getRows(): array {
        return $this->rows;
    }

    public  function getAffectedRows(): int {
        return $this->affectedRows;
    }

    public  function getInsertedId(){
        return $this->insertedId;
    }

}
```

Я добавил `getSql` и `getParams`, они пригодятся для отладки, когда запрос вернул не то,
что вы ожидали.

```php
<?php

class MySQL {
    
    // ...
    
    public function query(string $sql, array $params): \PDOStatement {
        // ...
        
        return new MySQLResult(
            $sql,
            $params,
            $st->fetchAll(),
            $st->rowCount(),
            $pdo->lastInsertId()
        );
    }
}

```

## Массивы в значениях переменных

TODO

## Работа с базой данных в большом приложении

Пока ваше приложение представляет из себя несколько скриптов, можно дёргать
`$GLOBALS['db']->query` там где вам нужно и ни о чём не переживать. Но обычно довольно скоро это становится неудобно.
Бессистемные вызовы `$GLOBALS['db']->query` расползаются по коду и их становится трудно поддерживать
(например если вы решили изменить столбцы в таблице).

Для получения контроля над запросами в базу, можно разместить все запросы в классах специальных
объектов, которые будут предоставлять интерфейс хранилища для сущностей, которыми оперирует приложение.

Допстим мы разрабатывам приложение с пользователями и постами. Мы хотим просматривать
список последних публикаций, список лучших публикаций за день, все публикации определённого
автора, ну и конечно, просматривать отдельную публикацию. Нам понадобятся сущности
Публикация и Пользователь. Чтобы получать их из базы заведём соответствующие хранилища.

```php
<?php

class PostsRepository {

    private $db;

    public function __construct(\MySQL $db) {
        $this->db = $db;
    }

    public function selectLastPosts(int $limit): array {
        $result = $this->db->query('
            select *
            from `posts`
            where `active` = true
            order by `pubDate` desc
            limit :limit 
        ', [':limit' => $limit]);
        
        return array_map([$this, 'map'], $result->getRows());
    }

    public function selectMostPopularPosts(\DateTimeInterface $toDate, int $limit): array {
        $result = $this->db->query('
            select *
            from `posts`
            where
                `active` = true
                and `pubDate` >= :toDate
            order by
                `views` desc
                `pubDate` desc
            limit :limit 
        ', [
            ':toDate' => $toDate->format('Y-m-d H:i:s'),
            ':limit' => $limit
        ]);
        
        return array_map([$this, 'map'], $result->getRows());
    }

    public function selectByAuthor(int $authorId, int $limit): array {
        $result = $this->db->query('
            select *
            from `posts`
            where
                `active` = true
                and `authorId` = :authorId
            order by `pubDate` desc
            limit :limit 
        ', [':authorId' => $authorId, ':limit' => $limit]);
        
        return array_map([$this, 'map'], $result->getRows());
    }

    public function getById(int $id): ?\Post {
        $result = $this->db->query('
            select *
            from `posts`
            where
                `active` = true
                and `id` = :id 
        ', [':id' => $id]);
        $rows = $result->getRows();
        $row = $rows[0]?? null;

        return $row? $this->map($row) : null;
    }

    private function map($row) {
        return new \Post(
            $row['id'],
            $row['title'],
            $row['pubDate']
        );
    }

}
```

Хранилище Пользователей и Комментариев создаются подобным образом. Общий код можно вынести в суперкласс.
Каждое хранилище предоставляет методы для получения сущностей в соответствии с логикой приложения.

```php
<?php

$postRepository = new \PostRepository($GLOBALS['db']);
$posts = $postRepository->selectMostPopularPosts(new DateTimeImmutable('today'), 20);

```

Обратите внимание, что мы не используем `$GLOBALS` внутри классов хранилищ. Мы передаём им объект
для работы с базой в конструкторе. Это как раз та аккуратность использования `$GLOBALS`, о которой
я говорил выше.

## Транзакции

Рано или поздно вам понадобятся транзакции. В некоторых случаях бывает сложно уследить
находимся мы в транзакции или нет. Чтобы не полагаться на память и внимательность
(которые часто подводят) можно сделать работу с транзакциями явной.

Создадим класс объекта трнзакции.

```php
<?php

class MySQLTransaction {

    private $db;

    private $steps = [];

    public function __construct(\MySQL $db) {
        $this->db = $db;
    }

    public function addStep(callable $step) {
        $this->steps[] = $step;
    }

    public function execute($initial = null) {
        $pdo = $this->db->getPdo();
        $pdo->beginTransaction();
        $result = $initial;
        foreach ($this->steps as $step) {
            try {
                $result = $step($this->db, $result);
            } catch (\Throwable $e) {
                $pdo->rollBack();
                throw $e;
            }
        }
        $pdo->commit();
        
        return $result;
    }

}

```

Добавим фабричный метод в `MySQL`.

```php
<?php

class MySQL {
    
    // ...
    
    public function createTransaction(...$steps): \MySQLTransaction {
        $transaction = new \MySQLTransaction($this);
        foreach ($steps as $step) {
            $transaction->addStep($step);
        }

        return $transaction;
    }
}

```

Вот например как можно это использовать.

```php
<?php

$accountA = 123;
$accountB = 456;
$sum = 100500;
$transaction = $GLOBALS['db']->createTransaction(
    function (\MySQL $db) use ($accountA, $accountB, $sum) {
        $result = $db->query('
            select `amount`
            from `accounts`
            where `id` = :id
            for update
        ', [':id' => $accountA]);
        $rows = $result->rows();
        $amount = $rows[array_key_first($rows)]['amount']?? null;
        if ($amount === null) {
            throw new \Exception('Не найден кошелёк ' . $accountA);
        } elseif ($amount < $sum) {
            throw new \Exception('Не достаточно средств на кошельке ' . $accountA);
        }
        $db->query('
            update `accounts`
            set `amount` = `amount` - :sum
            where `id` = :id
        ', [':sum' => $sum, ':id' => $accountA]);
        $db->query('
            update `accounts`
            set `amount` = `amount` + :sum
            where `id` = :id
        ', [':sum' => $sum, ':id' => $accountB]);
    
        return [$accountA, $accountB, $sum];
    }
);
$transaction->addStep(function(\MySQL $db, $carry) {
    list($accounA, $accounB, $sum) = $carry;
    $db->query('
        insert into `paymentsLog`
        set
            `from` = :from,
            `to` = :to
            `sum` = :sum
    ', [
        ':from' => $accounA,
        ':to' => $accounB,
        ':sum' => $sum,
    ]);
});
$transaction->execute();
```

Со временем вы познакомитесь с гораздо более могучими библиотеками для работы
с базами данных. Но изложенные здесь принципы и решения наверняка пригодятся вам ещё не раз. 
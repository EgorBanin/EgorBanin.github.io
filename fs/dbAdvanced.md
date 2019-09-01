! Черновик

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

Хранилище Пользователей создаётся подобным образом. Общий код можно вынести в суперкласс.
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

Создадим класс объекта транзакции.

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
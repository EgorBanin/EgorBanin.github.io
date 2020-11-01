В разработке

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
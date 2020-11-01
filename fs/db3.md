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
для работы с базой в конструкторе.



Со временем вы познакомитесь с гораздо более могучими библиотеками для работы
с базами данных. Но изложенные здесь принципы и решения наверняка пригодятся вам ещё не раз. 
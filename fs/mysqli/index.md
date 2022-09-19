# Основы работы с MySQL (MariaDB) с помощью mysqli

Если вы только осваиваетесь в php, то скорее всего допускаете неочевидные ошибки при работе с базой данных. Особенно досадно, что такие ошибки могут быть довольно опасными. Я постараюсь вкратце объяснить, как всё работает, и помочь вам избежать типичных проблем.

## Что такое база данных и mysqli

База данных это специальная программа, которая умеет сохранять и извлекать ранее сохранённые данные. Она чем-то похожа на файл -- вы можете записать в неё данные, а можете прочитать их. Но в отличие от файла база данных предлагает гораздо более мощные инструменты взаимодействия с данными. Например MySQL позволяет работать с данными множеству пользователй одновременно, поддерживает разнообразные типы данных и язык запросов SQL, оптимизирует доступ к данным, обеспечивая высокую производительность.

Чтобы работать с базой данных, надо подключиться и взаимодействовать с ней с помощью специального интерфейса. Этот интерфейс в php обеспечивает библиотека [mysqli](https://www.php.net/manual/ru/book.mysqli.php). С этой библиотекой можно работать в процедурном и в ООП-стиле. В этой статье я буду разбирать только процедурный стиль.

## Подключение

Для подключения к базе используется функция [mysqli_connect](https://www.php.net/manual/ru/function.mysqli-connect.php). Код подключения выглядит приблизительно так:

```
$db = mysqli_connect('localhost', 'username', 'password', 'defaultDb', 3306);
```

Попытка подключения может быть неуспешной по разным причинам (например, админ поменял пароль). Поведением mysqli при ощибках управляет функция [mysqli_report](https://www.php.net/manual/ru/mysqli-driver.report-mode.php). До php 8.1 по умолчанию в случае проблем функции mysqli будут вызывать ошибки, после  -- исключения. Исключения гораздо удобнее, но если вы ещё не разобрались с ними, то я покажу как работать с ошибками. Для этого установим MYSQLI_REPORT_OFF.

```
mysqli_report(MYSQLI_REPORT_OFF);
$db = @mysqli_connect('localhost', 'username', 'password', 'defaultDb', 3306);
if ( ! $db) {
	echo '
		<h1>Технические неполадки</h1>
		<p>
			Сайт временно не работает, зайдите позже или свяжитесь с нами по телефону
			<a href="tel:+71234567890">+7 123 456-78-90</a>.
		</p>
	';
	error_log(sprintf('Не удалось подключиться к базе данных: %s', mysqli_connect_error()));
	exit(1);
}
```

> **Замечание насчёт @**
> 
> Вы могли встретить рекомендацию не использовать @ в своём коде. Это хороший совет, во многих случаях лучше управлять ошибками через настройки [error_reporting](https://www.php.net/manual/ru/errorfunc.configuration.php#ini.error-reporting) и [display_errors](https://www.php.net/manual/ru/errorfunc.configuration.php#ini.display-errors). Но в этом руководстве для новичков я осознано упрощаю код, чтобы вы могли сосредоточится именно на работе с mysqli.

Обработка ошибок -- важная часть любой программы. Именно в случае возникновения проблем пользователи особенно требовательны к вашему приложению. Кроме того и вам, хорошо бы получить уведомление о проблеме, чтобы как можно скорее отреагировать на неё. Поэтому не ограничивайтесь конструкцией типа `or die`. Обратите внимание, что в разных случаях на ошибки надо реагировать по-разному. Если мы пишем интерфейс для работы с базой данных, где пользователь сам вводит имя и пароль (например, phpMyAdmin или adminer), то ошибка подключения -- проблема пользователя, и ему надо предложить проверить и исправить данные подключения. Но если мы делаем сайт интернет-магазина, то та же ошибка подключения -- проблема магазина, и магазину надо помочь пользователю сориентироваться в этой ситуации.

### Настройка кодировки

Строки в вашем php-файле, строки, переданные вам пользователем, и строки в базе данных могут быть в разных кодировках. Обязательно настройте кодировку подключения с помощью [mysqli_set_charset](https://www.php.net/manual/ru/mysqli.set-charset.php).

```
if ( ! mysqli_set_charset($db, "utf8mb4")) {
	internalServerError(mysqli_error($db)); // какой-то ваш обработчик ошибки
}
```

Установка кодировки означает, что вы будете отправлять запросы в указанной кодировке, убедитесь что это действительно так. Если у вас всё в utf-8 (в современной разработке это почти всегда так), то `mysqli_set_charset($db, "utf8mb4")` -- правильный выбор.

> **utf8 и utf8mb4**
> 
> Исторически сложилось, что в MySQL реализация utf-8 была не полной, но в какой-то момент появилась полноценная. [utf8](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8.html) -- алиас для [utf8mb3](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8mb3.html), а новую назвали [utf8mb4](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8mb4.html).


## Запросы

Если подключение прошло удачно, можно пользоваться созданным подключением для выполнения запросов. Функция, которая делает запрос в базу данных и получает результат, называется [mysqli_query](https://www.php.net/manual/ru/mysqli.query.php). В случае проблем с доступом к базе данных или при ошибках в запросе она тоже будет возвращать false. Если запрос должен возвращать строки, то получить их можно с помощью [mysqli_fetch_all](https://www.php.net/manual/ru/mysqli-result.fetch-all.php) или, если вам недоступна эта функция (php < 8.1, с библиотекой libmysqlclient), [mysqli_fetch_assoc](https://www.php.net/manual/ru/mysqli-result.fetch-assoc.php).

```
$result = @mysqli_query($db, '
	select
		`id`,
		`title`
	from `articles`
	where `isPublished` = true
	order by `id` desc
');
if ( ! $result) {
	internalServerError(mysqli_error($db)); // какой-то ваш обработчик ошибки
}

$rows = [];
while ($row = mysqli_fetch_assoc($result)) {
	$rows[] = $row;
}

var_dump($rows); // [['id' => 1, 'title' => 'Основы работы с MySQL (MariaDB) с помощью mysqli'], ...]
```

### Переменные в запросах

При подстановке переменных в запрос надо быть особенно внимательным. Если просто канкатенировать строчки, то SQL-запрос может сломаться или даже стать вредноносным (например, удалит все данные из таблицы). Чтобы стало понятно как это работает, приведу пример:

```
// предположим, что мы разрешаем пользователю удалять статьи, автором которых он является
mysqli_query($db, '
	delete
	from `articles`
	where
		`author` = ' . $usreId . '
		and `id` = ' . $_POST['id'] . ' // это небезопасно!
');
```

Может показаться, что запрос верный. Но так как мы не можем знать, что пользователь передаст в `$_POST['id']`, то эта канкатенация может сильно изменить запрос. Допустим, пользователь передаст `1 or true`. В таком случае он удалит все статьи. Это называется [SQL-инъекция](https://ru.wikipedia.org/wiki/%D0%92%D0%BD%D0%B5%D0%B4%D1%80%D0%B5%D0%BD%D0%B8%D0%B5_SQL-%D0%BA%D0%BE%D0%B4%D0%B0).

Валидация не является выходом. Например, если мы добавляем в базу имя пользователя, которого зовут д'Артаньян (вполне валидное имя), то апостроф в его имени может быть интерпретирован базой как начало или конец строкового значения и вызовет ошибку.

Одно из решений этой проблемы -- экранирование. Специальня функция [mysqli_real_escape_string](https://www.php.net/manual/ru/mysqli.real-escape-string.php) экранирует все специальные символы строки, которые могли бы быть восприняты базой как часть SQL-синтаксиса. Обратите внимание на предупреждение в документации: "набор символов должен быть задан либо на стороне сервера, либо с помощью API-функции [mysqli_set_charset()](https://www.php.net/manual/ru/mysqli.set-charset.php)".

Все строки, вставляемые в SQL, должны быть экранированы этой функцией:

```
$name = $_POST['name']?? '';

$result = mysqli_query($db, '
	select
		`articles`.`id`,
		`articles`.`title`
	from `articles`
	inner join `users`
		on `users`.`id` = `articles`.`author`
	where
		`users`.`name` = "' . mysqli_real_escape_string($db, $name) . '"
		and `articles`.`isPublished` = true
	order by `articles`.`id` desc
	limit 10
');
```

Теперь мы можем получить статьи даже того автора, который назвал себя `Robert"; drop table students;--`.

Для чисел избежать инъекций поможет приведение типа:

```
$id = $_POST['id']?? '';

// предположим, что мы разрешаем пользователю удалять статьи, автором которых он является
mysqli_query($db, '
	delete
	from `articles`
	where
		`author` = ' . (int) $usreId . '
		and `id` = ' . (int) $id . ' // теперь всё ok
');
```

И ещё раз. Отнеситесь к экранированию очень серьёзно. Любые данные, которые приходят из запроса пользователя -- это строки, которые нельзя подставлять в запрос просто так.

> **Подготовленные запросы**
> 
> Другим способом подставить значения переменных в запрос являются [подготовленные запросы](https://www.php.net/manual/ru/mysqli.quickstart.prepared-statements.php). Обязательно изучите эту тему самостоятельно, когда освоитесь с основами mysqli.


## Заключение

- Чтобы не писать код подключения каждый раз, когда вам нужно сделать запрос в базу данных, создайте файл initDb.php и подлючайте его с помощью require_once.
- Не забудте настроить кодировку подключения с помощью mysqli_set_charset сразу после подключения.
- Не помещайте пользовательские данные в запрос без экранирования.
- Изучайте [документацию mysqli](https://www.php.net/manual/ru/book.mysqli.php). Там много полезного. 

Освоив азы, изучите объектно ориентированный интерфейс mysqli, подготовленные запросы, асинхронные запросы. Познакомьтесь с другой стандартной библиотекой для работы с базами данных -- PDO. Изучайте возможности вашей базы данных: типы данных, индексы, транзакции, партиционирование и репликацию.

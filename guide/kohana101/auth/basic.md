# Начало работы

Данный модуль представляет из себя систему [авторизации](http://ru.wikipedia.org/wiki/%D0%90%D0%B2%D1%82%D0%BE%D1%80%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F)
 пользователей. Это означает, что он не только идентифицирует пользователя, но и определяет наличие у него прав на
 выполнение каких-либо действий.

## Структура модуля {#structure}

По сути модуль состоит всего из нескольких файлов:

 * `classes/kohana/auth.php` - основная библиотека, содержащая базовые функции модуля. Класс абстрактный, работает через
   [драйверы](auth/drivers).
 * `classes/kohana/auth/file.php` - единственный штатный драйвер **Auth**. Для хранения пользовательских данных использует
   конфигурационные файлы.
 * `config/auth.php` - конфигурационный файл по умолчанию. Содержит базовые настройки для использования модуля **Auth** с
   драйвером [File](auth/file/basic).

## Создание {#new}

Модуль предполагает использование только одного экземпляра объекта **Auth** ([паттерн Singleton](http://ru.wikipedia.org/wiki/%D0%9E%D0%B4%D0%B8%D0%BD%D0%BE%D1%87%D0%BA%D0%B0_(%D1%88%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD_%D0%BF%D1%80%D0%BE%D0%B5%D0%BA%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F))).
 Класс **Auth** содержит статический метод [Auth::instance], для создания и доступа к этому экземпляру:

	// создаем Auth с настройками по умолчанию
	$auth = Auth::instance();
	// впрочем, не запрещено и создание отдельного объекта Auth через конструктор
	$auth1 = new Auth();
	// можно передать собственные настройки в обход конфига
	$auth2 = new Auth($params);
	// и это все разные объекты Auth!

[!!] Объект, созданный напрямую через конструктор, не будет доступен через `Auth::instance()`.


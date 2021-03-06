# Автоматический вход

В данной статье будет рассмотрен механизм "запоминания" пользователей в модуле **Auth**.

## Как это работает {#about}

1. Пользователь при входе ставит галочку "запомнить меня".
2. При вызове `Auth::instance()->login()` в качестве третьего параметра передается **TRUE**.
3. В случае успешного входа модуль создает специальный токен - объект, связывающий учетную запись с текущей клиентской
 сессией. Токен содержит:
 - Идентификатор учетной записи (то есть `user_id` модели `User`)
 - Хеш поля User Agent (хранится в `Request::$user_agent`)
 - Время жизни (метку времени, после которой он становится устаревшим)
 - Сгенерированную уникальную строку из 32 символов (буквы и цифры)
4. Токен сохраняется в БД (таблица `user_tokens`) и в куках браузера пользователя.
5. В следующий раз, когда пользователь откроет страницу без аутентификации (в качестве гостя), методы [Auth::logged_in]
 и [Auth::get_user] автоматически вызовут [Auth::auto_login]. Если в куках обнаружится актуальный токен, по нему из таблицы
  `user_tokens` будет извлечена информация о пользователе. Сперва осуществляется сверка User Agent - если его хеш совпадает
  с сохраненным в токене, то токен будет использован для автоматического входа без ввода пароля.

## Как использовать {#using}

Специально вызывать метод `auto_login()` не требуется. Так как стандартная процедура проверки аутентификации содержит
 вызов [Auth::get_user] или [Auth::logged_in], то проверка на автоматический вход будет осуществлена на этом этапе.

## Очистка токенов {#gc}

Постепенно в таблице `user_tokens` накапливаются сохраненные токены. Так как они имеют ограниченный срок жизни, необходимо
 периодически удалять старые токены. Это происходит автоматически при создании нового токена. Примерно 1 раз из 100 инициируется
 очистка устаревших токенов с помощью метода [Model_User_Token::delete_expired].

# Кэширование ответов

Скорее всего, вы уже знакомы с HTTP-кэшированием, [применяемым](basic/request/caching) при работе с [запросами](basic/request).
 Там вся основная работа велась на уровне запроса, то есть __запрашивающая__ сторона в бОльшей степени управляла кэшированием.
 В данной статье будет рассматриваться вариант с управлением на уровне источника данных, то есть кэширование ответов.

## Общая информация

Наиболее частая ситуация - необходимость кэшировать различного рода статику: картинки, скрипты, стили и т.д. Обычно эта
 задача возлагается на web-сервер, но иногда необходимо реализовать кэширование своими силами. Согласно спецификации
 HTTP/1.1, нам необходимо использовать так называемые [Entity Tags](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.11)
 (сокращенно `ETags`).

Схема простая:

 * Сервер вычисляет строку (`ETag`), идентифицирующую отправляемый ответ на запрос. Логично, что значение ETag для двух одинаковых
  ответов должно совпадать.
 * Вместе с запрошенными данными возвращается специальный заголовок `ETag`, позволяющий клиенту связать URL с этим
  сгенерированным идентификатором.
 * При последующих запросах браузер клиента сам добавит HTTP-заголовок `If-None-Match`, содержащий тот самый `ETag` в качестве
  значения.
 * Сервер обрабатывает поступивший запрос, заново вычисляет `ETag` для отправляемого ответа, и сравнивает его с полученным
  от клиента. Если они совпадают, то можно не отправлять тело ответа, достаточно вернуть HTTP-код _304 Not Modified_ и закрывать
  соединение.

[!!] Конечно, использование `ETag` не освобождает сервер от необходимости принять и обработать запрос, подготовить ответ. Но, как
 минимум, уменьшается длительность соединения (меньше времени на выдачу ответа). А затраты на обработку запроса уменьшаются
 засчет других видов кэширования.

## Простой пример

Достаточный минимум для генерации `ETag` в **Kohana** - вызов метода `check_cache()` объекта `Response`:

	class Controller_Welcome extends Controller {

		public function action_index()
		{
			$this->response->body('Hello World');
			// просим систему автоматически сгенерировать ETag и отдать клиенту
			$this->response->check_cache(NULL, $this->request);
		}

	}

Первый параметр метода `check_cache()` - это само значение `ETag`. Мы можем вычислять его самостоятельно, а можем возложить это
 на систему. Во втором случае в качестве `ETag` выступит hash ответа.

[!!] Обратите внимание, что необходимо передать текущий объект `Request` в качестве параметра. С его помощью объект `Response`
 проверит поступившие HTTP-заголовки на наличие в них соответствующего `ETag`.

[!!] Вызывая `check_cache()`, будьте готовы к тому, что дальнейшая обработка запроса закончится. Если значения `ETag` совпали, то
 с точки зрения сервера, дальнейшая работа не имеет смысла, т.к. у клиента в кэше имеется актуальная копия ответа.

Запустив `http://kohana/welcome` в первый раз, мы увидим примерно такие заголовки ответа:

	Cache-Control:must-revalidate
	Connection:keep-alive
	Content-Length:11
	Date:Sat, 27 Oct 2012 19:26:35 GMT
	Etag:"cbca887ffdab3de0f456798111da222f96b4c894"
	Server:nginx/1.2.2
	X-Powered-By:PHP/5.3.16

Обновим браузер, и обнаружим, что код ответа поменялся - вместо _200_ (_OK_) пришел _304_ (_Not Modified_). Это значит, что сервер
 шлет нам сигнал - используйте отправленный ранее ответ. Если посмотреть в отправленные заголовки запроса, то увидим там
 заголовок `If-None-Match`, который позволил определить, что какая-то версия ответа у нас уже есть, мы просто не уверены в
 ее актуальности:

	Cache-Control:max-age=0
	Connection:keep-alive
	Host:kohana
	If-None-Match:"cbca887ffdab3de0f456798111da222f96b4c894"

## Более сложный пример

Чтобы далеко не ходить, рассмотрим реальный код - контроллер `Controller_Userguide` из модуля `Userguide`. Он используется
 для показа страниц документации (в том числе и этой), а его метод `action_media()` отвечает за выдачу статики (иллюстрации
 к статьям). В экшен передается имя файла для загрузки.

	public function action_media()
	{
		// Get the file path from the request
		$file = $this->request->param('file');

		// Find the file extension
		$ext = pathinfo($file, PATHINFO_EXTENSION);

		// Remove the extension from the filename
		$file = substr($file, 0, -(strlen($ext) + 1));

		if ($file = Kohana::find_file('media/guide', $file, $ext))
		{
			// Check if the browser sent an "if-none-match: <etag>" header, and tell if the file hasn't changed
			$this->response->check_cache(sha1($this->request->uri()).filemtime($file), $this->request);

			// Send the file content as the response
			$this->response->body(file_get_contents($file));

			// Set the proper headers to allow caching
			$this->response->headers('content-type',  File::mime_by_ext($ext));
			$this->response->headers('last-modified', date('r', filemtime($file)));
		}
		else
		{
			// Return a 404 status
			$this->response->status(404);
		}
	}

Первое, на что стоит обратить внимание - `ETag` генерируется вручную. Это хеш от двух сцепленных строк - запрошенного URI
 (который по сути однозначно идентифицирует файл картинки) и метка времени последнего изменения файла. В совокупности эти
 два значения однозначно определяют ответ на запрос - у других файлов будет отличаться URI, а старые версии этой картинки
 имели другую метку времени. Таким образом, хеш от этой строки можно считать уникальным идентификатором запрашиваемого ресурса.
 Выгода в том, что сам файл еще даже не был загружен - это потребуется только если не сработает кэширование через `ETag`.

Естественно, никто не мешает комбинировать различные виды кэширования. Например, можно дополнительно отправлять заголовок
 `Cache-Control: max-age` или `Expires`, чтобы браузер клиента какое-то время брал изображение из кэша, не обращаясь к нашему
 серверу.


## Итоговое задание

Напишите функцию `fetch_resource(resource_id, timeout)`, которая:

- Принимает два аргумента: `resource_id` - ID записи, `timeout` - время ожидания ответа (опционально)
- Выполняет GET-запрос к `https://jsonplaceholder.typicode.com/posts/{resource_id}`
- Если статус ответа **200**, логирует `INFO` и возвращает Pydantic модель.
- Если статус ответа **404**, логирует `ERROR` с описанием и возвращает None.
- При других ошибках (тайм-аут, соединение), логирует на уровне `WARNING` и возвращает None.
- Используйте блок `try/except` с `logger.exception()` для полного стека.
- Добавить аннотации типов

Способы проверки:
1) `resource_id=1` (существует). Вызов: `fetch_resource(resource_id=1)`
2) `resource_id=999` (не существует). Вызов: `fetch_resource(resource_id=999)`
3) Ошибка тайм-аута (передать `timeout=0.001`)

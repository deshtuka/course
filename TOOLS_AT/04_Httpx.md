# Тема 4. HTTP-клиент (Httpx)

## Введение

В предыдущих темах мы научились описывать структуры данных с помощью Pydantic и аннотаций типов. Теперь пришло время научиться получать эти данные из внешнего мира — через HTTP-запросы. В автотестах API это центральное действие: мы отправляем запросы, получаем ответы, валидируем их.

**HTTPX** — это современный HTTP-клиент для Python 3, который пришёл на смену устаревающему `requests`. Он сохраняет знакомый и простой API, к которому все привыкли, но добавляет возможности, необходимые в современной разработке: асинхронность, HTTP/2, строгие таймауты и полную поддержку аннотаций типов.

Для тестировщика HTTPX даёт:
- **Единый интерфейс** — синхронный и асинхронный API в одной библиотеке.
- **Производительность** — переиспользование соединений через клиент (Client) значительно ускоряет выполнение тестов.
- **Современные протоколы** — поддержка HTTP/2 для более эффективной работы.
- **Безопасность по умолчанию** — в отличие от `requests`, HTTPX имеет разумные таймауты.

> Официальная документация: https://www.python-httpx.org/

---

## Как установить HTTPX?

Так как это библиотека сторонняя то перед её использованием нужно установить следующей командой:
```
pip install httpx
```

---

### Базовые запросы

```python
import httpx

# GET-запрос
response = httpx.get('https://api.example.com/users')
print(response.status_code)  # 200
print(response.json())       # парсинг JSON

# POST-запрос с JSON телом
response = httpx.post(
    'https://api.example.com/users',
    json={'name': 'Alice', 'email': 'alice@example.com'}
)

# POST с form-data
response = httpx.post(
    'https://api.example.com/login',
    data={'username': 'alice', 'password': 'secret'}
)

# Заголовки
response = httpx.get(
    'https://api.example.com/protected',
    headers={'Authorization': 'Bearer token123'}
)
```

### Параметры в запросе

```python
# Query-параметры
response = httpx.get(
    'https://api.example.com/search',
    params={'q': 'python', 'page': 2}
)
# URL станет: https://api.example.com/search?q=python&page=2
```

### Обработка ответа

```python
response = httpx.get('https://api.example.com/users/1')

# Статус код
response.status_code

# Текст ответа
response.text          # как строка
response.content       # как байты
response.json()        # парсинг JSON (выбросит исключение если не JSON)

# Заголовки
response.headers['Content-Type']

# Информация о запросе
response.request.url
response.request.headers
```

### Обработка ошибок

HTTPX имеет иерархию исключений, которая позволяет гибко обрабатывать ошибки:

```python
import httpx

try:
    response = httpx.get('https://api.example.com/users/1')
    response.raise_for_status()  # выбросит исключение HTTPStatusError для 4xx/5xx
except httpx.HTTPStatusError as e:
    print(f'Ошибка HTTP: {e.response.status_code} - {e.response.text}')
except httpx.ConnectError as e:
    print(f'Не удалось подключиться: {e}')
except httpx.TimeoutException as e:
    print(f'Таймаут: {e}')
except httpx.HTTPError as e:
    print(f'Общая ошибка HTTP: {e}')
```

Иерархия исключений позволяет отлавливать как конкретные ошибки, так и общие.

---

## Клиент (Client)

Если вы делаете больше одного запроса к одному хосту, **всегда используйте `Client`**. Это аналог `requests.Session()`, но мощнее.

### Зачем нужен Client?

При использовании топ-уровневых функций (`httpx.get()`, `httpx.post()`) HTTPX устанавливает новое соединение для каждого запроса. `Client` использует пул соединений и переиспользует TCP-подключения, сохраняя куки, заголовки сессии, что даёт:
- снижение задержки (без повторного handshake);
- снижение нагрузки на CPU;
- снижение сетевого трафика.

### Создание и использование Client

```python
import httpx

# Рекомендуемый способ — через контекстный менеджер
with httpx.Client() as client:
    response = client.get('https://api.example.com/users')
    # соединения автоматически закрываются при выходе из блока
```

Или с явным закрытием:

```python
client = httpx.Client()
try:
    response = client.get('https://api.example.com/users')
finally:
    client.close()
```

### Настройка Client

Client позволяет задать параметры, которые будут применяться ко всем запросам:

```python
with httpx.Client(
    base_url='https://api.example.com',           # базовый URL
    headers={'User-Agent': 'my-tests/1.0'},       # заголовки для всех запросов
    timeout=30.0,                                 # таймаут для всех запросов
    follow_redirects=True,                        # следовать редиректам
    cookies={'session_id': 'abc123'},             # cookie для всех запросов
    params={'api_key': 'key123'},                 # параметры для всех запросов
) as client:
    # Теперь можно указывать только относительный путь из-за base_url
    response = client.get('/users/1')  # → https://api.example.com/users/1?api_key=key123
    response = client.post('/users', json={'name': 'Bob'})
```

### Объединение конфигураций

Параметры, заданные на уровне клиента и на уровне конкретного запроса, объединяются:

```python
with httpx.Client(
    headers={'X-Auth': 'from-client'},
    params={'client_id': 'client1'}
) as client:
    response = client.get(
        'https://example.com',
        headers={'X-Custom': 'from-request'},
        params={'request_id': 'request1'}
    )
    # В запросе будут и X-Auth, и X-Custom
    # Параметры: client_id=client1&request_id=request1
```

### Client в автотестах

В тестах Client обычно создаётся один раз на весь тестовый класс или модуль (подробнее будет в другом разделе). Пример:

```python
import pytest
import httpx

class TestUserAPI:
    @pytest.fixture
    def client(self):
        with httpx.Client(
            base_url='https://api.example.com',
            timeout=10.0,
            headers={'Accept': 'application/json'}
        ) as client:
            yield client

    def test_get_user(self, client):
        response = client.get('/users/1')
        assert response.status_code == 200
        data = response.json()
        assert data['id'] == 1
```

---

## Распространенные ошибки

### 1. Забыли про таймауты

В `requests` таймаут нужно было задавать явно. В HTTPX он есть по умолчанию, но иногда его нужно настраивать под конкретные сценарии. Если не задать таймаут на клиенте, а запросы к медленному API будут висеть, тесты могут неожиданно падать по `ConnectTimeout`.

**Решение:** всегда явно задавайте `timeout` при создании Client.

```python
client = httpx.Client(timeout=30.0)  # 30 секунд на все операции
```

Можно задать разные таймауты для разных этапов:

```python
timeout = httpx.Timeout(30.0, connect=5.0)  # 30 сек общий, 5 сек на подключение
client = httpx.Client(timeout=timeout)
```

### 2. Неправильная работа с редиректами

В отличие от `requests`, HTTPX **не** следует редиректам по умолчанию. Если ваш API возвращает 301/302, запрос не дойдёт до конечного ресурса.

**Решение:** явно включите `follow_redirects`:

```python
# На уровне конкретного запроса
response = client.get(url, follow_redirects=True)

# Или на уровне клиента
client = httpx.Client(follow_redirects=True)
```

### 3. Путаница между `data` и `content`

В HTTPX есть чёткое разделение:
- `data` — для form-data (словарь);
- `json` — для JSON;
- `content` — для сырых байтов или текста.

Использование `data` с текстом/байтами вызовет предупреждение об устаревании и будет удалено в HTTPX 1.0.

**Правильно:**
```python
# form-data
httpx.post(url, data={'key': 'value'})

# JSON
httpx.post(url, json={'key': 'value'})

# сырой текст
httpx.post(url, content=b'raw data')
```

### 4. Проблемы с кодировкой при загрузке файлов

HTTPX строго требует, чтобы файлы открывались в **бинарном режиме**:

```python
# Неправильно
with open('file.txt', 'r') as f:
    httpx.post(url, files={'file': f})

# Правильно
with open('file.txt', 'rb') as f:
    httpx.post(url, files={'file': f})
```

### 5. Cookie на уровне запроса

В HTTPX cookie должны задаваться на уровне клиента, а не на уровне отдельного запроса:

```python
# Не поддерживается
client = httpx.Client()
client.post(url, cookies={'key': 'value'})

# Поддерживается
client = httpx.Client(cookies={'key': 'value'})
client.post(url)
```

### 6. Создание Client внутри цикла

Создание нового Client для каждого запроса в цикле уничтожает все преимущества пула соединений:

```python
# Плохо — каждый раз новый Client
for user_id in user_ids:
    with httpx.Client() as client:
        response = client.get(f'/users/{user_id}')

# Хорошо — один Client на все запросы
with httpx.Client() as client:
    for user_id in user_ids:
        response = client.get(f'/users/{user_id}')
```

### 7. Использование с Pydantic

Теперь соединим Pydantic с HTTPX. Вместо того чтобы вручную проверять каждый ключ ответа, мы сразу превратим ответ в модель и затем проверим значения.

Пример функции, которая получает пользователя по ID и возвращает модель `User`:

```python
import httpx
from pydantic import ValidationError

def get_user(client: httpx.Client, user_id: int) -> User | None:
    response = client.get(f"/users/{user_id}")
    if response.status_code == 200:
        try:
            return User(**response.json())
        except ValidationError as e:
            # Логируем ошибку и возвращаем None или выбрасываем исключение
            print(f"Ошибка валидации: {e}")
            return None
    return None
```

В тесте мы можем использовать эту функцию и проверять поля:

```python
def test_user_retrieval(client):
    user = get_user(client, 1)
    assert user is not None
    assert user.id == 1
    assert user.email.endswith("@example.com")
```

Если API изменится и перестанет возвращать поле `email`, Pydantic упадет, и тест укажет на проблему раньше, чем мы дойдём до проверки.

---

## Краткий итог

- **HTTPX** — это современная замена `requests` с поддержкой асинхронности, HTTP/2 и строгой типизацией.
- API максимально совместим с `requests`, поэтому переход требует минимальных усилий.
- Для тестов **всегда используйте `Client`** — это даёт пул соединений, переиспользование TCP и значительный прирост производительности.
- Настраивайте `timeout`, `follow_redirects` и другие параметры на уровне клиента — это сделает тесты предсказуемыми.
- Не забывайте про отличия от `requests`: редиректы по умолчанию выключены, cookie только на клиенте, строгое разделение `data`/`json`/`content`.
- Обрабатывайте исключения — иерархия HTTPX позволяет гибко реагировать на разные типы ошибок.

В следующей теме мы соединим HTTPX с Pydantic и Loguru, чтобы получить полноценный инструмент для тестирования API: отправляем запрос → логируем → валидируем ответ моделью.

---

## Аналоги: почему HTTPX лучше requests?

Библиотека `requests` много лет была стандартом де-факто для HTTP-запросов в Python. Однако индустрия не стоит на месте, и сегодня HTTPX предлагает ряд преимуществ, критически важных для современного тестирования.

### 1. Асинхронность из коробки

`requests` — исключительно синхронная библиотека. Если вам нужно отправить 100 запросов параллельно, придётся использовать многопоточность, что нагружает систему.

HTTPX поддерживает как синхронный, так и асинхронный режимы. В асинхронном режиме один поток может обслуживать тысячи одновременных соединений. Результаты тестов впечатляют: **HTTPX выполняет 100 параллельных запросов за 0.19 секунд**. Для сравнения, `requests` с многопоточностью показывает значительно худшие результаты.

В рамках базового курса мы будем использовать синхронный режим, но сам факт наличия асинхронности говорит о том, что библиотека готова к росту ваших проектов.

### 2. HTTP/2

`requests` не поддерживает HTTP/2. HTTPX — поддерживает. HTTP/2 позволяет мультиплексировать запросы в рамках одного соединения, что dramatically снижает задержки и улучшает обработку запросов. Для тестирования современных API это становится всё более актуальным.

### 3. Клиент с пулом соединений

В `requests` для переиспользования соединений нужно использовать `Session`. В HTTPX это `Client`. Однако HTTPX идёт дальше: клиент поддерживает не только cookie и заголовки, но и настройки прокси, таймаутов, HTTP/2 и многое другое.

### 4. Таймауты по умолчанию

Это критично для тестов. `requests` не имеет таймаутов по умолчанию — запрос может висеть бесконечно, «уронив» всё прогон тестов. HTTPX по умолчанию использует разумные таймауты для всех сетевых операций. Если вам нужно поведение как в `requests`, вы можете явно установить `timeout=None`.

### 5. Строгая типизация

HTTPX полностью аннотирован типами. В связке с современной IDE вы получаете автодополнение и раннее обнаружение ошибок — то, чего так не хватало в `requests`.

### Сравнительная таблица

| Возможность           | requests           | HTTPX            |
|-----------------------|--------------------|------------------|
| Синхронные запросы    | +                  | +                |
| Асинхронные запросы   | -                  | +                |
| HTTP/2                | -                  | +                |
| Таймауты по умолчанию | -                  | +                |
| Аннотации типов       | Частично           | Полностью        |
| Пул соединений        | Session            | Client           |
| Современный стек      | urllib3            | HTTPCore         |
| Сообщество            | Огромное (10+ лет) | Активно растущее |


---

### Тестовые задания

**1. Каковы ключевые особенности библиотеки HTTPX?**

А) Поддержка как синхронного, так и асинхронного API, а также HTTP/1.1 и HTTP/2.
Б) Только синхронный API и поддержка HTTP/1.1.
В) Только асинхронный API, предназначенный для высоких нагрузок.  
Г) Это форк библиотеки Requests, не поддерживающий HTTP/2.

**Правильный ответ:** А

---

**2. Что произойдёт, если при использовании `httpx.get()` не указать параметр `timeout`?**

А) Запрос будет выполняться бесконечно, пока не получит ответ.  
Б) **Будет использован таймаут по умолчанию — 5 секунд на каждую фазу (подключение, чтение, запись).**  
В) Будет возбуждено исключение `TimeoutError`.  
Г) Таймаут будет установлен на 60 секунд.

**Правильный ответ:** Б

---

**3. Какой метод используется для явного закрытия `Client` и освобождения всех ресурсов, если он **не используется** в качестве контекстного менеджера?**

А) `client.close()`
Б) `await client.aclose()`
В) `client.shutdown()`
Г) `del client`

**Правильный ответ:** A

---

**4. Как правильно выполнить синхронный GET-запрос с использованием HTTPX без создания клиента?**

А) `httpx.get_sync('https://api.example.com')`  
Б) `httpx.Client().get('https://api.example.com')`
В) `httpx.request('GET', 'https://api.example.com')`  
Г) `httpx.get('https://api.example.com')`

**Правильный ответ:** Г

---

**5. Как в HTTPX правильно передать JSON-данные в POST-запросе?**

А) `client.post(url, data=json.dumps({'key': 'value'}))`  
Б) **`client.post(url, json={'key': 'value'})`**  
В) `client.post(url, body={'key': 'value'})`  
Г) `client.post(url, form={'key': 'value'})`

**Правильный ответ:** Б

---

**6. Как получить содержимое ответа в виде строки (текста) в HTTPX?**

А) `response.body`  
Б) **`response.text`**  
В) `response.content`  
Г) `response.read()`

**Правильный ответ:** Б

---

**7. Какой тип ответа возвращается при успешном выполнении синхронного запроса в HTTPX?**

А) `dict`  
Б) `bytes`  
В) **`httpx.Response`**  
Г) `requests.Response`

**Правильный ответ:** В

---

### Практическое задание

Создайте Client с базовым URL `https://jsonplaceholder.typicode.com`, таймаутом 5 секунд и заголовком `Accept: application/json`. Напишите функцию, которая получает список постов `/posts` и добавьте следующие проверки:
- Кол-во постов больше 0
- Статус код 200

**Дополнительное задание**: провалидируйте контракт JSON ответа с помощью библиотеки Pydantic изученной ранее

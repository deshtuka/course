# Тема 6. Декораторы

## Введение

Это один из самых мощных и элегантных инструментов Python. Они позволяют «обернуть» существующую функцию или метод, добавив к ней новое поведение без изменения её исходного кода. В контексте автоматизации тестирования декораторы становятся незаменимыми помощниками: они помогают реализовать повторяющиеся кросс-функциональные задачи — повторные попытки при нестабильных тестах, замер времени выполнения, логирование входных и выходных данных, обработка исключений, а также интеграцию с фреймворками (например, pytest использует декораторы для фикстур, маркеров, параметризации).

---

## Что такое декоратор?

Декоратор — это функция, которая принимает другую функцию в качестве аргумента, добавляет к ней некоторую логику (до, после или вокруг вызова) и возвращает новую функцию (или исходную, но модифицированную). В Python декораторы применяются с использованием символа `@` перед определением функции.

**Пример:**

```python
def my_decorator(func):
    def wrapper():
        print("Что-то делаем до вызова функции")
        func()
        print("Что-то делаем после вызова функции")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# Вывод:
# Что-то делаем до вызова функции
# Hello!
# Что-то делаем после вызова функции
```

Здесь `wrapper` — это замыкание, которое запоминает переданную функцию `func`. Когда мы вызываем `say_hello()`, на самом деле вызывается `wrapper`, который в нужный момент вызывает исходную функцию.

---

## Декораторы для функций с аргументами

Если декорируемая функция принимает аргументы, нужно, чтобы `wrapper` принимал `*args` и `**kwargs` и передавал их дальше.

```python
def log_call(func):
    def wrapper(*args, **kwargs):
        print(f"Вызов {func.__name__} с аргументами {args} {kwargs}")
        result = func(*args, **kwargs)
        print(f"Результат: {result}")
        return result
    return wrapper

@log_call
def add(a, b):
    return a + b

add(3, 5)
# Вывод:
# Вызов add с аргументами (3, 5) {}
# Результат: 8
```

---

## Сохранение метаданных: `functools.wraps`

Когда мы заменяем функцию обёрткой, теряются её метаданные — имя, docstring, аннотации. Это может мешать при отладке и работе с инструментами (например, pytest использует имя функции для генерации отчётов). Исправляется это с помощью декоратора `@wraps` из модуля `functools`.

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        """Обёртка, которая ничего не делает"""
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def example():
    """Документация примера"""
    pass

print(example.__name__)   # example (без wraps было бы wrapper)
print(example.__doc__)    # Документация примера
```

**Правило:** всегда используйте `@wraps` в своих декораторах, если только нет веских причин не делать этого.

---

## Декораторы с параметрами

Иногда декоратору нужны настройки — например, количество повторных попыток или тайм-аут. Для этого мы создаём функцию, которая принимает параметры и возвращает сам декоратор.

**Общий шаблон:**

```python
def retry(retries=3, delay=1):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for attempt in range(retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == retries - 1:
                        raise
                    print(f"Попытка {attempt+1} не удалась: {e}. Повтор через {delay}с")
                    time.sleep(delay)
            return None
        return wrapper
    return decorator

@retry(retries=5, delay=2)
def unstable_operation():
    # ...
    pass
```

Обратите внимание на трёхуровневую вложенность: внешняя функция принимает параметры, средняя — декорируемую функцию, внутренняя — обёртку.

---

## Практические декораторы для автотестов

### 1. Повторные попытки (retry)

Декоратор `retry` позволяет автоматически перезапустить тест или вызов API при временных сбоях. Но на практике рекомендуется исключать перезапуск, а решать проблему падения теста.

```python
import time
from loguru import logger

def retry(max_attempts=3, delay=1, backoff=2, exceptions=(Exception,)):
    """Повторяет выполнение функции при возникновении указанных исключений.
    Args:
        max_attempts: максимальное число попыток
        delay: начальная задержка (сек)
        backoff: множитель увеличения задержки
        exceptions: кортеж исключений, которые перехватываем
    """
    def decorator(func):
        def wrapper(*args, **kwargs):
            current_delay = delay
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts:
                        logger.error(f"Последняя попытка {attempt} не удалась: {e}")
                        raise
                    logger.warning(f"Попытка {attempt} не удалась: {e}. Повтор через {current_delay:.2f}с")
                    time.sleep(current_delay)
                    current_delay *= backoff
            return None
        return wrapper
    return decorator
```

**Применение в тесте API:**

```python
@retry(max_attempts=3, exceptions=(httpx.TimeoutException, httpx.NetworkError))
def get_user_data(client, user_id):
    response = client.get(f"/users/{user_id}")
    response.raise_for_status()
    return response.json()
```

### 2. Замер времени выполнения кода

Этот декоратор помогает выявить медленные тесты, запросы, функции и другое.

```python
import time
from loguru import logger

def timing(threshold=None):
    """Замеряет время выполнения функции.
    Args:
        threshold: если время превышает порог (сек), логируем как WARNING, иначе DEBUG
    """
    def decorator(func):
        def wrapper(*args, **kwargs):
            start = time.perf_counter()
            result = func(*args, **kwargs)
            elapsed = time.perf_counter() - start
            level = "WARNING" if threshold and elapsed > threshold else "DEBUG"
            logger.log(level, f"{func.__name__} выполнен за {elapsed:.4f} сек")
            return result
        return wrapper
    return decorator

@timing(threshold=1.0)
def slow_api_call(client):
    # ...
    pass
```

### 3. Логирование входа/выхода и аргументов

Полезно для отладки вспомогательных функций.

```python
def log_io(func):
    def wrapper(*args, **kwargs):
        logger.debug(f"Вход в {func.__name__} с args={args}, kwargs={kwargs}")
        result = func(*args, **kwargs)
        logger.debug(f"Выход из {func.__name__} с результатом {result}")
        return result
    return wrapper
```

### 4. Обработка исключений с логированием

Автоматически ловит исключения, логирует их и либо пробрасывает дальше, либо возвращает значение по умолчанию.

```python
def handle_exceptions(default=None, log_level="ERROR"):
    def decorator(func):
        def wrapper(*args, **kwargs):
            try:
                return func(*args, **kwargs)
            except Exception as e:
                logger.log(log_level, f"Исключение в {func.__name__}: {e}", exc_info=True)
                return default
        return wrapper
    return decorator

@handle_exceptions(default=[])
def fetch_posts(client):
    response = client.get("/posts")
    response.raise_for_status()
    return response.json()
```

### 5. Декоратор для проверки ответа API с использованием Pydantic

Упрощает валидацию ответа в тестах.

```python
from pydantic import BaseModel, ValidationError

def validate_response(model_cls):
    def decorator(func):
        def wrapper(*args, **kwargs):
            response_data = func(*args, **kwargs)  # ожидаем, что это dict или список
            if isinstance(response_data, list):
                return [model_cls(**item) for item in response_data]
            else:
                return model_cls(**response_data)
        return wrapper
    return decorator

@validate_response(PostModel)
def get_posts(client):
    response = client.get("/posts")
    response.raise_for_status()
    return response.json()
```

---

## Комбинирование декораторов

Декораторы можно накладывать друг на друга. Порядок применения важен: верхний декоратор применяется последним (ближайший к функции — первый).

```python
@timing(threshold=0.5)
@retry(max_attempts=2)
def get_user(client, user_id):
    # ...
    pass
```

> Важно! В этом примере сначала будет применён `retry`, затем `timing`. То есть замер времени будет включать все повторные попытки.

---

## Распространенные ошибки при работе с декораторами

1. **Забывают `@wraps`** — теряются метаданные, что мешает отладке и работе инструментов (например, pytest может неправильно показывать имя теста).
2. **Неправильный порядок декораторов** — например, `@retry` должен быть ближе к функции, чем `@timing`, если вы хотите замерять общее время с учётом повторных попыток.
3. **Декоратор с параметрами без учёта параметров** — путают синтаксис: если декоратор принимает параметры, то он должен возвращать функцию-декоратор, а не использоваться как `@my_decorator` напрямую.
4. **Игнорирование `*args, **kwargs`** — если декорируемая функция принимает аргументы, а обёртка их не передаёт, будет ошибка.
5. **Мутация изменяемых объектов в декораторе** — если декоратор использует изменяемый объект (список, словарь) как значение по умолчанию, это может привести к неожиданному поведению.
6. **Перехват слишком широких исключений** — использование `except Exception` без перевыброса может скрыть ошибки, которые должны приводить к падению теста.
7. **Слишком сложные декораторы** — старайтесь не перегружать декоратор логикой, лучше выносить её в отдельные функции.

---

## Краткий итог

- **Декораторы** — мощный инструмент для добавления повторяющейся логики к функциям без изменения их кода.
- В автотестах они особенно полезны для **retry**, **замера времени**, **логирования**, **обработки исключений** и **валидации**.
- Для декораторов с параметрами применяйте трёхуровневую вложенность.
- Не забывайте о правильном порядке комбинирования декораторов.
- Изучите встроенные декораторы pytest — они составляют основу тестового фреймворка.

---

### Тестовые задания

**1. Что произойдёт, если применить декораторы в таком порядке**:  
```python
@deco1
@deco2
def test():
    pass
```  

А) `deco1` выполнится первым, затем `deco2`  
Б) `deco2` выполнится первым, затем `deco1`  
В) оба декоратора выполняются одновременно  
Г) порядок не определён  

**Правильный ответ:** Б

---

**2. Как правильно написать кастомный декоратор, который измеряет время выполнения теста и выводит его в лог?**  
А)
```python
def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(time.time() - start)
        return result
    return wrapper
```  
Б)  
```python
def timer(func):
    start = time.time()
    result = func()
    print(time.time() - start)
    return result
```  
В)
```python
def timer(func, *args):
    start = time.time()
    func(*args)
    print(time.time() - start)
```  
Г)  
```python
@timer
def test():
    ...
```  

**Правильный ответ:** А

---

**3. Декоратор `@retry` (из сторонних библиотек) в автотестах обычно используется для:**  
А) увеличения скорости выполнения теста  
Б) повторного запуска теста при его падении (flaky tests)  
В) пропуска теста по условию  
Г) изменения порядка выполнения тестов  

**Правильный ответ:** Б

---

**4. Что из перечисленного нельзя сделать с помощью декораторов в тестах?**  
А) Логировать аргументы функции  
Б) Менять функции  
В) Изменять глобальные переменные в процессе выполнения теста  
Г) Динамически создавать тестовые функции во время выполнения  

**Правильный ответ:** В

---

**5. Какой декоратор из стандартной библиотеки Python часто используется для объявления методов класса как статических (не требующих экземпляра)?**  
А) `@staticmethod`  
Б) `@classmethod`  
В) `@property`  
Г) `@abstractmethod`  

**Правильный ответ:** А

---

**6. Какое утверждение о декораторах в Python верно?**  
А) Декоратор – это функция, которая принимает другую функцию и возвращает новую функцию (или тот же объект)  
Б) Декоратор может быть применён только к функциям, но не к классам  
В) Декораторы выполняются во время вызова декорируемой функции, а не во время её определения  
Г) В одном проекте нельзя использовать более трёх декораторов на одной функции  

**Правильный ответ:** А

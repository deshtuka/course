# Вариант решения №1
```python
import httpx
from loguru import logger
from pydantic import BaseModel


class Post(BaseModel):
    id: int
    userId: int
    title: str
    body: str


def fetch_resource(resource_id: int, timeout: float | None = None) -> Post | None:
    url = f"https://jsonplaceholder.typicode.com/posts/{resource_id}"

    try:
        response = httpx.get(url, timeout=timeout)
    except httpx.RequestError as e:
        logger.warning(f"Проблема с запросом для {resource_id}: {e}")
        return None

    if response.status_code == 200:
        logger.info(f"Запись {resource_id} получена успешно")
        return Post(**response.json())

    elif response.status_code == 404:
        logger.error(f"Запись {resource_id} не найдена: {response.status_code} — {response.text}")
    else:
        logger.warning(f"Неожиданный статус {response.status_code} для записи {resource_id}: {response.text}")
    return None


if __name__ == "__main__":
    fetch_resource(resource_id=1)
    fetch_resource(resource_id=999)
    fetch_resource(resource_id=1, timeout=0.001)
```



# Варианта решения №2
```python
import httpx
from loguru import logger
from pydantic import BaseModel


class Post(BaseModel):
    userId: int
    id: int
    title: str
    body: str


def log_request(request: httpx.Request) -> None:
    logger.info(f"-> {request.method} {request.url}")


def fetch_resource(resource_id: int, timeout: float = 30.0) -> Post | None:
    url = f"https://jsonplaceholder.typicode.com/posts/{resource_id}"
    try:
        with httpx.Client(
                timeout=timeout,
                event_hooks={
                    'request': [log_request],
                }
        ) as client:
            response = client.get(url)

            if response.status_code == 200:
                logger.info(f"Успешно (resource_id = {resource_id})")
                return Post(**response.json())

            elif response.status_code == 404:
                logger.error(f"Ошибка {response.status_code} (resource_id = {resource_id}): {response.text[:200]}")
            else:
                logger.warning(f"Ошибка (resource_id = {resource_id}) {response.status_code}: {response.text[:200]}")

    except (httpx.ConnectError, httpx.TimeoutException) as e:
        logger.warning(f"Ошибка соединения (resource_id = {resource_id}): {e} ")

    except Exception as e:
        logger.exception(f"Неожиданная ошибка при получении resource_id = {resource_id}, {e}")

    finally:
        return None


if __name__ == "__main__":
    post1 = fetch_resource(resource_id=1)
    logger.info(f"post1 = {post1}")

    post2 = fetch_resource(resource_id=999)
    logger.info(f"post2 = {post2}")

    post3 = fetch_resource(resource_id=1, timeout=0.001)
    logger.info(f"post3 = {post3}")
```
## Установка и управление драйверами

### Selenium Manager (встроенный)

Начиная с **Selenium 4.6**, управление драйверами автоматизировано через **Selenium Manager** — официальный инструмент, встроенный в библиотеку. Он написан на Rust и поставляется в составе каждого релиза Selenium.

**Принцип работы**: при вызове `ChromeDriver()` Selenium Manager определяет установленную версию браузера, скачивает совместимый драйвер, кеширует его и возвращает путь. Всё происходит автоматически, без необходимости вручную указывать путь к драйверу.

**Кеш** хранится в `~/.cache/selenium/` (Linux/macOS) или `%USERPROFILE%\.cache\selenium\` (Windows).

**Настройка** через переменные окружения:
- `SE_CACHE_PATH` — путь к кешу
- `SE_MANAGER_PATH` — путь к бинарнику менеджера

### webdriver-manager (сторонняя библиотека)

До появления Selenium Manager широко использовалась библиотека **webdriver-manager** (особенно в Python). Она выполняет схожие функции: скачивание, разрешение, кеширование и переиспользование драйверов.

**Основные отличия**:
| Характеристика | Selenium Manager | webdriver-manager |
|---|---|---|
| Встроен в Selenium | Да (4.6+) | Нет (внешняя библиотека) |
| Автоопределение драйвера | Да | Да |
| Автоскачивание браузера | Да (4.16+) | Нет |
| Offline-режим | Да | Частично |
| Поддержка языков | Все биндинги Selenium | В основном Java / Python |

---

## Capabilities и опции браузера

**Capabilities** — это W3C-совместимый набор настроек, определяющих поведение браузерной сессии. Каждый браузер имеет свой класс опций:

- **Chrome** → `ChromeOptions`
- **Firefox** → `FirefoxOptions`  
- **Edge** → `EdgeOptions`
- **Safari** → `SafariOptions`

Опции позволяют управлять:
- путём к бинарнику браузера
- аргументами командной строки
- расширениями
- настройками прокси
- W3C-капабилити

**Пример (Python)**:
```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_argument("--window-size=1920,1080")
options.add_argument("--disable-gpu")
options.add_experimental_option("prefs", {"download.default_directory": "/path"})

driver = webdriver.Chrome(options=options)
```

---

## Headless-режим

Headless-режим запускает браузер без графического интерфейса — идеально для CI/CD и серверных сред.

**Chrome (рекомендуемый новый режим)**:
```python
options.add_argument("--headless=new")  # новый headless-режим (рекомендуется)
options.add_argument("--headless")       # старый режим
options.add_argument("--disable-gpu")    # часто используется вместе с headless
```

**Firefox**:
```python
from selenium.webdriver.firefox.options import Options
firefox_options = Options()
firefox_options.add_argument("-headless")
```

---

## Selenium vs Playwright: сравнение

| Критерий                 | Selenium                                         | Playwright                                       |
|--------------------------|--------------------------------------------------|--------------------------------------------------|
| **Год создания**         | 2004                                             | 2019 (Microsoft)                                 |
| **Архитектура**          | W3C WebDriver через HTTP                         | Прямое соединение с браузером через CDP          |
| **Поддерживаемые языки** | Java, Python, C#, Ruby, JS, Kotlin и др.         | JS/TS, Python, Java, .NET                        |
| **Браузеры**             | Chrome, Firefox, Edge, Safari, Internet Explorer | Chromium, Firefox, WebKit (Chrome, Edge, Safari) |
| **Автоожидания**         | Требуются явные ожидания (WebDriverWait)         | Встроенные auto-waiting                          |
| **Скорость**             | Медленнее из-за HTTP-протокола                   | Быстрее за счёт прямого соединения               |
| **Параллелизм**          | Через Selenium Grid                              | Встроенные browser contexts                      |
| **Отладка**              | Стандартные инструменты                          | Видеозапись, трассировка, inspector              |
| **Экосистема**           | Огромная, 20+ лет развития                       | Растущая, современная                            |
| **Код**                  | Более многословный                               | Более лаконичный (до −70% кода)                  |

**Пример Playwright (Python)**:
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://example.com")
    page.fill("input[name='q']", "Playwright")
    page.click("text=Search")
    assert "Results" in page.title()
    browser.close()
```

---

## Вывод

**Selenium** остаётся надёжным выбором для корпоративных проектов с широкой языковой поддержкой и устоявшейся экосистемой. Selenium Manager полностью автоматизирует управление драйверами, избавляя от рутины.

**Playwright** — современная альтернатива с более быстрым выполнением, встроенными автоожиданиями и удобным API, что делает его предпочтительным для новых проектов.

Выбор зависит от конкретных задач: Selenium — для зрелых enterprise-систем, Playwright — для современных веб-приложений и CI/CD-пайплайнов.
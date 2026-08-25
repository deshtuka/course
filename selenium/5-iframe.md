В этой статье мы сравним, как два популярных инструмента для автоматизации браузеров — **Selenium** и **Playwright** — справляются с тремя классическими задачами: переключением между окнами и вкладками, работой с iframe и обработкой всплывающих alert-окон.

Главное различие между ними лежит в самой архитектуре. Selenium использует подход **явного переключения контекста** — вы буквально перемещаете «фокус» драйвера на нужный элемент (окно, фрейм или alert) и затем работаете с ним. Playwright же работает через **объектную модель**: каждое окно/вкладка — это отдельный объект `Page`, каждый iframe — это `FrameLocator`, а alert обрабатывается через события. Это фундаментальное отличие определяет все дальнейшие сравнения.

---

## 1. Работа с окнами и вкладками

### Selenium
В Selenium для работы с окнами и вкладками используется концепция **window handles** — строковых идентификаторов. Чтобы переключиться на новое окно, нужно:

1. Получить список всех открытых окон (`driver.window_handles`).
2. Найти handle нужного окна.
3. Переключиться на него (`driver.switch_to.window(handle)`).
4. После работы — переключиться обратно.

```python
from selenium import webdriver

driver = webdriver.Chrome()
driver.get("https://example.com")

# Сохраняем handle текущего окна
main_window = driver.current_window_handle

# Кликаем по ссылке, которая открывает новое окно/вкладку
driver.find_element(By.LINK_TEXT, "Open new tab").click()

# Получаем все handles
all_windows = driver.window_handles

# Переключаемся на новое окно (последнее в списке)
new_window = all_windows[-1]
driver.switch_to.window(new_window)

# Работаем в новом окне
print(driver.title)

# Возвращаемся обратно
driver.switch_to.window(main_window)
driver.quit()
```

Недостатки подхода:
- Нужно вручную управлять списком handles.
- Сложно отследить, какое окно открылось именно от вашего действия.
- Код становится громоздким при работе с несколькими окнами.

### Playwright
В Playwright каждое окно или вкладка — это отдельный объект `Page` в рамках одного `BrowserContext`. Нет никакого `switch_to` — вы просто получаете ссылку на новый `Page` и работаете с ним напрямую.

Для перехвата нового окна используется метод `context.expect_page()`:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    context = browser.new_context()
    page = context.new_page()
    page.goto("https://example.com")

    # Ожидаем открытие новой страницы
    with context.expect_page() as new_page_info:
        page.locator("text=Open new tab").click()
    
    new_page = new_page_info.value  # Это объект Page нового окна
    print(new_page.title())
    
    # Старая страница (page) всё ещё доступна, переключаться не нужно
    print(page.title())
    
    browser.close()
```

Преимущества:
- Никакого переключения контекста — обе страницы доступны одновременно.
- Код читаемый и предсказуемый.
- Не нужно возиться со строковыми идентификаторами.

---

## 2. Работа с iframe

### Selenium
В Selenium, чтобы взаимодействовать с элементами внутри iframe, нужно **переключить контекст** драйвера на этот iframe. После этого все операции будут выполняться внутри фрейма. По окончании работы нужно вернуться в основной документ.

```python
from selenium import webdriver
from selenium.webdriver.common.by import By

driver = webdriver.Chrome()
driver.get("https://example.com")

# Находим iframe и переключаемся в него
iframe = driver.find_element(By.ID, "iframe-id")
driver.switch_to.frame(iframe)

# Теперь все действия — внутри iframe
driver.find_element(By.ID, "username").send_keys("testuser")
driver.find_element(By.XPATH, "//button[text()='Submit']").click()

# Возвращаемся в основной документ
driver.switch_to.default_content()

# Можно также вернуться на один уровень выше (в родительский фрейм)
# driver.switch_to.parent_frame()

driver.quit()
```

Проблемы:
- Нужно не забыть переключиться обратно, иначе последующие операции будут выполняться внутри iframe.
- При работе с вложенными iframe код становится запутанным.
- Если iframe динамически подгружается, нужно дополнительно обрабатывать ожидания.

### Playwright
Playwright решает эту проблему кардинально иначе — через **`frame_locator`**. Вы просто создаёте локатор для iframe и затем внутри него ищете элементы. **Никакого переключения контекста не требуется**.

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    page = browser.new_page()
    page.goto("https://example.com")

    # Создаём FrameLocator для iframe
    frame = page.frame_locator("#iframe-id")
    
    # Все действия — через frame.locator()
    frame.locator("#username").fill("testuser")
    frame.locator("button:has-text('Submit')").click()
    
    # Для работы с основным документом просто продолжаем использовать page
    page.locator("#main-element").click()
    
    browser.close()
```

Для работы с **вложенными iframe** можно цепочкой вызывать `frame_locator`:

```python
# Если внутри iframe есть ещё один iframe
inner_frame = page.frame_locator("#outer-iframe").frame_locator("#inner-iframe")
inner_frame.locator("#input-field").fill("nested value")
```

Преимущества:
- Нет переключения контекста — код остаётся линейным и понятным.
- Поддержка вложенных iframe без головной боли.
- Автоматические ожидания — Playwright сам дождётся появления iframe.

---

## 3. Обработка alert (диалоговых окон)

### Selenium
В Selenium alert — это ещё один контекст, на который нужно переключиться с помощью `driver.switch_to.alert`. После этого можно принять (`accept()`), отклонить (`dismiss()`) или ввести текст (`send_keys()`).

```python
from selenium import webdriver

driver = webdriver.Chrome()
driver.get("https://example.com")

# Кликаем по кнопке, которая вызывает alert
driver.find_element(By.ID, "alert-button").click()

# Переключаемся на alert и принимаем его
alert = driver.switch_to.alert
print(alert.text)      # Выводим текст alert
alert.accept()         # Нажимаем OK

# Или alert.dismiss() — для отмены
# Или alert.send_keys("text") — для prompt

driver.quit()
```

Важный нюанс: в Selenium alert нужно обрабатывать **после** того, как он появился. Если попытаться обработать alert до его появления — будет ошибка.

### Playwright
Playwright использует **событийную модель** для работы с диалогами. Вы подписываетесь на событие `"dialog"` **до** того, как произойдёт действие, которое вызовет alert.

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    page = browser.new_page()
    page.goto("https://example.com")

    # Определяем обработчик для диалога
    def handle_dialog(dialog):
        print(f"Alert message: {dialog.message}")
        dialog.accept()   # Нажимаем OK
        # dialog.dismiss()   # Для отмены
        # dialog.accept("text")  # Для prompt с вводом текста

    # Подписываемся на событие ДО клика
    page.on("dialog", handle_dialog)

    # Теперь кликаем — alert будет обработан автоматически
    page.locator("#alert-button").click()

    browser.close()
```

Преимущества Playwright:
- **Неблокирующая обработка** — скрипт не зависает в ожидании alert.
- Обработчик можно повесить один раз и он будет срабатывать на все диалоги.
- Не нужно переключать контекст — всё через события.
- Код чище и надёжнее.

---

## Сводная таблица сравнения

| Аспект | Selenium | Playwright |
|--------|----------|------------|
| **Окна/вкладки** | `driver.switch_to.window(handle)` с использованием window handles | Каждое окно — объект `Page`, перехват через `context.expect_page()` |
| **iframe** | `driver.switch_to.frame()` + `switch_to.default_content()` | `page.frame_locator()` без переключения контекста |
| **Alert** | `driver.switch_to.alert` после появления | `page.on("dialog")` — событийная модель до появления |
| **Переключение контекста** | Явное, ручное (switch_to) | Отсутствует — работа через объекты и локаторы |
| **Автоожидания** | Требуются явные `WebDriverWait` | Встроенные автоожидания |
| **Сложность кода** | Высокая (много switch_to) | Низкая (линейный код) |

---

## Вывод

Selenium — это зрелый и проверенный инструмент, но его подход с **явным переключением контекста** (switch_to) делает код более громоздким и подверженным ошибкам, особенно в сложных сценариях с несколькими окнами и вложенными iframe.

Playwright предлагает **более современную и интуитивную модель**:
- Окна — это объекты `Page`.
- iframe — это `FrameLocator`.
- Alert — это события.

В результате код становится короче, чище и надёжнее. Если вы начинаете новый проект или мигрируете с Selenium, Playwright определённо заслуживает внимания благодаря своей продуманной архитектуре и удобству работы с этими классическими задачами.
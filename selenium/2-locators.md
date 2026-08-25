## Обновлённый материал: Поиск элементов в Selenium и Playwright (примеры на Python)

В этой статье мы сравниваем подходы к поиску элементов в **Selenium** и **Playwright**, приводя все примеры кода на **Python**. Оба инструмента решают задачу автоматизации браузера, но их API кардинально отличаются. Рассмотрим стратегии поиска, автоматические ожидания и устойчивость тестов.

---

## 1. Поиск элементов в Selenium: класс `By`

В Selenium (Python) поиск реализован через класс `By`, содержащий статические строковые константы для разных стратегий.

### Основные стратегии `By.*`

| Стратегия | Константа | Пример использования |
|-----------|-----------|---------------------|
| По ID | `By.ID` | `driver.find_element(By.ID, "username")` |
| По атрибуту name | `By.NAME` | `driver.find_element(By.NAME, "email")` |
| По CSS-селектору | `By.CSS_SELECTOR` | `driver.find_element(By.CSS_SELECTOR, "div.container > button")` |
| По XPath | `By.XPATH` | `driver.find_element(By.XPATH, "//button[@type='submit']")` |
| По классу | `By.CLASS_NAME` | `driver.find_element(By.CLASS_NAME, "btn-primary")` |
| По тегу | `By.TAG_NAME` | `driver.find_element(By.TAG_NAME, "h1")` |
| По тексту ссылки | `By.LINK_TEXT` | `driver.find_element(By.LINK_TEXT, "Подробнее")` |
| По части текста ссылки | `By.PARTIAL_LINK_TEXT` | `driver.find_element(By.PARTIAL_LINK_TEXT, "Подро")` |

### Особенности API Selenium

```python
from selenium.webdriver.common.by import By

# Поиск одного элемента (первый подходящий)
element = driver.find_element(By.ID, "search-box")

# Поиск всех подходящих элементов (список)
elements = driver.find_elements(By.CLASS_NAME, "menu-item")
```

- `find_element` выбрасывает `NoSuchElementException`, если элемент не найден.
- `find_elements` возвращает пустой список, если ничего не найдено.
- Все поиски **мгновенны** – если элемент ещё не появился, тест упадёт. Требуются явные ожидания (см. раздел 3.2).

---

## 2. Поиск элементов в Playwright: Locator API

Playwright (Python) не использует класс `By`. Вместо этого предлагается **единый метод `locator()`** и набор **семантических методов** для более устойчивого и читаемого кода.

### Основные способы поиска в Playwright (Python)

```python
from playwright.sync_api import sync_playwright  # или async_api

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()

    # 1. CSS и XPath через единый locator
    page.locator('button.submit').click()
    page.locator('xpath=//button[@type="submit"]').click()

    # 2. Семантические локаторы (рекомендуемый подход)
    page.get_by_role('button', name='Submit').click()
    page.get_by_text('Войти').click()
    page.get_by_label('Имя пользователя').fill('admin')
    page.get_by_placeholder('Введите email').fill('user@example.com')
    page.get_by_test_id('unique-element').click()

    # 3. Комбинирование и фильтрация
    page.locator('div.card').filter(has_text='Новость').click()
```

### Семантические локаторы Playwright (Python)

| Метод | Назначение | Пример |
|-------|------------|--------|
| `get_by_role()` | Поиск по ARIA-роли | `page.get_by_role("button", name="Отправить")` |
| `get_by_text()` | Поиск по тексту | `page.get_by_text("Добро пожаловать")` |
| `get_by_label()` | Поиск по атрибуту `label` | `page.get_by_label("Пароль")` |
| `get_by_placeholder()` | Поиск по плейсхолдеру | `page.get_by_placeholder("Поиск...")` |
| `get_by_test_id()` | Поиск по `data-testid` | `page.get_by_test_id("submit-btn")` |
| `get_by_alt_text()` | Поиск изображений по `alt` | `page.get_by_alt_text("Логотип")` |

---

## 3. Ключевые отличия API поиска

### 3.1. Философия: стратегии vs. семантика

**Selenium** предлагает низкоуровневые стратегии (по ID, классу, XPath и т.д.). Вы указываете **как** искать.

**Playwright** ориентируется на **семантику** – вы описываете **что** ищете (кнопку с текстом, поле с подписью). Это делает код устойчивее к изменениям вёрстки:

```python
# Selenium: поиск по атрибуту
driver.find_element(By.ID, "submit-btn")

# Playwright: поиск по роли и тексту (семантически)
page.get_by_role("button", name="Отправить")
```

### 3.2. Автоматические ожидания (Auto-waiting)

**Selenium** требует явных ожиданий. Без `WebDriverWait` поиск мгновенно завершается ошибкой, если элемент отсутствует:

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(driver, 10)
element = wait.until(EC.presence_of_element_located((By.ID, "dynamic-content")))
```

**Playwright** имеет **встроенные автоматические ожидания**. Любое действие с локатором (`.click()`, `.fill()`, `.check()`) автоматически дожидается, пока элемент станет доступным для взаимодействия (появится в DOM, станет видимым, не перекрыт и т.д.):

```python
# Playwright — не нужно явное ожидание
page.get_by_text("Загружено").click()  # автоматически дождётся появления текста
```

Это значительно сокращает код и уменьшает количество нестабильных тестов.

### 3.3. Работа с динамическим контентом и Shadow DOM

**Selenium** не поддерживает Shadow DOM из коробки – приходится использовать `execute_script()`.

**Playwright** **из коробки** проникает сквозь Shadow DOM, работая с элементами внутри теневых корней как с обычными.

### 3.4. Цепочки и фильтрация

**Selenium** не поддерживает удобное построение цепочек – каждый новый поиск начинается с корня.

**Playwright** позволяет **фильтровать** и **уточнять** локаторы:

```python
# Найти div с классом "card", внутри которого есть текст "Новость"
page.locator('div.card').filter(has_text='Новость').click()

# Найти элемент внутри iframe
page.frame_locator('#modal-iframe').get_by_role('button', name='Закрыть').click()
```

### 3.5. Устойчивость к изменениям

**Selenium** с XPath и CSS часто "ломается" при изменении классов или структуры DOM.

**Playwright** с семантическими локаторами (`get_by_role`, `get_by_text`) более устойчив, так как опирается на **содержание** и **смысл**, а не на структуру.

---

## 4. Сравнительная таблица

| Критерий | Selenium | Playwright |
|----------|----------|------------|
| **Способ поиска** | `By.ID`, `By.NAME`, `By.CSS_SELECTOR`, `By.XPATH` и др. | `locator()`, `get_by_role()`, `get_by_text()`, `get_by_test_id()` и др. |
| **Семантические локаторы** | Отсутствуют | Есть (роль, текст, лейбл, плейсхолдер) |
| **Автоматические ожидания** | Нет (требуется `WebDriverWait`) | Есть (встроены в каждый локатор) |
| **Shadow DOM** | Требуется `execute_script()` | Поддерживается из коробки |
| **Фильтрация/цепочки** | Ограничена | Мощная (`.filter()`, `.nth()`, `.first()`, `.last()`) |
| **Читаемость кода** | Средняя (много шаблонного кода) | Высокая (семантичные методы) |
| **Объём кода для ожиданий** | Большой | Минимальный |

---

## 5. Рекомендации по выбору стратегий

### Для Selenium (Python)
- **Приоритет:** `By.ID` – самый быстрый и надёжный.
- **CSS-селекторы** предпочтительнее XPath (быстрее, читаемее).
- **XPath** используйте только когда CSS не справляется (сложный обход, поиск по тексту).
- **Всегда** применяйте `WebDriverWait` для динамического контента.

### Для Playwright (Python)
1. **Начинайте с семантических локаторов:** `get_by_role()` – самый устойчивый.
2. **Используйте `get_by_text()`** для поиска по тексту.
3. **Применяйте `get_by_test_id()`** для элементов, которые сложно найти семантически.
4. **CSS и XPath** – только как крайнее средство.
5. **Избегайте абсолютных XPath** (`/html/body/div[1]/...`) – они крайне нестабильны.

---

## Итог

Главное различие API поиска Selenium и Playwright – **философия**. Selenium даёт **низкоуровневые стратегии** и требует ручного управления ожиданиями. Playwright предлагает **семантические локаторы** и **встроенные автоматические ожидания**, что делает код чище, надёжнее и проще в поддержке.

Playwright сознательно отказался от аналога `By.*` – вместо вопроса «как найти элемент?» вы задаёте вопрос «какой элемент мне нужен?» (по его роли, тексту или тестовому идентификатору). Это современный подход, который особенно полезен в динамичных веб-приложениях.
# ActionChains в Selenium и аналоги в Playwright: сложные взаимодействия с элементами

В автоматизации тестирования веб-приложений часто требуется имитировать сложные действия пользователя: наведение курсора, перетаскивание (drag-and-drop), выполнение цепочек действий (клик, удержание, перемещение, отпускание). В Selenium для этого используется класс `ActionChains`, а в Playwright — встроенные методы и класс `Locator` с поддержкой цепочек. В этой статье мы подробно разберём оба подхода, сравним их синтаксис, возможности и особенности.

## 1. Основы ActionChains в Selenium

`ActionChains` — это API в Selenium WebDriver, позволяющее строить цепочки действий, которые затем выполняются единым блоком. Это полезно для имитации сложных пользовательских жестов, таких как:

- наведение мыши (hover)
- перетаскивание элемента
- удержание клавиш-модификаторов (Ctrl, Shift)
- последовательность кликов и перемещений

### Инициализация

```python
from selenium import webdriver
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.common.by import By

driver = webdriver.Chrome()
actions = ActionChains(driver)
```

Важно: `ActionChains` не выполняет действия немедленно — они накапливаются в очереди, и для запуска нужно вызвать метод `.perform()`.

### Наведение (hover)

```python
element = driver.find_element(By.ID, "my-element")
actions.move_to_element(element).perform()
```

Можно также переместить курсор на координаты:

```python
actions.move_by_offset(100, 50).perform()
```

### Drag and Drop (перетаскивание)

Есть два способа:

```python
source = driver.find_element(By.ID, "source")
target = driver.find_element(By.ID, "target")
actions.drag_and_drop(source, target).perform()
```

Или ручное построение:

```python
actions.click_and_hold(source).move_to_element(target).release().perform()
```

### Цепочки действий

Вы можете комбинировать любые действия:

```python
actions.move_to_element(element).click().double_click().context_click().perform()
```

Доступные методы:

- `move_to_element()`
- `click_and_hold()`
- `release()`
- `key_down()`, `key_up()`
- `pause()` (для задержки)

### Пример: сложная последовательность

```python
element1 = driver.find_element(By.ID, "el1")
element2 = driver.find_element(By.ID, "el2")

actions.click_and_hold(element1) \
       .move_to_element(element2) \
       .release() \
       .click(element2) \
       .double_click() \
       .perform()
```

## 2. Действия в Playwright

Playwright предлагает более современный и гибкий подход. Вместо отдельного класса «цепочки», каждый элемент (`Locator`) имеет методы для взаимодействия, а также можно использовать отдельный класс `Mouse` и `Keyboard` для низкоуровневого управления. Всё это работает асинхронно (но мы будем использовать синхронный API для простоты сравнения).

### Установка и инициализация

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto("https://example.com")
```

### Наведение (hover)

Просто вызываем метод `hover()` на локаторе:

```python
page.locator("#my-element").hover()
```

### Drag and Drop

Playwright имеет встроенный метод `drag_to()`:

```python
source = page.locator("#source")
target = page.locator("#target")
source.drag_to(target)
```

Также можно использовать низкоуровневый API мыши:

```python
box = source.bounding_box()
page.mouse.move(box["x"] + box["width"]/2, box["y"] + box["height"]/2)
page.mouse.down()
# ... переместить и отпустить
```

### Цепочки действий

Playwright не требует отдельного объекта для цепочки — вы можете вызвать несколько методов подряд на одном локаторе, но это не создаёт единую очередь. Для сложных последовательностей можно использовать `page.mouse` и `page.keyboard`:

```python
page.mouse.move(100, 200)
page.mouse.down()
page.mouse.move(300, 400)
page.mouse.up()
```

Или с задержкой:

```python
page.mouse.move(100, 200)
page.wait_for_timeout(500)  # ожидание
page.mouse.click(300, 400)
```

Однако Playwright предоставляет более элегантный способ — использование метода `locator.evaluate()` для выполнения пользовательских действий на стороне браузера, но это уже выходит за рамки стандартных жестов.

## 3. Сравнение подходов

| Функция | Selenium (ActionChains) | Playwright |
|---------|-------------------------|------------|
| Наведение | `.move_to_element().perform()` | `.hover()` |
| Перетаскивание | `.drag_and_drop(source, target).perform()` | `.drag_to(target)` |
| Удержание кнопки | `.click_and_hold().perform()` | `.mouse.down()` / `.mouse.up()` |
| Клавиши-модификаторы | `.key_down(Keys.CONTROL).click(element).key_up(Keys.CONTROL).perform()` | `.keyboard.down("Control")`, затем `.click()`, затем `.keyboard.up("Control")` |
| Цепочки | Все действия накапливаются в одном объекте, выполняются при `.perform()` | Разные методы на `page.mouse`/`keyboard`, либо последовательный вызов, но без автоматической группировки |
| Ожидания | Нет встроенных ожиданий между действиями (можно добавить `.pause()`) | Можно использовать `page.wait_for_timeout()` или `page.wait_for_selector()` |
| Отказоустойчивость | Действия выполняются без проверки готовности элементов (нужно явно ждать) | Встроенные автоматические ожидания (элемент будет ждать появления, видимости и т.д.) |

### Ключевые различия

1. **Синтаксис**: Selenium требует явного вызова `.perform()`, Playwright выполняет действия немедленно (но с автоматическими ожиданиями).
2. **Гибкость**: Playwright предлагает как высокоуровневые методы (`drag_to`), так и низкоуровневый доступ к мыши/клавиатуре. Selenium тоже имеет низкоуровневые методы, но они менее интуитивны.
3. **Автоматические ожидания**: В Playwright большинство действий автоматически ждут, пока элемент станет доступным. В Selenium нужно использовать явные ожидания (`WebDriverWait`), иначе могут возникать ошибки.
4. **Производительность**: Playwright работает быстрее благодаря использованию протокола Chrome DevTools Protocol (CDP) и отсутствию дополнительной прослойки WebDriver.

## 4. Расширенные примеры

### Пример 1: Перетаскивание с задержкой

**Selenium**:

```python
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

source = WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.ID, "source")))
target = WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.ID, "target")))

actions = ActionChains(driver)
actions.click_and_hold(source).pause(1).move_to_element(target).release().perform()
```

**Playwright**:

```python
source = page.locator("#source")
target = page.locator("#target")
source.hover()
page.mouse.down()
page.wait_for_timeout(1000)  # задержка 1 секунда
target.hover()
page.mouse.up()
```

Или с использованием `drag_to` с опцией `force=True`, если нужно игнорировать видимость.

### Пример 2: Наведение и выпадающее меню

**Selenium**:

```python
menu = driver.find_element(By.ID, "menu")
submenu = driver.find_element(By.ID, "submenu")
ActionChains(driver).move_to_element(menu).move_to_element(submenu).click().perform()
```

**Playwright**:

```python
menu = page.locator("#menu")
submenu = page.locator("#submenu")
menu.hover()
submenu.hover()  # или сразу клик
submenu.click()
```

### Пример 3: Сочетание клавиш

**Selenium**:

```python
from selenium.webdriver.common.keys import Keys

element = driver.find_element(By.ID, "input")
actions = ActionChains(driver)
actions.key_down(Keys.CONTROL).click(element).key_up(Keys.CONTROL).perform()
```

**Playwright**:

```python
element = page.locator("#input")
page.keyboard.down("Control")
element.click()
page.keyboard.up("Control")
```

## 5. Когда что использовать?

- **Selenium** — классический инструмент, широко распространён, много документации. Подходит для проектов, где уже используется Selenium, или при необходимости поддержки старых браузеров.
- **Playwright** — современный фреймворк с удобным API, автоматическими ожиданиями, лучшей производительностью и поддержкой множества браузеров (Chromium, Firefox, WebKit). Рекомендуется для новых проектов.

В плане сложных действий Playwright предоставляет более интуитивные методы, а также встроенные механизмы ожидания, что упрощает написание стабильных тестов.

## 6. Заключение

И Selenium, и Playwright позволяют эмулировать сложные пользовательские взаимодействия, но подходы различаются. Selenium использует паттерн «цепочка действий» с явным запуском, а Playwright предлагает как высокоуровневые методы (например, `.drag_to()`), так и низкоуровневый контроль через `mouse` и `keyboard`. Playwright также выигрывает за счёт автоматических ожиданий и более современного API. Выбор зависит от контекста проекта, но для новых автоматизаций Playwright выглядит предпочтительнее.

---

Материал подготовлен для практикующих автоматизаторов. Экспериментируйте с примерами, адаптируйте под свои задачи и не забывайте про отладку!
---
{"dg-publish":true,"permalink":"/interview/interview-preparations/4-mro-dunder-methods-context-managers/"}
---

### 1. MRO и C3-линеаризация

MRO (Method Resolution Order) определяет порядок обхода классов при поиске атрибутов и методов. CPython использует алгоритм C3-линеаризации, который гарантирует сохранение локального порядка предков: класс-потомок всегда проверяется раньше родителей, а порядок указания базовых классов в объявлении сохраняется.

Функция `super()` возвращает не объект родительского класса, а прокси-объект. Он ищет методы в MRO текущего экземпляра (`self`), начиная со следующего класса за тем, внутри которого вызван `super()`.

```python
class Base:
    def process(self):
        print("Base")

class A(Base):
    def process(self):
        print("A start")
        super().process()
        print("A end")

class B(Base):
    def process(self):
        print("B start")
        super().process()
        print("B end")

class C(A, B):
    def process(self):
        print("C start")
        super().process()
        print("C end")

# MRO для C: C -> A -> B -> Base -> object
C().process()
```

#### Задача 1. Конфликт C3-линеаризации

**Условие:** Почему следующий код упадет на этапе ком компиляции класса `C`? Как изменить иерархию, не удаляя наследование?

```python
class A: pass
class B(A): pass
class C(A, B): pass
```

**Разбор:**
Для класса `C` указан порядок `(A, B)`. Локальный порядок предков требует проверять `A` раньше `B`. Однако класс `B` наследуется от `A`, поэтому MRO класса `B` требует проверять `B` раньше `A`. C3-линеаризация обнаруживает, что `A` находится в хвосте MRO класса `B`, и блокирует создание класса `C` с ошибкой `TypeError`.

**Решение:** Поменять порядок базовых классов согласно MRO родителя: `class C(B, A): pass`.

#### Задача 2. Порядок выполнения в кооперативном наследовании

**Условие:** Опишите последовательность вывода в консоль для приведенного выше примера `C().process()`.

**Разбор:**

1. `C.process()` печатает `"C start"` и вызывает `super().process()`. По MRO класса `C` следующим идет `A`.
2. `A.process()` печатает `"A start"` и вызывает `super().process()`. В контексте MRO объекта `C` следующим после `A` идет **`B`**, а не `Base`.
3. `B.process()` печатает `"B start"` и вызывает `super().process()`. Следующий в MRO — `Base`.
4. `Base.process()` печатает `"Base"`.
5. Стек вызовов разворачивается: печатаются `"B end"`, `"A end"`, `"C end"`.

### 2. Жизненный цикл объекта: `__new__`, `__init__` и `__del__`

Создание объекта в CPython разделено на выделение памяти и инициализацию состояния:

- `__new__(cls, *args, **kwargs)` — статический метод (реализованный на уровне C API), отвечающий за аллокацию памяти. Он принимает класс первым аргументом и обязан вернуть новый экземпляр.
    
- `__init__(self, *args, **kwargs)` — метод инициализации. Вызывается CPython **только если** `__new__` вернул экземпляр класса `cls` или его подкласса. Если `__new__` возвращает объект другого типа, `__init__` текущего класса не вызывается.
    
- `__del__(self)` — финализатор. Вызывается CPython в момент, когда счетчик ссылок `ob_refcnt` объекта становится равным нулю.


```python
import threading

class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        if not cls._instance:
            with cls._lock:
                if not cls._instance:
                    cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, value: int):
        # Внимание: __init__ будет вызываться ПРИ КАЖДОМ обращении ThreadSafeSingleton(val)
        self.value = value
```

#### Задача 1. Перехват управления в `__new__`

**Условие:** Что выведет код и почему?

```python
class PluginA:
    def __init__(self):
        print("Init A")

class PluginB:
    def __init__(self):
        print("Init B")

class Router:
    def __new__(cls, mode: str):
        if mode == "A":
            return PluginA()
        return PluginB()

    def __init__(self, mode: str):
        print("Init Router")

r = Router("A")
```

**Разбор:**
Код выведет только `"Init A"`. Метод `Router.__new__` возвращает экземпляр `PluginA`. Так как `isinstance(PluginA(), Router)` возвращает `False`, CPython игнорирует `Router.__init__`. Вызов `PluginA()` внутри `__new__` самостоятельно исполняет `PluginA.__init__`.

#### Edge Cases / Trade-offs

- **Повторная инициализация Синглтона:** При реализации Синглтона через `__new__`, метод `__init__` исполняется при каждом вызове `Singleton()`. Чтобы избежать перезаписи полей, в классе нужно завести флаг инициализации (например, `_initialized = True`).
    
- **Недетерминированность `__del__`:** `__del__` не гарантирует моментального выполнения при аварийном завершении процесса или при наличии циклических ссылок, обрабатываемых сборщиком мусора (GC). Использовать `__del__` для закрытия сетевых сокетов или файлов запрещено.

### 3. Контекстные менеджеры

Контекстный менеджер гарантирует детерминированное выделение и освобождение ресурсов через оператор `with`.

Синхронный протокол требует реализации двух методов:

- `__enter__(self)` — выделяет ресурс и возвращает объект, связываемый с переменной после `as`.
- `__exit__(self, exc_type, exc_val, exc_tb)` — выполняется при выходе из блока `with`. Принимает тип исключения, объект исключения и traceback. Если внутри блока `with` исключений не возникло, аргументы равны `None`.

Если `__exit__` возвращает `True`, CPython **подавляет** возникшее исключение. Если возвращается `False` или `None`, исключение штатно пробрасывается выше по стеку.

Асинхронный протокол `async with` использует методы `__aenter__` и `__aexit__`, возвращающие корутины.

```python
import asyncio
import logging

class AsyncTransaction:
    def __init__(self, db_client):
        self.db = db_client

    async def __aenter__(self):
        await self.db.execute("BEGIN")
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            await self.db.execute("ROLLBACK")
            logging.error(f"Transaction aborted due to: {exc_val}")
            return False  # Пробрасываем исключение дальше
        await self.db.execute("COMMIT")
        return True
```

#### Задача 1. Атомарный менеджер состояния словаря

**Условие:** Напишите контекстный менеджер `atomic_config`, который принимает словарь конфигурации и модифицирует его. Если внутри блока `with` происходит ошибка, все изменения словаря должны полностью откатываться.

```python
from copy import deepcopy
from typing import Dict, Any

class atomic_config:
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self._snapshot = {}

    def __enter__(self) -> Dict[str, Any]:
        self._snapshot = deepcopy(self.config)
        return self.config

    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        if exc_type is not None:
            self.config.clear()
            self.config.update(self._snapshot)
        return False  # Ошибка должна быть обработана вызывающим кодом
```

#### Edge Cases / Trade-offs

- **Скрытые баги при `return True`:** Случайный возврат `True` из `__exit__` приводит к «глотанию» критических ошибок (включая `NameError` или `TypeError`), усложняя отладку.
    
- **Ошибки в `__enter__`:** Если исключение происходит внутри метода `__enter__`, метод `__exit__` **не вызывается**. Ресурс, выделенный до момента сбоя внутри `__enter__`, должен подчищаться через `try...except` прямо внутри `__enter__`.

#### Interview Tip

Проверяется знание декоратора `@contextlib.contextmanager`. Важно понимать, что он превращает генератор в контекстный менеджер: код до `yield` выполняется в `__enter__`, сам `yield` возвращает значение, а блок `finally` после `yield` выполняется в `__exit__`.

### 4. Сводная таблица dunder-методов CPython

| **Категория**              | **Метод**                      | **Назначение и особенности CPython**                                           |
| -------------------------- | ------------------------------ | ------------------------------------------------------------------------------ |
| **Жизненный цикл**         | `__new__(cls, ...)`            | Выделяет память под объект. Возвращает новый экземпляр.                        |
|                            | `__init__(self, ...)`          | Инициализирует экземпляр после создания.                                       |
|                            | `__del__(self)`                | Вызывается при обнулении счетчика ссылок (`ob_refcnt == 0`).                   |
| **Представление**          | `__repr__(self)`               | Однозначное техническое представление объекта (для разработчиков/REPL).        |
|                            | `__str__(self)`                | Человекочитаемое представление объекта (`str()`, `print()`).                   |
|                            | `__format__(self, spec)`       | Обработка форматирования в f-строках и `format()`.                             |
| **Доступ к атрибутам**     | `__getattr__(self, name)`      | Вызывается **только** если атрибут не найден в `__dict__` или MRO.             |
|                            | `__getattribute__(self, name)` | Вызывается **безусловно** при каждом обращении к любому атрибуту.              |
|                            | `__setattr__(self, name, val)` | Перехватывает запись атрибута. Требует аккуратности во избежание рекурсии.     |
|                            | `__delattr__(self, name)`      | Перехватывает удаление атрибута (`del obj.attr`).                              |
| **Контейнеры / Sequences** | `__len__(self)`                | Возвращает длину. Вызывается функцией `len()`. Обязан возвращать `int >= 0`.   |
|                            | `__getitem__(self, key)`       | Доступ по индексу/ключу/срезу (`obj[key]`).                                    |
|                            | `__setitem__(self, key, val)`  | Запись по индексу/ключу (`obj[key] = val`).                                    |
|                            | `__contains__(self, item)`     | Проверка вхождения (`item in obj`). При отсутствии CPython итерирует объект.   |
| **Итерация**               | `__iter__(self)`               | Возвращает объект-итератор (реализующий `__next__`).                           |
|                            | `__next__(self)`               | Возвращает следующий элемент или выбрасывает `StopIteration`.                  |
| **Сравнение / Хэш**        | `__eq__(self, other)`          | Проверка равенства (`==`). При сбросе сбрасывает `__hash__` в `None`.          |
|                            | `__hash__(self)`               | Вычисляет хэш для `dict`/`set`. Обязан быть неизменным весь жизненный цикл.    |
|                            | `__lt__`, `__le__`, ...        | Методы сравнения (`<`, `<=`). Используются `functools.total_ordering`.         |
| **Вызов и Контекст**       | `__call__(self, ...)`          | Позволяет вызывать экземпляр класса как функцию (`obj()`).                     |
|                            | `__enter__` / `__exit__`       | Протокол синхронного контекстного менеджера.                                   |
|                            | `__aenter__` / `__aexit__`     | Протокол асинхронного контекстного менеджера.                                  |
|                            | `__await__(self)`              | Возвращает итератор для интеграции объекта с оператором `await`.               |
| **Приведение типов**       | `__bool__(self)`               | Возвращает булево значение. Если не реализован, CPython проверяет `__len__()`. |
|                            | `__index__(self)`              | Преобразует объект в `int` для использования в срезах (`list[obj]`).           |

### 5. Оптимизация памяти объектов: `__slots__`

Стандартный экземпляр класса CPython хранит динамические атрибуты в словаре `self.__dict__`. Динамический словарь требует от 100 до 150+ байт на объект только под структуру хэш-таблицы.

Инструкция `__slots__` отключает создание `self.__dict__` и `self.__weakref__` (если они не указаны явно), заменяя их массивом указателей фиксированной длины напрямую в C-структуре `PyObject`. Имена атрибутов регистрируются на уровне класса в виде дескрипторов доступа (member descriptors).

```python
import sys

class UserDefault:
    def __init__(self, user_id: int, email: str):
        self.user_id = user_id
        self.email = email

class UserSlotted:
    __slots__ = ('user_id', 'email')
    def __init__(self, user_id: int, email: str):
        self.user_id = user_id
        self.email = email

u1 = UserDefault(1, "test@example.com")
u2 = UserSlotted(1, "test@example.com")

# Размер u1 складвается из размера структуры + размера его __dict__
print(sys.getsizeof(u1) + sys.getsizeof(u1.__dict__))  # ~ 150-200 байт
print(sys.getsizeof(u2))                               # ~ 48-56 байт
```

#### Задача 1. Наследование и динамические атрибуты в slotted-классах

**Условие:** Почему следующий код упадет с ошибкой? Как разрешить динамическое добавление полей, сохранив базовую экономию памяти?

```python
class BaseNode:
    __slots__ = ('id',)

class CustomNode(BaseNode):
    __slots__ = ('payload',)

node = CustomNode()
node.id = 1
node.payload = {"data": 123}
node.meta = "extra"
```

**Разбор:**
Класс `CustomNode` явно объявляет `__slots__`, поэтому CPython не выделяет `__dict__` для его экземпляров. При попытке присвоения `node.meta = "extra"` выбрасывается `AttributeError: 'CustomNode' object has no attribute 'meta'`.

**Решение:** Чтобы вернуть возможность динамического создания полей для отдельных атрибутов, необходимо явно добавить `'__dict__'` в `__slots__` дочернего класса:

`__slots__ = ('payload', '__dict__')`.

#### Edge Cases / Trade-offs

- **Множественное наследование:** Если два базовых класса имеют непустой `__slots__`, дочерний класс упадет с `TypeError: multiple bases have instance lay-out conflict`. Множественное наследование разрешено только если максимум один базовый класс имеет непустые слоты.
    
- **Слабые ссылки:** Помещение объекта в `weakref` невозможно, если `'__weakref__'` не объявлен явно в `__slots__`.

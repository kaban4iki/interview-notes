---
{"dg-publish":true,"permalink":"/interview/practical-tasks/advanced-python/"}
---

### 1. Singleton через метакласс

**Задача:** Реализовать потокобезопасный (thread-safe) паттерн Singleton с использованием метаклассов.

```python
import threading
from typing import Any

class SingletonMeta(type):
    _instances: dict[type, Any] = {}
    _lock: threading.Lock = threading.Lock()

    def __call__(cls, *args: Any, **kwargs: Any) -> Any:
        with cls._lock:
            if cls not in cls._instances:
                instance = super().__call__(*args, **kwargs)
                cls._instances[cls] = instance
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    pass
```

### 2. Декоратор Retry с экспоненциальной задержкой

**Задача:** Написать декоратор для асинхронных функций, который делает $N$ попыток перезапуска при возникновении исключения с увеличивающимся интервалом.

```python
import asyncio
from functools import wraps
from typing import Callable, Any, Coroutine

def retry(retries: int, base_delay: float = 0.1) -> Callable:
    def decorator(func: Callable[..., Coroutine[Any, Any, Any]]) -> Callable:
        @wraps(func)
        async def wrapper(*args: Any, **kwargs: Any) -> Any:
            last_exc: Exception = Exception("Unknown error")
            for i in range(retries):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    last_exc = e
                    await asyncio.sleep(base_delay * (2 ** i))
            raise last_exc
        return wrapper
    return decorator
```

### 3. Декоратор для профилирования CPU и RAM

**Задача:** Создать контекстный менеджер или декоратор, который замеряет время выполнения и дельту потребления памяти.


```python
import time
import tracemalloc
from contextlib import contextmanager
from typing import Generator

@contextmanager
def profile_resource() -> Generator[None, None, None]:
    tracemalloc.start()
    start_time = time.perf_counter()
    try:
        yield
    finally:
        current, peak = tracemalloc.get_traced_memory()
        elapsed = time.perf_counter() - start_time
        print(f"Time: {elapsed:.4f}s, Peak RAM: {peak / 10**6:.2f}MB")
        tracemalloc.stop()
```

### 4. Registry паттерн для плагинов

**Задача:** Реализовать систему автоматической регистрации классов-обработчиков при их наследовании от базового класса.

```python
class HandlerRegistry(type):
    _registry: dict[str, type] = {}

    def __init__(cls, name: str, bases: tuple, attrs: dict[str, Any]) -> None:
        super().__init__(name, bases, attrs)
        if name != "BaseHandler":
            cls._registry[attrs.get("cmd", name.lower())] = cls

    @classmethod
    def get_handler(mcs, cmd: str) -> type:
        return mcs._registry[cmd]

class BaseHandler(metaclass=HandlerRegistry):
    pass

class AuthHandler(BaseHandler):
    cmd = "auth"
```

### 5. Type-safe Dataclass клон

**Задача:** Написать простую версию dataclass, которая при инициализации проверяет соответствие типов переданных аргументов аннотациям.

```python
class StrictModel:
    def __init__(self, **kwargs: Any) -> None:
        hints = self.__annotations__
        for key, value in kwargs.items():
            if key in hints and not isinstance(value, hints[key]):
                raise TypeError(f"Expected {hints[key]} for {key}, got {type(value)}")
            setattr(self, key, value)

class User(StrictModel):
    id: int
    name: str
```

### 6. Асинхронный контекстный менеджер для Lock

**Задача:** Реализовать обертку над `asyncio.Lock`, которая логирует время ожидания захвата блокировки.

```python
import asyncio
import time

class LoggedLock:
    def __init__(self) -> None:
        self._lock = asyncio.Lock()

    async def __aenter__(self) -> None:
        start = time.monotonic()
        await self._lock.acquire()
        print(f"Lock acquired in {time.monotonic() - start:.4f}s")

    async def __aexit__(self, exc_type: Any, exc_val: Any, exc_tb: Any) -> None:
        self._lock.release()
```

### 7. Descriptor для валидации данных

**Задача:** Создать дескриптор `PositiveInt`, который запрещает устанавливать отрицательные значения в атрибуты класса.

```python
class PositiveInt:
    def __init__(self) -> None:
        self._name: str = ""

    def __set_name__(self, owner: type, name: str) -> None:
        self._name = name

    def __set__(self, instance: Any, value: int) -> None:
        if value < 0:
            raise ValueError(f"{self._name} must be positive")
        instance.__dict__[self._name] = value

class Inventory:
    count = PositiveInt()
```

### 8. LRU Cache на OrderedDict

**Задача:** Реализовать LRU-кеш вручную (без `functools`), ограничивающий количество записей.

```python
import functools
from typing import Hashable, Any, TypeVar
from collections import OrderedDict

Key = Hashable
Value = TypeVar('Value')


class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache: dict[Key, Value] = OrderedDict()
    
    def get(self, key: Key) -> Value:
        if key in self.cache:
            self.cache.move_to_end(key)
            return self.cache[key]
        raise KeyError
    
    def put(self, key: Key, value: Value):
        if len(self.cache) >= self.capacity:
            self.cache.popitem(last=False)
        
        if key not in self.cache:
            self.cache[key] = value
        
        self.cache.move_to_end(key)
    

def lru_cache(capacity: int):
    def decorator(func):
        cache = LRUCache(capacity=capacity)
        
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            key = (*args, *sorted(tuple(kwargs.items())))
            try:
                return cache.get(key)
            except KeyError:
                value = func(*args, **kwargs)
                cache.put(key, value)
                return value
        return wrapper
    return decorator
```

### 9. Декоратор с доступом к внутреннему состоянию

**Задача:** Создать декоратор-счетчик вызовов, который позволяет обнулить счетчик извне (через атрибут функции).

```python
import functools


def counter(func):
    cnt = 0
    
    def clear():
        nonlocal cnt
        cnt = 0
    
    func.clear_cnt = clear
    
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        nonlocal cnt
        cnt += 1
        print(cnt)
        return func(*args, **kwargs)
    return wrapper
```

### 10. Cвой range

**Задача:** Реализовать класс `RangeLike`, работающий как `range`.

```python
class RangeLike:
	def __init__(self, start: int, stop: int):
		self.start = start
		self.stop = stop
		self.current = start
		
	def __iter__(self):
		self.current = start
		return self
	
	def __next__(self):
		if self.current < self.stop:
			value = self.current
			self.current += 1
			return value
		raise StopIteration
```
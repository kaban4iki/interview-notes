---
{"dg-publish":true,"permalink":"/interview/practical-tasks/asyncio-and-concurrency/"}
---

### Ограничение параллелизма (Semaphore)

**Задача:** Реализовать `fetch_all(tasks, limit)`, которая выполняет корутины с ограничением количества одновременно запущенных задач.

```python
import asyncio
from typing import Any, Coroutine

async def fetch_all(tasks: list[Coroutine], limit: int) -> list[Any]:
    semaphore = asyncio.Semaphore(limit)

    async def sem_task(task: Coroutine) -> Any:
        async with semaphore:
            return await task

    return await asyncio.gather(*(sem_task(t) for t in tasks))
```

**Вопросы и ответы:**

- **В: Почему Semaphore лучше, чем разбивка списка на чанки (chunks)?**
    
    - **О:** Чанки заставляют ждать завершения самой медленной задачи в группе перед запуском следующей. Семафор же запускает новую задачу сразу, как только освобождается «слот», обеспечивая максимальную утилизацию ресурсов.
        
- **В: Что будет, если одна задача упадет с ошибкой?**
    
    - **О:** `asyncio.gather` по умолчанию немедленно пробросит исключение выше, но остальные задачи продолжат выполняться (хотя результат их будет трудно получить). Чтобы собрать все результаты (включая ошибки), нужно передать `return_exceptions=True`.

### Batch-процессор с накоплением

**Задача:** Класс, который собирает объекты в буфер и обрабатывает их пачкой при достижении размера $N$ или по истечении времени $T$.

```python
import asyncio
from typing import Any

class BatchProcessor:
    def __init__(self, batch_size: int, timeout: float) -> None:
        self.batch_size = batch_size
        self.timeout = timeout
        self.queue: list[Any] = []
        self._lock = asyncio.Lock()
        self._event = asyncio.Event()

    async def add(self, item: Any) -> None:
        async with self._lock:
            self.queue.append(item)
            if len(self.queue) >= self.batch_size:
                self._event.set()

    async def run(self) -> None:
        while True:
            try:
                await asyncio.wait_for(self._event.wait(), timeout=self.timeout)
            except asyncio.TimeoutError:
                pass
            
            async with self._lock:
                if self.queue:
                    batch, self.queue = self.queue, []
                    await self._process_batch(batch)
                self._event.clear()

    async def _process_batch(self, batch: list[Any]) -> None:
        # Логика сохранения в БД или отправки в API
        pass
```

**Вопросы и ответы:**

- **В: В чем риск использования `asyncio.Lock` в методе `add` при очень высокой нагрузке?**
    
    - **О:** При огромном количестве конкурентных вызовов `add` возникнет очередь на захват лока, что может замедлить event loop. Если порядок не критичен, лучше использовать `asyncio.Queue`, которая внутри оптимизирована для таких сценариев.
        
- **В: Как реализовать Graceful Shutdown?**
    
    - **О:** Нужно перехватить сигнал остановки (например, `CancelledError`), выйти из цикла `while True` и в блоке `finally` проверить `if self.queue:`, чтобы обработать последние "остатки" данных перед завершением приложения.

### Гонка задач (Race Pattern)

**Задача:** Запустить несколько запросов и вернуть результат того, кто ответил первым, отменив остальные.

```python
import asyncio
from typing import Any

async def get_fastest(coros: list[Coroutine]) -> Any:
    tasks = [asyncio.create_task(c) for c in coros]
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )
    
    for task in pending:
        task.cancel()
    
    # Ждем завершения отмены, чтобы избежать предупреждений
    if pending:
        await asyncio.gather(*pending, return_exceptions=True)
    
    return await done.pop()
```

**Вопросы и ответы:**

- **В: Зачем явно вызывать `task.cancel()`?**
    
    - **О:** Если их не отменить, они продолжат висеть в event loop, потребляя CPU, память и удерживая сетевые соединения, даже если их результат нам больше не нужен.
        
- **В: Как вернуть первый УСПЕШНЫЙ результат, игнорируя ошибки?**
    
    - **О:** Вместо `asyncio.wait` лучше использовать цикл по `asyncio.as_completed(tasks)`. Мы перехватываем исключение в `next(iter)`, и если оно возникло — переходим к следующей задаче, пока не найдем успешную или пока список не иссякнет.
        

---

### Синхронизация через Barrier

**Задача:** Создать точку синхронизации, где $N$ воркеров ждут друг друга.

```python
import asyncio

class Barrier:
    def __init__(self, n: int) -> None:
        self.n = n
        self.waiting = 0
        self.cond = asyncio.Condition()

    async def wait(self) -> None:
        async with self.cond:
            self.waiting += 1
            if self.waiting == self.n:
                self.cond.notify_all()
                self.waiting = 0
            else:
                await self.cond.wait()
```

**Вопросы и ответы:**

- **В: В чем преимущество `Condition` перед `Event` здесь?**
    
    - **О:** `Condition` позволяет атомарно проверить состояние под локом и уснуть. `Event` не умеет считать "сколько человек его ждут" без внешнего счетчика, что может привести к race condition при инкременте `self.waiting`.
        
- **В: Что делать, если один воркер завис?**
    
    - **О:** Нужно использовать `asyncio.wait_for(barrier.wait(), timeout=X)`. Но важно помнить: если один воркер отвалился по таймауту, Barrier переходит в "сломанное" состояние, и логика перезагрузки барьера должна быть предусмотрена.
        

---

### Safe Fire-and-Forget (Background Tasks)

**Задача:** Запустить фоновую задачу так, чтобы она не была удалена GC и логировала свои ошибки.

```python
import asyncio
import logging
from typing import Coroutine, Any

_background_tasks: set[asyncio.Task] = set()

def run_background(coro: Coroutine[Any, Any, Any]) -> None:
    task = asyncio.create_task(coro)
    _background_tasks.add(task)
    
    def _handle_result(t: asyncio.Task) -> None:
        _background_tasks.discard(t)
        try:
            t.result() # Проверка на наличие исключения
        except asyncio.CancelledError:
            pass
        except Exception:
            logging.exception("Background task failed")

    task.add_done_callback(_handle_result)
```

**Вопросы и ответы:**

- **В: Зачем нужен глобальный сет `_background_tasks`?**
    
    - **О:** Event Loop хранит только слабые ссылки (weak references) на задачи. Если задачу никто не "авейтит" и на нее нет сильной ссылки, Python может удалить ее в процессе работы через Garbage Collector, даже если она не завершилась.
        
- **В: Почему `add_done_callback`, а не `await`?**
    
    - **О:** Потому что это Fire-and-Forget. Функция `run_background` должна завершиться мгновенно, не блокируя вызывающий поток, а callback сработает сам в будущем "бесплатно" для текущего контекста.

---

### Async Batch Downloader

**Задача:** Написать функцию, принимающую urls: List\[str] и max_concurrent: int. Использовать aiohttp и asyncio.Semaphore. Ограничить количество одновременных запросов значением max_concurrent. Ошибки сети или HTTP-статусы 4xx/5xx для одного URL не должны прерывать скачивание остальных. Функция должна возвращать словарь со статусами и результатами.

```python
import asyncio
from typing import Any
import aiohttp
from collections import defaultdict

async def load(
    url: str, 
    session: aiohttp.ClientSession, 
    semaphore: asyncio.Semaphore
) -> tuple[int | str, tuple[str, Any]]:
    # Семафор защищает только конкретный сетевой запрос, не плодя лишние сессии
    async with semaphore:
        try:
            # Правильный контекстный менеджер для запроса
            async with session.get(
	            url, 
	            timeout=aiohttp.ClientTimeout(total=10)
			) as response:
                body = await response.text()
                return response.status, (url, body)
        except aiohttp.ClientError as e:
            # Отлавливаем именно сетевые ошибки aiohttp, а не вообще всё
            return "NETWORK_ERROR", (url, str(e))
        except asyncio.TimeoutError:
            return "TIMEOUT", (url, None)
        except Exception as e:
            # Защита от непредвиденных исключений
            return "UNKNOWN_ERROR", (url, str(e))

async def batch_downloader(
	urls: list[str], 
	max_concurrent: int
) -> dict[int | str, list[tuple[str, Any]]]:
    semaphore = asyncio.Semaphore(max_concurrent)
    result_dict = defaultdict(list)
    
    # Сессия создается ОДИН раз для всех запросов.
    async with aiohttp.ClientSession() as session:
        # Создаем генератор задач
        tasks = [load(url, session, semaphore) for url in urls]
        # Запускаем конкурентно
        results = await asyncio.gather(*tasks)
        
    for status, res in results:
        result_dict[status].append(res)
        
    return result_dict
```

**Вопросы и ответы:**

- **В: Почему сессия создаётся во внешней функции и передаётся в `load`?**
    
    - **О:** Смысл `ClientSession` в `aiohttp` — это создание пула соединений (Connection Pooling) и переиспользование TCP-сообщений (HTTP Keep-Alive). В случае создания сессии в `load` для каждого URL будет происходить полный цикл: DNS-резолвинг, TCP-handshake, TLS-handshake, отправка запроса и закрытие соединения.

---

### Thread-safe KV Storage

**Задача:** Реализовать потокобезопасный класс `ThreadSafeKV` с методами `set(key, value, ttl_sec=None)`, `get(key)` и `delete(key)`. Использовать `threading.Lock` или `threading.RLock`. Протухшие по TTL ключи не должны возвращаться. Реализовать очистку памяти: либо лениво при вызове get, либо через фоновый демон-поток с периодическим запуском.

```python
import threading
import time
from typing import Any, NamedTuple

class CacheEntry(NamedTuple):
    value: Any
    expires_at: float | None  # Абсолютное время в секундах epoch epoch


class ThreadSafeKV:
    def __init__(self, clean_interval_sec: float = 60.0):
        # Используем обычный Lock, так как рекурсивный захват в рамках одного метода не требуется
        self._lock = threading.Lock()
        self._storage: dict[str, CacheEntry] = {}
        
        # Настройка и запуск фонового потока для очистки протухших ключей
        self._clean_interval = clean_interval_sec
        self._cleaner_thread = threading.Thread(
            target=self._bg_cleaner, 
            daemon=True, 
            name="ThreadSafeKV-Cleaner"
        )
        self._cleaner_thread.start()

    def set(self, key: str, value: Any, ttl_sec: float | None = None) -> None:
        expires_at = time.time() + ttl_sec if ttl_sec is not None else None
        entry = CacheEntry(value=value, expires_at=expires_at)
        
        with self._lock:
            self._storage[key] = entry

    def get(self, key: str) -> Any | None:
        with self._lock:
            entry = self._storage.get(key)
            if entry is None:
                return None
            
            # Ленивая проверка TTL
            if entry.expires_at is not None and time.time() > entry.expires_at:
                del self._storage[key]  # Удаляем протухший элемент
                return None
                
            return entry.value

    def delete(self, key: str) -> bool:
        with self._lock:
            if key in self._storage:
                del self._storage[key]
                return True
            return False

    def _bg_cleaner(self) -> None:
        """Периодическая полная очистка хранилища от протухших ключей."""
        while True:
            time.sleep(self._clean_interval)
            now = time.time()
            
            # Собираем ключи на удаление во избежание RuntimeError: dictionary changed size during iteration
            keys_to_delete = []
            
            with self._lock:
                for key, entry in self._storage.items():
                    if entry.expires_at is not None and now > entry.expires_at:
                        keys_to_delete.append(key)
                
                for key in keys_to_delete:
                    self._storage.pop(key, None)
```

**Вопросы и ответы:**

- **В: Если вся эта операция (и сбор, и удаление) происходит внутри одного и того же непрерывного вызова контекстного менеджера `with self._lock`, может ли другой поток вызвать `delete()` или `set()` в этот момент? Действительно ли там необходима повторная проверка/безопасный `pop`, или здесь есть избыточность синхронизации?**
    
    - **О:** В текущей реализации ситуация, когда ключ удалили _во время_ работы этого блока, физически невозможна. Повторная проверка избыточна. Держать блокировку на протяжении всего обхода большого словаря — плохая практика (длинный Stop-the-World). Правильный шаг — собрать ключи под короткой блокировкой, отпустить её, а затем удалять элементы по одному, снова кратковременно захватывая мьютекс. Вот при такой оптимизации безопасный `pop(key, None)` становится критически важным, так как пока очиститель «спал» между удалениями, пользовательский поток мог легитимно вызвать `delete(key)` или обновить ключ через `set()`.
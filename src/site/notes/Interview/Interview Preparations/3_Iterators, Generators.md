---
{"dg-publish":true,"permalink":"/interview/interview-preparations/3-iterators-generators/"}
---

# Iterable, Iterator и внутреннее устройство генераторов

Итерируемый объект (Iterable) — это сущность, возвращающая итератор при вызове функции `iter()` (под капотом вызывается дандер-метод `__iter__()`). 

Итератор (Iterator) — это объект, сохраняющий внутреннее состояние обхода, реализующий метод `__next__()` и возвращающий `self` в своем `__iter__()`. Когда элементы заканчиваются, итератор обязан выбросить исключение `StopIteration`, которое неявно обрабатывается циклом `for`.

Генераторы (создаваемые через ключевое слово `yield`) — это встроенный в CPython механизм реализации итераторов, работающий через управление фреймами стека (stack frames).

**Edge Cases / Trade-offs:** Генераторы обеспечивают константное потребление памяти `O(1)` независимо от объема обрабатываемых данных. Расплата за это — однопроходность (single-pass). Генератор невозможно "отмотать" назад или обойти дважды.

```python
class CustomRange:
    """Низкоуровневая реализация итератора (эквивалент генератора)"""
    def __init__(self, stop):
        self.current = 0
        self.stop = stop

    def __iter__(self):
        return self  # Итератор всегда возвращает сам себя

    def __next__(self):
        if self.current >= self.stop:
            raise StopIteration
        value = self.current
        self.current += 1
        return value

def custom_range_gen(stop):
    """Эквивалентный генератор — интерпретатор сгенерирует класс-машину состояний за нас"""
    current = 0
    while current < stop:
        yield current
        current += 1
```

# Двунаправленные генераторы (Coroutines) и yield from

### Теория

Оператор `yield` в Python является выражением, то есть он может не только отдавать значения, но и принимать их извне. Метод генератора `.send(value)` возобновляет выполнение фрейма, и `yield` возвращает переданное значение. Это превращает генератор в сопрограмму (coroutine) на основе потока данных.

Конструкция `yield from <iterable>` работает вот так:

1. `yield from` автоматически пробрасывает вызовы `.send()` и `.throw()` во вложенный генератор (subgenerator).
    
2. `yield from` умеет извлекать значение, которое вложенный генератор возвращает через оператор `return`. В CPython `return value` внутри генератора транслируется в `raise StopIteration(value)`. Оператор `yield from` перехватывает это исключение и возвращает его полезную нагрузку.

```python
def subgenerator():
    received = yield "Ready"
    print(f"Subgenerator received: {received}")
    return "Subgenerator Result"

def delegating_generator():
    # result получит значение "Subgenerator Result" после завершения subgenerator
    result = yield from subgenerator()
    print(f"Delegator got: {result}")
    yield "Done"

gen = delegating_generator()
print(gen.send(None))      # Инициализация (всегда None). Выведет: Ready
print(gen.send("Data"))    # Выведет: Subgenerator received: Data \n Delegator got: Subgenerator Result \n Done
```

### Реальный пример: Конвейер скользящего среднего

Двунаправленные генераторы позволяют организовать обработку потоковых данных в стиле Reactive Programming без накладных расходов на создание классов со состоянием.

Генератор хранит окно значений во внутреннем состоянии фрейма, принимает входящие метрики через метод `.send()` и сразу возвращает вычисленное значение.

```python
from collections import deque
from typing import Generator, Optional

def moving_average(window_size: int) -> Generator[Optional[float], float, None]:
    """
    Вычисляет скользящее среднее для потока чисел.
    
    Yields:
        Optional[float]: Текущее среднее или None, пока окно не заполнено.
    Receives:
        float: Очередное значение из потока.
    """
    window = deque(maxlen=window_size)
    window_sum = 0.0
    val = None

    while True:
        # Принимаем значение и одновременно отдаем результат предыдущей итерации
        new_val = yield val
        
        if len(window) == window_size:
            window_sum -= window[0]
            
        window.append(new_val)
        window_sum += new_val
        
        if len(window) == window_size:
            val = window_sum / window_size
        else:
            val = None

# Использование
calc = moving_average(window_size=3)

# Первичный запуск — продвигаем генератор до первого yield
next(calc)  # или calc.send(None)

print(calc.send(10.0))  # None (окно: [10.0])
print(calc.send(20.0))  # None (окно: [10.0, 20.0])
print(calc.send(30.0))  # 20.0 (окно: [10.0, 20.0, 30.0])
print(calc.send(40.0))  # 30.0 (окно: [20.0, 30.0, 40.0])
```

### Разбор механики

1. Вызов `next(calc)` выполняет код функции до первого выражения `yield val`, где `val` изначально равен `None`. Инструкция `new_val = yield` останавливает фрейм в ожидании входящих данных.
    
2. Вызов `calc.send(10.0)` передает число `10.0` напрямую в переменную `new_val`. Фрейм возобновляет работу, пересчитывает сумму за $O(1)$ благодаря кольцевому буферу `deque`, доходит до следующего витка цикла и блокируется на `yield`, возвращая текущий `val`.
    
3. Память выделяется один раз при создании `deque(maxlen=N)`. Накладные расходы на вызов метода `.send()` минимальны по сравнению с вызовом методов экземпляра класса.

# Задачи с собеседований

### Задача 1. Ловушка исчерпания генератора

**Условие:** Разработчик написал функцию проверки наличия элементов в потоке данных. Почему второй цикл не выполняется и как решить проблему без загрузки данных в оперативную память?

```python
def process_data(data_stream):
    print("First pass:")
    for item in data_stream:
        print(item)
        if item == 2:
            break
            
    print("Second pass:")
    for item in data_stream:
        print(item)

stream = (x for x in range(5))
process_data(stream)
```

**Разбор механики:** Код выведет `0, 1, 2` в первом цикле и `3, 4` во втором. Объект генератора `stream` сохраняет состояние (указатель). После прерывания первого цикла `break`, генератор остается приостановленным на элементе `2`. Второй цикл `for` просто продолжает опрашивать тот же самый инстанс итератора с места последней остановки. Если бы первый цикл дошел до конца, второй не вывел бы ничего, так как генератор уже находился бы в состоянии `StopIteration`.

**Решение:** Если требуется независимый обход одного и того же ленивого потока, используется `itertools.tee`.
```python
import itertools

def process_data_fixed(data_stream):
    stream1, stream2 = itertools.tee(data_stream, 2)
    # Теперь stream1 и stream2 — независимые итераторы
```

**Interview Tip:** `tee` под капотом использует очередь (FIFO). Если `stream1` уходит далеко вперед, а `stream2` не читается, `tee` будет кэшировать все промежуточные элементы в оперативной памяти, что уничтожит все преимущества `O(1)` памяти генератора и приведет к OOM (Out Of Memory).

### Задача 2. Выравнивание (Flatten) вложенного списка произвольной глубины

**Условие:** Напишите генератор, который принимает глубоко вложенный массив (например, `[1, [2, [3, 4], 5], 6]`) и возвращает плоскую последовательность элементов. Строки не должны разбиваться на символы.

```python
from typing import Iterable, Any

def flatten(items: Iterable[Any]):
    for item in items:
        # Проверка на Iterable, но исключаем строки и байты, 
        # иначе произойдет бесконечная рекурсия
        if isinstance(item, Iterable) and not isinstance(item, (str, bytes)):
            yield from flatten(item)
        else:
            yield item

data = [1, [2, "hello", [3, 4]], b"bytes", 5]
print(list(flatten(data)))  # [1, 2, 'hello', 3, 4, b'bytes', 5]
```

**Разбор механики:** Задача проверяет два аспекта: умение писать рекурсивные генераторы с использованием `yield from` и понимание того, что строка в Python является итерируемым объектом. Если не добавить проверку `not isinstance(item, (str, bytes))`, алгоритм попытается итерироваться по строке `"hello"`, получит символ `"h"`. Символ `"h"` в Python — это строка длиной 1, которая тоже итерируема. Вызов `flatten("h")` снова выдаст `"h"`, что приведет к переполнению стека вызовов `RecursionError`.

**Edge Cases / Trade-offs:** Рекурсия в Python ограничена (по умолчанию 1000 фреймов стека). Если вложенность массивов составит 1001 уровень, код упадет. Для обхода этого ограничения необходимо переписать алгоритм с использованием собственного стека на базе списка (`list.pop() / list.extend()`), отказавшись от рекурсии вызовов функций.

### Задача 3. Контроль выполнения ресурсоемкого итератора с таймаутом

**Условие:** Напишите функцию-генератор `timeout_generator(iterable, max_seconds)`, которая оборачивает произвольный итератор и прекращает генерацию элементов, если суммарное время обработки превысило `max_seconds`. Замер времени должен включать время, затрачиваемое вызывающим кодом на обработку каждого элемента.

```python
import time
from typing import Iterable, Any, Generator

def timeout_generator(
	iterable: Iterable[Any], 
	max_seconds: float
) -> Generator[Any, None, None]:
    start_time = time.perf_counter()
    iterator = iter(iterable)
    
    for item in iterator:
        yield item
        # Замеряем время ПОСЛЕ того, как вызывающий код возвращает управление через next()
        if time.perf_counter() - start_time > max_seconds:
            return

# Пример использования
def slow_process():
    stream = range(100)
    for data in timeout_generator(stream, max_seconds=0.5):
        print(f"Processing {data}")
        time.sleep(0.2)  # Имитация тяжелой обработки на стороне consumer

slow_process()
```

**Разбор механики:** Ключевой момент — понимание точки передачи управления при работе с `yield`. Код внутри `timeout_generator` засыпает на строке `yield item`. Выполнение передается внешнему циклу `for data in ...`.

Время, потраченное во внешнем цикле на `time.sleep(0.2)`, учитывается при выполнении проверки `time.perf_counter() - start_time > max_seconds` на следующем шаге итерации. Когда лимит времени превышен, генератор делает `return`, что автоматически генерирует исключение `StopIteration` и корректно завершает внешний цикл.
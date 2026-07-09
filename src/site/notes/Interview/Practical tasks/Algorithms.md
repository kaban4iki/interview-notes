---
{"dg-publish":true,"permalink":"/interview/practical-tasks/algorithms/"}
---

### 1. Слияние временных интервалов (Merge Intervals)

**Задача:** Дан список периодов активности клиента (начало, конец). Некоторые периоды пересекаются. Нужно вернуть список максимально длинных непересекающихся интервалов.

```python
def merge_intervals(intervals: list[tuple[int, int]]) -> list[tuple[int, int]]:
    sorted_intervals = sorted(intervals, key=lambda v: v[0])
    
    grouped_intervals = []
    
    start, end = sorted_intervals[0]
    for next_start, next_end in sorted_intervals[1:]:
        if next_start > end:
            grouped_intervals.append((start, end))
            start, end = next_start, next_end
        elif next_end > end:
            end = next_end
    
    grouped_intervals.append((start, end))
    
    return grouped
```

- **В:** Какая сложность алгоритма и можно ли её улучшить?
    
- **О:** Сложность $O(n \log n)$ из-за сортировки. Улучшить нельзя, так как для поиска пересечений нам необходимо знать порядок событий.

### 2. Скользящее среднее в потоке (Moving Average)

**Задача:** Реализовать структуру, которая принимает значения транзакций и выдает среднее за последние $K$ записей. Оптимизировать по времени.

```python
from collections import deque

class MovingAverage:
    def __init__(self, size: int):
        self.size = size
        self.queue: deque[float] = deque()
        self.current_sum: float = 0.0

    def next(self, val: float) -> float:
        if len(self.queue) == self.size:
            self.current_sum -= self.queue.popleft()
		
        self.queue.append(val)
        self.current_sum += val
        return self.current_sum / len(self.queue)
```

- **В:** Почему мы храним `current_sum`, а не считаем `sum(list)` каждый раз?
    
- **О:** Чтобы добиться сложности $O(1)$ на каждый вызов. `sum()` дал бы $O(K)$.

### 3. Top-K элементов (Transaction Frequency)

**Задача:** Найти $K$ самых частых ID магазинов в логе из $N$ транзакций.

```python
import heapq
from collections import Counter

def top_k_frequent(ids: list[str], k: int) -> list[str]:
    count = Counter(ids)
    return [
	    item for item, _ in heapq.nlargest(
			    k, count.items(), key=lambda x: x[1]
		    )
		]
```

- **В:** Какая сложность этого решения?
    
- **О:** $O(N + N \log K)$. Подсчет — $O(N)$, извлечение из кучи — $O(N \log K)$.

### 4. Нахождение пересечения двух списков клиентов

**Задача:** Даны два списка ID клиентов (отсортированные). Найти общих клиентов за $O(N)$.

```python
def intersect(list_a: list[int], list_b: list[int]) -> list[int]:
    i, j = 0, 0
    res = []
    while i < len(list_a) and j < len(list_b):
        if list_a[i] == list_b[j]:
            res.append(list_a[i])
            i += 1; j += 1
        elif list_a[i] < list_b[j]:
            i += 1
        else:
            j += 1
    return res
```

- **В:** Что если один список намного короче другого?
    
- **О:** Эффективнее использовать бинарный поиск по длинному списку для каждого элемента короткого. Сложность станет $O(M \log N)$.

### 5. Реализация Rate Limiter (Token Bucket)

**Задача:** Алгоритмически реализовать логику "корзины токенов" без использования внешних библиотек.

```python
import time

class TokenBucket:
    def __init__(self, capacity: int, fill_rate: float):
        self.capacity = capacity
        self.fill_rate = fill_rate
        self.tokens = capacity
        self.last_fill = time.monotonic()

    def request(self) -> bool:
        now = time.monotonic()
        added = (now - self.last_fill) * self.fill_rate
        self.tokens = min(self.capacity, self.tokens + added)
        self.last_fill = now
        
        if self.tokens > 0:
            self.tokens -= 1
            return True
        return False
```

- **В:** Как сделать эту структуру потокобезопасной?
    
- **О:** Добавить `threading.Lock` вокруг обновления `self.tokens` и `self.last_fill`.

### 6. Группировка анаграмм (Transaction Descriptions)

**Задача:** Сгруппировать строки, которые являются анаграммами друг друга.

```python
from collections import defaultdict

def group_anagrams(descriptions: list[str]) -> list[list[str]]:
    groups: dict[tuple[int, ...], list[str]] = defaultdict(list)
    
    for desc in descriptions:
        count = [0] * 26
        for char in desc.lower():
            if 'a' <= char <= 'z':
                count[ord(char) - ord('a')] += 1
        groups[tuple(count)].append(desc)
        
    return list(groups.values())
```

**Вопросы и ответы:**

- **В: Почему кортеж `tuple(count)` лучше, чем `sorted(string)`?**
    - **О:** Сортировка — это $O(L \log L)$, а подсчет частот — $O(L)$. На длинных строках разница существенна.
        
- **В: Как обработать Unicode/Кириллицу?**
    - **О:** Использовать обычный `dict` для подсчета частот в качестве ключа (предварительно превратив его в `frozenset` или отсортированный кортеж пар `(char, count)`).

### 7. Поиск вложенных структур (Stack)

**Задача:** Проверить валидность вложенности скобок/тегов.

```python
def is_valid_config(stream: str) -> bool:
    stack: list[str] = []
    mapping = {")": "(", "}": "{", "]": "["}
    
    for char in stream:
        if char in mapping.values():
            stack.append(char)
        elif char in mapping:
            if not stack or stack.pop() != mapping[char]:
                return False
    return not stack
```

**Вопросы и ответы:**

- **В: Какова пространственная сложность?**
    - **О:** $O(n)$ в худшем случае (например, только открывающие скобки).
        
- **В: Как оптимизировать для гигантских файлов?**
    - **О:** Использовать генератор, читающий файл посимвольно: `for char in file.read(1)`.

### 8. Задача о рюкзаке (Dynamic Programming)

**Задача:** Выбор оптимального набора задач с разной ценностью и весом при ограничении "грузоподъемности" (времени/ресурсов).

```python
def max_profit(weights: list[int], values: list[int], capacity: int) -> int:
    n = len(values)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            if weights[i-1] <= w:
                dp[i][w] = max(values[i-1] + dp[i-1][w - weights[i-1]], dp[i-1][w])
            else:
                dp[i][w] = dp[i-1][w]
                
    return dp[n][capacity]
```

**Вопросы и ответы:**

- **В: Как оптимизировать память до $O(capacity)$?**
    - **О:** Использовать одномерный массив `dp`, так как для расчета `i`-й строки нужна только `i-1`-я.

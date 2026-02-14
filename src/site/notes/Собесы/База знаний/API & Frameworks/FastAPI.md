---
{"dg-publish":true,"permalink":"/sobesy/baza-znanij/api-and-frameworks/fast-api/"}
---

## Общая архитектура FastAPI

**Кратко:**  
FastAPI — это композиция:
- **ASGI** (интерфейс)
- **Starlette** (HTTP, routing, middleware)
- **Pydantic** (валидация, сериализация)

Сам FastAPI — orchestration layer.

**Типовой вопрос:**  
Чем FastAPI принципиально отличается от Flask/Django?

**Ответ:**  
FastAPI:
- построен на ASGI
- изначально async-first
- связывает типы Python с runtime-валидацией
- использует OpenAPI как первичный контракт

---

## ASGI и жизненный цикл запроса

**Кратко:**  
FastAPI не управляет event loop.  
Он исполняется внутри ASGI-сервера (uvicorn, hypercorn).

**Типовой вопрос:**  
Как FastAPI обрабатывает HTTP-запрос?

**Ответ:**  
1. ASGI-сервер принимает соединение  
2. Передаёт request в приложение  
3. Выполняются middleware  
4. Выбирается маршрут  
5. Вызывается endpoint  
6. Ответ возвращается через ASGI

FastAPI не контролирует concurrency — это ответственность сервера.

---

## Routing

**Кратко:**  
Routing полностью реализован в Starlette.  
FastAPI лишь добавляет типизацию и зависимости.

**Типовой вопрос:**  
Как FastAPI выбирает маршрут при конфликтах?

**Ответ:**  
- static routes имеют приоритет
- затем dynamic
- порядок объявления маршрутов важен

```python
@app.get("/users/me")
@app.get("/users/{id}")
```

---

## Dependency Injection (Depends)

**Кратко:**  
Depends — это runtime-граф зависимостей, а не DI-контейнер.

**Типовой вопрос:**  
Почему Depends — не полноценный DI?

**Ответ:**  
Depends:
- создаётся на каждый запрос
- не имеет lifecycle singleton
- выполняется динамически

Подходит для:
- DB-сессий
- auth
- permissions

```python
async def get_db():
	async with Session() as s:
		yield s
```

---

## Middleware

**Кратко:**  
Middleware — глобальный слой обработки HTTP-запросов.

**Типовой вопрос:**  
Когда middleware — плохой выбор?

**Ответ:**  
Когда логика:
- относится к конкретному endpoint
- зависит от бизнес-контекста

Middleware подходит для:
- логирования
- tracing
- headers
- rate limit

В FastAPI есть **встроенные middleware** для общих задач, например:
- **CORSMiddleware** — включает заголовки CORS в ответы, чтобы разрешить кросс-доменные запросы из веб-браузеров.
- **GZip Middleware** — сжимает данные ответов, чтобы уменьшить использование пропускной способности.

---

## Sync vs Async Endpoints

**Кратко:**  
FastAPI поддерживает sync и async функции.

**Типовой вопрос:**  
Что происходит при использовании sync endpoint?

**Ответ:**  
Sync-функция:
- выполняется в thread pool
- не блокирует event loop
- но потребляет ограниченные ресурсы

---

## Auth & Security

**Кратко:**  
FastAPI не реализует auth, только предоставляет инструменты.

**Типовой вопрос:**  
Почему security utilities FastAPI редко используют напрямую?

**Ответ:**  
Потому что:
- OAuth helpers — схемы, не реализация
- реальные системы требуют кастомной логики

FastAPI — интеграционный фреймворк.
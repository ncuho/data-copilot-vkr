# Data Copilot

Мультиагентная система интеллектуальной аналитики в Lakehouse. Тема выпускной квалификационной работы по направлению «Фундаментальная информатика и информационные технологии», МАИ, бакалавриат.

## О чём проект

Бизнес-пользователю нужен ответ на вопрос по данным, но до данных не добраться без SQL и понимания схемы хранилища. Data Copilot закрывает этот разрыв: по вопросу на естественном языке он сам проходит весь аналитический цикл, от разбора вопроса до готового ответа с визуализацией.

Цикл выглядит так:

1. Понять, что спрашивают, и разбить задачу на шаги.
2. Поднять семантику из каталога данных: бизнес-термины, определения метрик, связи между таблицами.
3. Написать и проверить SQL.
4. Выполнить запрос.
5. Разобрать результат и найти инсайты.
6. Построить визуализацию.

Координирует этапы оркестратор, за каждый этап отвечает специализированный агент. То есть это не один вызов языковой модели, а управляемый конвейер. При этом внутри агент устроен как гибкий ReAct-цикл: последовательность инструментов для каждого вопроса определяет модель, поэтому глубокий анализ и follow-up вопросы работают без жёсткого пайплайна.

## Чем это отличается от обычного агента с MCP

Агент с подключёнными инструментами видит сырые таблицы и пишет запрос одним вызовом модели. Для корпоративных данных этого мало. Здесь подход другой:

- RAG по каталогу данных, обогащённый сущностями и мягкими данными, а не голые имена таблиц.
- Память: кратковременная держит контекст разбора, долговременная сохраняет результаты и прошлые запросы.
- Дообучение на эталонных запросах: одобренные запросы возвращаются в контекст, агент учится на них.
- Write-операции в песочнице без глубокого копирования за счёт манифестов Iceberg.
- Проверяемый конвейер запросов вместо одного вызова модели.
- Полный провенанс и аудит: каждый результат прослеживается до запроса и снимка данных.

Отдельно про исполнение. SQL выполняется в режиме read-only, то есть только SELECT. Прочие операции, например запись в песочнице, не копируют данные целиком: Iceberg пересоздаёт манифесты, то есть метаданные, а не сами файлы, и дельту догоняет по мете.

## Архитектура (коротко)

```
datacatalog-ui ── панель «Data Copilot» (чат + MiniBI/Plotly + «View SQL»)
        │  REST + SSE
        ▼
data-copilot (FastAPI + LangChain/LangGraph)
 ├─ orchestrator (LangGraph): classify → agent(ReAct-цикл) → HITL → exec → chart
 ├─ tools: search | get_table_schema | list_schemas | run_sql | recall_memory | build_chart
 ├─ LLMProvider: self-hosted(OpenAI-совместимый) | OpenAI | Anthropic
 ├─ state: checkpointer (диалоги/сессии) + будущий PostgreSQL
 └─ auth: JWT-мост к OMD
        │
        ├─► OpenMetadata: /search/vector/query + /search/query + /ai/context/find + /contextCenter/memories
        └─► Trino / ClickHouse / Postgres (SELECT-only, per-user identity)
```

Концептуально это цепочка: запрос пользователя, оркестратор, затем агенты семантики каталога, генерации SQL, исполнения, анализа инсайтов и визуализации. Сверху и снизу её опоясывают сквозные механизмы: память и дообучение, а также guardrails, human-in-the-loop, трассировка и аудит.

## Структура

```
app/
├── config.py                 # настройки (pydantic-settings, .env)
├── main.py                   # FastAPI-приложение
├── api/
│   ├── chat.py               # эндпоинты диалогов / сообщений / одобрения SQL
│   └── schemas.py            # модели запросов/ответов
└── core/
    ├── llm/provider.py       # LLM-провайдер (self-hosted/openai/anthropic)
    ├── graph/
    │   ├── state.py          # AgentState
    │   ├── tools.py          # инструменты агента
    │   ├── nodes.py          # classify + agent (ReAct-цикл)
    │   └── graph.py          # сборка графа
    ├── sql/
    │   ├── validator.py      # SELECT-only (sqlglot) + allowlist
    │   └── executor.py       # QueryExecutor (Trino/CH/PG + mock)
    └── search/client.py      # клиент поиска OMD + RRF-merge
```

## Стек

Python, LangGraph, OpenMetadata, dbt, Trino, Iceberg, Superset, pgvector, PostgreSQL, Docker, Kubernetes.

## Быстрый старт

```bash
# 1. Окружение
uv sync                      # создаст .venv (Python 3.11) и установит зависимости
cp .env.example .env         # заполните LLM-ключи / OMD-адрес

# 2. Запуск (в режиме mock, без реальных OMD/СУБД)
uv run uvicorn app.main:app --reload --port 8000

# 3. Проверка
curl http://localhost:8000/health
```

## Human-in-the-loop

1. `POST /api/v1/dialogs` — создать диалог.
2. `POST /api/v1/dialogs/{id}/messages` `{"question": "...", "mode": "interactive"}`
   — если агент дошёл до SQL, вернётся `needs_approval=true` + `pending_sql`.
3. `POST /api/v1/dialogs/{id}/messages/approve` `{"action": "approve", "sql": "..."}`
   — продолжить граф (или `reject`).

В режиме `autopilot` SQL исполняется без остановки (в рамках лимитов и
SELECT-only валидации).

## Безопасность

- Только `SELECT` (детерминированная валидация через `sqlglot`).
- Allowlist таблиц/колонок из прав пользователя в OMD.
- Лимиты строк и таймаута на каждом запросе.
- HITL-одобрение SQL по умолчанию.
- Write-операции в песочнице без глубокого копирования через манифесты Iceberg.

## Roadmap

- **P2** — реальный `SearchClient` (semantic/hybrid/NLQ) + `list_schemas`.
- **P3** — генерация SQL со schema-context, ужесточение allowlist.
- **P4** — реальные `TrinoExecutor` / `ClickHouseExecutor` / `PostgresExecutor`
  с имперсонацией (X-Trino-User / X-ClickHouse-User / SET ROLE).
- **P5** — MiniBI: chart-spec → Plotly-рендерер в UI, отчёт + «View SQL».
- **P6** — диалоги в PostgreSQL, долгая память (`contextMemory`), аудит.
- **P7** — streaming (SSE), rate-limit, prompt-injection-защита, observability.

## Статус

В работе. Код и README обновляются по мере продвижения по теме.

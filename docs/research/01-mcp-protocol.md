# Протокол Model Context Protocol (MCP) — исследование по первоисточникам

> Назначение: база знаний для проекта «1C MCP» (Windows + VS Code + агент в VS Code + 1С:Предприятие).
> Характер документа: **исследование** (сбор источников и сравнительных таблиц), без предложения архитектуры.
> Дата составления: 2026-10-14. Актуальная ревизия спецификации на момент составления — **2026-07-28**.

---

## 1. Что такое MCP

Model Context Protocol (MCP) — открытый стандарт, представленный компанией Anthropic **25 ноября 2024 года** как способ стандартизовать подключение ИИ-приложений (хостов) к внешним системам — данным, инструментам и шаблонам промптов. Авторы — Дэвид Сориа Парра (David Soria Parra) и Джастин Шпар-Саммерс (Justin Spahr-Summers).

Ключевые характеристики:

- **Формат сообщений**: JSON-RPC 2.0, кодировка UTF-8.
- **Двухслойная модель**: «слой данных» (data layer — семантика JSON-RPC, примитивы, обнаружение возможностей) поверх «транспортного слоя» (transport layer — механизмы доставки: stdio, HTTP).
- **Модель «хост — клиент — сервер»** (client-host-server).
- **Версионирование по дате** (YYYY-MM-DD), а не semver.
- В декабре 2025 года MCP передан фонду **Agentic AI Foundation** (направленный фонд под эгидой Linux Foundation, соучредители Anthropic, Block и OpenAI) — управление перешло в вендор-нейтральную организацию.

Первоисточники: [анонс Anthropic](https://anthropic.com/news/model-context-protocol), [спецификация](https://modelcontextprotocol.io/specification/2026-07-28), [репозиторий спецификации](https://github.com/modelcontextprotocol/modelcontextprotocol).

---

## 2. Архитектура (host / client / server)

MCP следует архитектуре «клиент-сервер», в которой **хост** (host) — это ИИ-приложение (например, Claude Desktop, Claude Code, **Visual Studio Code**), устанавливающее соединения с одним или несколькими MCP-серверами. Хост делает это, создавая **по одному MCP-клиенту на каждый сервер**; каждый клиент поддерживает выделенное соединение со своим сервером.

Три роли:

| Роль | Описание | Пример |
|---|---|---|
| **MCP Host (хост)** | ИИ-приложение, координирующее один или несколько MCP-клиентов; управляет жизненным циклом соединений, политиками безопасности, согласием пользователя, интеграцией LLM | Visual Studio Code, Claude Desktop |
| **MCP Client (клиент)** | Компонент, поддерживающий соединение с одним сервером и получающий от него контекст для хоста; 1:1 с сервером | объект-клиент внутри VS Code |
| **MCP Server (сервер)** | Программа, предоставляющая контекст (tools/resources/prompts) клиентам; локальная или удалённая | filesystem-сервер, Sentry MCP |

Пример из официальной документации: VS Code выступает хостом; при подключении к серверу Sentry среда исполнения VS Code создаёт клиентский объект, обслуживающий это соединение; при подключении ко второму серверу создаётся ещё один клиентский объект.

### 2.1. Два слоя

- **Слой данных** — протокол обмена на базе JSON-RPC 2.0: структура и семантика сообщений, обнаружение версий и возможностей (`server/discover`), примитивы (tools, resources, prompts), уведомления и прогресс.
- **Транспортный слой** — механизмы и каналы связи: установка соединения, фрейминг сообщений, авторизация.

### 2.2. Обнаружение возможностей (в ревизии 2026-07-28)

Современная ревизия сделала протокол **stateless**: каждый запрос самодостаточен и несёт в поле `_meta`:

- `io.modelcontextprotocol/protocolVersion` — версия протокола;
- `io.modelcontextprotocol/clientCapabilities` — возможности клиента;
- `io.modelcontextprotocol/clientInfo` — идентификатор и версия клиента.

`server/discover` — обязательный для сервера RPC, который клиент **может** вызвать для получения версий, возможностей и идентификатора сервера заранее (но не обязан). При несовпадении версии сервер отвечает ошибкой `UnsupportedProtocolVersionError` (код `-32022`) со списком поддерживаемых версий.

### 2.3. Принципы проектирования

1. Серверы должны быть **предельно простыми** в создании (сложность оркестрации берёт на себя хост).
2. Серверы должны быть **композируемыми** (много серверов, единый протокол).
3. Серверы **не должны читать всю переписку** и «видеть» другие серверы (изоляция, границы безопасности контролирует хост).

---

## 3. Примитивы протокола

Примитивы — важнейшая концепция MCP: они определяют, чем клиенты и серверы могут обмениваться. Каждый тип примитива имеет методы обнаружения (`*/list`), получения (`*/get`) и, где применимо, исполнения (`tools/call`).

### 3.1. Серверные примитивы

| Примитив | Назначение | Методы |
|---|---|---|
| **Tools** | Исполняемые функции, которые ИИ может вызывать для действий (файловые операции, вызовы API, запросы к БД) | `tools/list`, `tools/call` |
| **Resources** | Источники данных, дающие контекст (содержимое файлов, записи БД, ответы API) через URI-шаблоны | `resources/list`, `resources/read`, `resources/subscribe` (подписка на изменения) |
| **Prompts** | Переиспользуемые шаблоны, структурирующие взаимодействие с моделью (системные промпты, few-shot) | `prompts/list`, `prompts/get` |

### 3.2. Клиентские примитивы

| Примитив | Назначение | Статус в 2026-07-28 |
|---|---|---|
| **Roots** | Границы файловой системы / рабочих папок, к которым серверу разрешён доступ; сервер запрашивает `roots/list`, клиент шлёт `notifications/roots/list_changed` | **Deprecated** (с 2026-07-28); замена — передача путей через параметры инструментов или конфигурацию сервера |
| **Sampling** | Запрос сервером LLM-генерации через клиента (`sampling/createMessage`), чтобы не встраивать собственный LLM-SDK | **Deprecated** (с 2026-07-28); замена — прямая интеграция с API провайдеров LLM |
| **Elicitation** | Запрос сервером дополнительной информации/подтверждения у пользователя (`elicitation/create`) | Активен; в 2026-07-28 переведён на паттерн Multi Round-Trip Requests (MRTR) — результат `InputRequiredResult` вместо постоянно открытого потока |
| **Logging** | Передача сервером логов клиенту | **Deprecated** (с 2026-07-28); замена — `stderr` (stdio) или OpenTelemetry |

> Примечание: в задании примитивы «sampling» и «roots» упомянуты как основные. Важно зафиксировать, что в **текущей** ревизии 2026-07-28 оба они **объявлены устаревшими** с минимальным окном миграции 12 месяцев (удаление не ранее ревизии, выпущенной 2027-07-28 или позже). В «легаси»-ревизиях (2025-11-25 и ранее) они полноценно активны.

### 3.3. Уведомления и подписки

Уведомления (JSON-RPC notifications, без поля `id`) не требуют ответа и доставляются по подписке: клиент открывает поток `subscriptions/listen` с фильтром (например, `toolsListChanged`), и сервер шлёт `notifications/tools/list_changed` при изменении списка инструментов. Это событийный, best-effort механизм синхронизации.

---

## 4. Транспорты

Официально стандартизованы **два** транспорта (плюс возможность кастомных). Исторически существовал третий — HTTP+SSE (из ревизии 2024-11-05), заменённый на Streamable HTTP.

- **stdio** — новые строки-разделители JSON-RPC поверх стандартных потоков дочернего процесса (клиент запускает сервер как subprocess).
- **Streamable HTTP** — каждый запрос это HTTP POST на единый MCP-endpoint; ответы приходят как JSON-объект либо как SSE-поток в рамках запроса.
- **HTTP+SSE (устаревший)** — из ревизии 2024-11-05: отдельный `GET /sse` для стрима от сервера + `POST`-endpoint для запросов клиента.

Ключевые правила stdio:

- Сервер читает JSON-RPC из `stdin`, пишет в `stdout`.
- Сообщения разделяются переводами строки, **не должны** содержать встроенных переводов.
- `stdout` **только** для валидных MCP-сообщений; логи — только в `stderr`.
- Клиент **не должен** писать в `stdin` сервера ничего, кроме валидных MCP-сообщений.

Ключевые правила Streamable HTTP:

- Единый endpoint (например `https://example.com/mcp`), поддерживающий и POST, и GET.
- Клиент может открыть `GET` с заголовком `Accept: text/event-stream`, чтобы получать SSE-поток сообщений от сервера.
- В ревизии 2026-07-28 добавлены обязательные HTTP-заголовки `Mcp-Method` и `Mcp-Name` (и зеркалирование `_meta` в заголовки), чтобы шлюзы/балансировщики маршрутизировали запросы без разбора тела; тело всегда остаётся источником истины (расхождение заголовка и тела → ошибка `HeaderMismatch`).
- Безопасность: обязательная проверка `Origin` (защита от DNS rebinding), локальная привязка к `127.0.0.1`, обязательная аутентификация.

### Таблица 1. Сравнение транспортов MCP

| Критерий | stdio | Streamable HTTP | HTTP+SSE (устаревший) |
|---|---|---|---|
| Топология | Локальный subprocess (1 клиент на процесс) | Удалённый сервер, много клиентов, stateless-запросы | Удалённый сервер, постоянный SSE-стрим + POST |
| Направление | Двунаправленный через stdin/stdout | POST-запрос → JSON или request-scoped SSE; server→client через `subscriptions/listen` | GET-стрим сервер→клиент + POST клиент→сервер |
| Фрейминг | Newline-delimited JSON-RPC | HTTP-запросы; опционально SSE внутри ответа | SSE-события + JSON-RPC в POST |
| Сессия / состояние | Процесс живёт, пока жив хост | Stateless (в 2026-07-28 сессий и `Mcp-Session-Id` нет) | Stateful (endpoint-событие с session id) |
| Маршрутизация шлюзами | Н/п (локальный процесс) | Да: заголовки `Mcp-Method`/`Mcp-Name`, обычный round-robin LB | Затруднена (нужен sticky session) |
| Статус | Активен (рекомендован «когда возможно») | Активен (стандарт для удалённых серверов) | Deprecated (с 2025-11-25; отдельный график удаления) |
| Типичный сценарий | Локальные серверы: filesystem, git, IDE-плагины | Удалённые/облачные серверы, серверлес | Легаси-развёртывания |

---

## 5. Официальные SDK

Официальные SDK собраны в организации GitHub `modelcontextprotocol`. Они разделены на **уровни (tiers)** по полноте функций, поддержке протокола и обязательствам сопровождения.

Полный список (по странице SDK, ревизия 2026-07-28):

| SDK | Репозиторий | Tier |
|---|---|---|
| TypeScript | [modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk) | Tier 1 |
| Python | [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) | Tier 1 |
| C# | [modelcontextprotocol/csharp-sdk](https://github.com/modelcontextprotocol/csharp-sdk) | Tier 1 |
| Go | [modelcontextprotocol/go-sdk](https://github.com/modelcontextprotocol/go-sdk) | Tier 1 |
| Rust | [modelcontextprotocol/rust-sdk](https://github.com/modelcontextprotocol/rust-sdk) | Tier 1 |
| Java | [modelcontextprotocol/java-sdk](https://github.com/modelcontextprotocol/java-sdk) | Tier 2 |
| Ruby | [modelcontextprotocol/ruby-sdk](https://github.com/modelcontextprotocol/ruby-sdk) | Tier 2 |
| Swift | [modelcontextprotocol/swift-sdk](https://github.com/modelcontextprotocol/swift-sdk) | Tier 3 |
| PHP | [modelcontextprotocol/php-sdk](https://github.com/modelcontextprotocol/php-sdk) | Tier 3 |
| Kotlin | [modelcontextprotocol/kotlin-sdk](https://github.com/modelcontextprotocol/kotlin-sdk) | Tier 3 |

Каждый SDK предоставляет одинаковую функциональность (создание серверов с tools/resources/prompts, клиентов, локальные и удалённые транспорты, типобезопасное соответствие протоколу), но следует идиомам своего языка.

Заметки по сопровождению:

- **C# SDK** — разрабатывается в сотрудничестве с **Microsoft**; v2.0 реализует ревизию 2026-07-28.
- **Kotlin SDK** — в сотрудничестве с **JetBrains**.
- **Java SDK** — в сотрудничестве с **Spring AI**.
- К моменту ревизии 2026-07-28 обновлены до нового стандарта TypeScript, Python, Go и C# SDK (Tier 1). TypeScript SDK стал семейством пакетов `@modelcontextprotocol/*` 2.0.0 (`server`, `client`, `core`, `node`, `hono`, `express`, `server-legacy`); Python SDK в линейке 2.x (например, 2.2.0 от 2026-09-07) параллельно с maintenance-линейкой 1.x.

### Таблица 2. Сравнение официальных SDK (по состоянию на ревизию 2026-07-28)

| SDK | Язык / платформа | Tier | Сопровождение | Примечания |
|---|---|---|---|---|
| TypeScript | TypeScript/JavaScript (Node.js, Bun, Deno, браузер) | 1 | modelcontextprotocol | Express-подобный API; Standard Schema (Zod/Valibot/ArkType); stdio + Streamable HTTP; адаптеры Express/Hono/Node |
| Python | Python (asyncio) | 1 | modelcontextprotocol | FastMCP API; stdio + HTTP; пакет `mcp` |
| C# | .NET (C#) | 1 | с Microsoft | NuGet `ModelContextProtocol`; async/await; v2.0 под 2026-07-28 |
| Go | Go | 1 | modelcontextprotocol | — |
| Rust | Rust | 1 | modelcontextprotocol | крейт `rmcp` |
| Java | Java / JVM | 2 | с Spring AI | интеграция со Spring |
| Ruby | Ruby | 2 | modelcontextprotocol | гем MCP |
| Kotlin | Kotlin / JVM / Android | 3 | с JetBrains | корутины; `io.modelcontextprotocol:kotlin-sdk` |
| Swift | Swift | 3 | modelcontextprotocol | — |
| PHP | PHP | 3 | modelcontextprotocol | — |

---

## 6. Версионирование спецификации и текущая ревизия

- Схема версионирования — **по дате** (YYYY-MM-DD), не semver. Идентификатор версии = дата последнего breaking change.
- Репозиторий: спецификация и схема живут в **`modelcontextprotocol/modelcontextprotocol`** (монолит: спецификация + документация + блог); ранее — отдельный репозиторий `modelcontextprotocol/specification`. Схема определяется в TypeScript (`schema/<version>/schema.ts`), из которой генерируются JSON Schema и справочник.
- Специальный идентификатор **`draft`** обозначает незавершённую работу и может меняться без предупреждения.

Хронология ревизий:

| Ревизия | Ключевое |
|---|---|
| **2024-11-05** | Первый релиз: JSON-RPC 2.0, stateful-соединения, tools/resources/prompts (сервер) + sampling (клиент); транспорты stdio и HTTP+SSE |
| **2025-03-26** | Доработки; уточнения JSON-RPC |
| **2025-06-18** | Streamable HTTP стандартизован; structured tool output (`structuredContent`); OAuth 2.1 (ресурсные серверы, RFC 8707); elicitation; resource links; удалён JSON-RPC batching; заголовок `MCP-Protocol-Version` |
| **2025-11-25** | Годовщина; stateful с `initialize`-хендшейком; HTTP+SSE deprecate в пользу Streamable HTTP; Tasks (экспериментально в ядре) |
| **2026-07-28** | **Текущая.** Крупнейшая ревизия с запуска: stateless-ядро (удалены `initialize`/`initialized` и `Mcp-Session-Id`), header-based routing (`Mcp-Method`/`Mcp-Name`), Multi Round-Trip Requests (MRTR), кешируемые list-результаты, формальный framework расширений (Apps, Tasks), ужесточение авторизации (RFC 9207, CIMD вместо DCR), формальная политика deprecation (мин. 12 мес.) |

Терминология совместимости (ревизия 2026-07-28):

- **Modern** — версии с per-request метаданными (2026-07-28 и новее).
- **Legacy** — версии с `initialize`-хендшейком (2025-11-25 и ранее).
- **Dual-era** — реализация, поддерживающая обе эпохи.

Разделение «legacy → modern» по JSON Schema: ревизии 2024-11-05…2025-06-18 использовали JSON Schema draft-07; с 2025-11-25 — JSON Schema 2020-12.

---

## 7. Источники

1. https://modelcontextprotocol.io | Официальный сайт MCP (документация и спецификация) | official | 2024 (постоянно обновляется)
2. https://modelcontextprotocol.io/specification/2026-07-28/architecture | Спецификация: Архитектура (ревизия 2026-07-28) | official | 2026-07-28
3. https://modelcontextprotocol.io/specification/2026-07-28/basic/transports | Спецификация: Транспорты | official | 2026-07-28
4. https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning | Спецификация: Версионирование и совместимость | official | 2026-07-28
5. https://modelcontextprotocol.io/specification/2026-07-28/changelog | Спецификация: Changelog ревизии | official | 2026-07-28
6. https://modelcontextprotocol.io/docs/2026-07-28/sdk | Список официальных SDK и их уровней (tiers) | official | 2026-07-28
7. https://github.com/modelcontextprotocol/modelcontextprotocol | Репозиторий спецификации MCP (schema + docs + blog) | official | 2024 (репозиторий создан 2024-09-24)
8. https://github.com/modelcontextprotocol/typescript-sdk | Официальный TypeScript SDK | official | 2024 (запущен с анонсом)
9. https://github.com/modelcontextprotocol/python-sdk | Официальный Python SDK | official | 2024 (запущен с анонсом)
10. https://github.com/modelcontextprotocol/java-sdk | Официальный Java SDK (совместно со Spring AI) | official | 2025
11. https://github.com/modelcontextprotocol/kotlin-sdk | Официальный Kotlin SDK (совместно с JetBrains) | official | 2025
12. https://github.com/modelcontextprotocol/csharp-sdk | Официальный C# SDK (совместно с Microsoft) | official | 2025
13. https://anthropic.com/news/model-context-protocol | Анонс MCP от Anthropic («Introducing the Model Context Protocol») | official | 2024-11-25
14. https://blog.modelcontextprotocol.io/posts/2026-07-28 | Официальный анонс ревизии 2026-07-28 | official | 2026-07-28
15. https://devblogs.microsoft.com/dotnet/announcing-v20-of-the-official-mcp-csharp-sdk/ | Microsoft DevBlogs: анонс v2.0 C# SDK под ревизию 2026-07-28 | secondary-industry | 2026-07
16. https://blog.cloudflare.com/mcp-v2/ | Cloudflare: «The next generation of MCP» (разбор stateless-ревизии) | secondary-industry | 2026-07
17. https://en.wikipedia.org/wiki/Model_Context_Protocol | Википедия: обзорная статья о MCP | secondary-industry | 2024 (постоянно обновляется)

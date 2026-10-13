# Существующие MCP-решения для 1С:Предприятие

> Исследование (сбор источников и сравнительных таблиц, без проектирования собственной архитектуры).
> Дата сбора: 2026-10-14. Метод: web-поиск (site:infostart.ru, site:github.com, реестры MCP-серверов glama/pulsemcp/mcptoplist и др.).

## Общий вывод (gap-анализ)

**Единого «золотого стандарта» MCP-сервера для 1С пока нет.** Экосистема фрагментирована на несколько архитектурных подходов, каждый со своими ограничениями:

1. **Расширение 1С + HTTP-сервис** (`/hs/mcp/`) — наиболее «нативный» путь (MCP-протокол реализуется внутри платформы, BSL). Ограничение: HTTP-сервисы 1С поддерживают только Basic-авторизацию, а MCP-протокол — OAuth 2.0; прямое подключение требует публикации базы без аутентификации (небезопасно). Обходится Python/Node-прокси со stdio/HTTP.
2. **Внешняя обработка (.epf) с встроенным HTTP-сервером** — не требует изменения конфигурации и публикации сервера 1С (ROCTUP/1c-mcp-toolkit).
3. **Native-компонент (DLL)** — MCP-транспорт на C++, бизнес-логика в 1С (DitriXNew/http1c).
4. **Внешний сервер по файлам выгрузки** — работает с `DumpConfigToFiles`/JSON-выгрузкой метаданных, не трогая живую базу (Node.js/Python/Docker).
5. **Инструменты для разработчика (EDT/VS Code)** — MCP поверх 1C:EDT (Java-плагин) и поверх расширения VS Code «1C: Platform Tools» (IPC).
6. **Синтаксис-помощники** — MCP-серверы поверх файла справки `shcntx_ru.hbk` (Elasticsearch/векторный поиск), дают AI знание синтаксиса платформы.
7. **Коммерческие продукты** (ARQA, RCS.kz) — коробочные on-premise решения с поддержкой.

**Зрелость неоднородна:** самые активные и проработанные — `vladimir-kharin/1c_mcp` (v1.6.1, релизы до 2026-08), `ROCTUP/1c-mcp-toolkit`, `DitriXNew` (http1c + EDT-MCP), `INFATON` (51 инструмент). Многие проекты — личные/разовые (artesk/1C_MCP_metadata без обновлений > года и без лицензии; «Компакт MCP» вообще не полноценный MCP-сервер).

**Пробелы (gaps):**
- Нет официального MCP-сервера от фирмы «1С» (ближайшее «официальное» — MCP-режим в BSL Language Server от сообщества 1c-syntax, не от самой фирмы).
- Нет единого стандарта авторизации: Basic-Auth против OAuth2, прокси-прокладки, «вшитые» реквизиты в `default.vrd` — всё это компромиссы безопасности.
- Нет единого решения «метаданные + данные + выполнение кода + отладка» под одной крышей — функциональность размазана по проектам.
- Отладка через MCP (RDBG) — экзотика (один проект `WSSAWER/1c_debug_mcp`).

---

## Найденные решения (по категориям)

### A. MCP-сервер внутри платформы 1С (расширение/обработка)

#### 1. vladimir-kharin/1c_mcp
- **Что:** фреймворк для создания MCP-серверов в 1С через расширение конфигурации. Ядро (протокол) реализовано в 1С; разработчик пишет только бизнес-логику инструментов (обработки в подсистеме `mcp_КонтейнерыИнструментов`). Из коробки — инструменты метаданных (список, структура объекта).
- **Транспорт:** HTTP-сервис 1С `mcp_APIBackend` (`.../база/hs/mcp/`); опциональный **Python-прокси** даёт stdio и HTTP (с поддержкой OAuth2 и «проброса» авторизации через браузер).
- **Что отдаёт:** метаданные конфигурации; кастомные инструменты (tools), resources, prompts.
- **Зрелость:** высокая (v1.6.1, релизы 2025-06 → 2026-08, активен).
- **Язык:** 1С (BSL) + Python (прокси). Лицензия MIT.
- **Ссылка:** https://github.com/vladimir-kharin/1c_mcp

#### 2. artesk/1C_MCP_metadata
- **Что:** расширение 1С с HTTP-сервисом `MCPmeta_MCPСервис` + PowerShell-скрипт-обёртка (stdio) для MCP-клиента.
- **Транспорт:** HTTP-сервис 1С; MCP-мост через PowerShell (stdio). SSE не реализован.
- **Что отдаёт:** структура метаданных (по типам объектов), детальная информация об объекте, поиск объектов (по имени/синониму/комментарию), валидация синтаксиса запросов 1С с описанием ошибки.
- **Зрелость:** низкая — коммитов > года нет, лицензия не указана.
- **Язык:** 1С (BSL) + PowerShell.
- **Ссылка:** https://github.com/artesk/1C_MCP_metadata

#### 3. ROCTUP/1c-mcp-toolkit
- **Что:** система интеграции AI-агентов с 1С через MCP и REST API. Два режима: **встроенный сервер** (HTTP-сервер прямо в обработке 1С `.epf`, без Python) и **прокси** (Python FastAPI + MCP SDK, long polling к внешней обработке).
- **Транспорт:** streamable-http (`/mcp`), REST (`/api/*`). Docker-образ для прокси (`roctup/1c-mcp-toolkit-proxy`).
- **Что отдаёт:** метаданные (дерево полей), данные из справочников (словарь), выполнение запросов (`execute_query`), `get_metadata`. Каналы (`?channel=...`), анонимизация (SpaCy NER), изоляция токенов.
- **Зрелость:** средняя-высокая, активно развивается; без изменения конфигурации и без COM-соединения; совместимость 8.2.13+/8.3.25.
- **Язык:** 1С (BSL) + Python.
- **Ссылка:** https://github.com/ROCTUP/1c-mcp-toolkit

#### 4. DitriXNew/MCP-DB-Client (http1c)
- **Что:** фреймворк MCP-серверов для 1С. Native-компонент **DLL** (C++17) берёт на себя MCP-транспорт (HTTP/SSE, JSON-RPC, сессии, auth, rate limiting), а 1С (обработка `.epf`) — бизнес-логику через `ExternalEvent`. Опциональная подсистема семантического поиска (RAG) на Rust (`rcore`).
- **Транспорт:** Streamable HTTP (`POST/GET /mcp`), SSE-уведомления.
- **Что отдаёт:** полный MCP (tools/resources/prompts с `listChanged`), прогресс-стриминг, пагинация; семантический поиск по собственному контенту; готовая интеграция с Vanessa Automation.
- **Зрелость:** проработанный (полная поддержка протокола, security: origin validation, Bearer token, rate limiting).
- **Язык:** C++17 (DLL) + 1С (BSL) + Rust (поисковое ядро).
- **Ссылка:** https://github.com/DitriXNew/MCP-DB-Client

#### 5. INFATON MCP Server (ООО «ИНФАТОН»)
- **Что:** open-source MCP-сервер для 1С (расширение, не модифицирует конфигурацию), опубликован в виде статьи на Infostart.
- **Транспорт:** HTTP (Basic Auth) через `/hs/mcp/` + stdio (Node.js-обёртка `index.mjs`).
- **Что отдаёт:** **51 инструмент** (v2.1): метаданные (`get_object_metadata`), данные (`get_balance`), создание объектов (`create_object`, `fill_on_basis`), CDC (`get_changes_since`), валидация документов, поиск дублей, бухгалтерия, пакетные операции, аудит, `execute_code` (журналируется в ЖурналРегистрации).
- **Зрелость:** v2.1.0 (14.05.2026), рост 35→51 инструмент за месяц; платформа 8.3.20+ (тест 8.3.27); конфигурации ERP 2.5, Бухгалтерия 3.0, УТ 11, КА 2.
- **Язык:** 1С (BSL) + Node.js.
- **Ссылка:** https://infostart.ru/1c/articles/2712602/

#### 6. RCS-kz/mcp-1c (коммерческий)
- **Что:** production MCP-сервер для 1С (Python), распространяется через npm `@rcs-kz/mcp-1c` (PyArmor-обфускация) + расширение `MCPService.cfe` в базе.
- **Транспорт:** stdio/HTTP (уточняется); клиенты Claude Desktop, Cursor, Cline, Continue.
- **Что отдаёт:** чтение документов/контрагентов, создание документов, проводки. Free Solo tier (10 вызовов/день, read-only, 4 инструмента) / Pro $20/мес (write, 7 инструментов).
- **Зрелость:** коммерческий продукт; docs-only репозиторий (исходники не открыты).
- **Язык:** Python + 1С (расширение .cfe).
- **Ссылка:** https://github.com/rcs-kz/mcp-1c

#### 7. ARQA MCP Server для 1С (коммерческий)
- **Что:** on-premise MCP-сервер от ARQA (Казахстан), расширение в 1С (установка ~5 минут, без изменения конфигурации).
- **Транспорт:** HTTP, TLS, данные не покидают периметр.
- **Что отдаёт:** создание документов (счета, акты, накладные), поиск/выборка (контрагенты, документы, остатки), работа через права доступа 1С.
- **Зрелость:** коммерческий продукт; все конфигурации 8.3 (Бухгалтерия, УТ, ERP, ЗУП, КА, отраслевые).
- **Язык:** н/д (1С-расширение).
- **Ссылка:** https://arqa.cc/ru/mcp-server

### B. MCP-серверы для разработчика (EDT / VS Code)

#### 8. DitriXNew/EDT-MCP
- **Что:** MCP-сервер-плагин для 1C:EDT (Eclipse), даёт AI-ассистентам доступ к рабочему пространству EDT.
- **Транспорт:** Streamable HTTP + SSE (`:8765/mcp`); мульти-EDT прокси `edt-mcp-proxy` (`:8764`).
- **Что отдаёт:** информация о проектах, ошибки/предупреждения, BSL-анализ (модули, структура, чтение/запись методов, поиск кода, call hierarchy), валидация запросов 1С, рефакторинг метаданных, скриншоты форм, управление приложениями (запуск/отладка).
- **Зрелость:** проработанный (MCP 2025-11-25, e2e + protocol-conformance тесты), EDT 2026.1/2026.2.
- **Язык:** Java (Eclipse-плагин, Maven/Tycho).
- **Ссылка:** https://github.com/DitriXNew/EDT-MCP

#### 9. yellow-hammer/mcp-1c-platform-tools
- **Что:** MCP-сервер (npm `@yellow-hammer/mcp-1c-platform-tools`), публикует команды расширения VS Code «1C: Platform Tools» как MCP-инструменты для агентов Cursor/VS Code.
- **Транспорт:** stdio (клиент → MCP-сервер); сервер ↔ расширение по TCP `127.0.0.1:40241` (IPC) с токеном `ONEC_IPC_TOKEN`.
- **Что отдаёт:** команды 1c-platform-tools (project_init, deps_initPackagedef, env_status и др.). Метаданные намеренно НЕ публикуются (агент читает XML выгрузки сам).
- **Зрелость:** community-проект yellow-hammer (MIT), v0.1.5, активен.
- **Язык:** TypeScript/Node.js.
- **Ссылка:** https://github.com/yellow-hammer/mcp-1c-platform-tools

#### 10. 1c-syntax/bsl-language-server (MCP-режим)
- **Что:** BSL Language Server (сообщество 1c-syntax) может работать как MCP-сервер — анализ кода BSL/OneScript для AI-агентов, тот же движок, что и в LSP-режиме.
- **Транспорт:** stdio (по умолчанию), SSE, Streamable HTTP (`java -jar bsl-language-server.jar mcp --protocol ...`).
- **Что отдаёт:** анализ кода BSL (диагностика, парсинг, те же providers, что LSP). Workspaces через MCP roots.
- **Зрелость:** зрелый проект (LSP-стандарт де-факто для BSL), MCP-режим добавлен как доп. возможность.
- **Язык:** Java.
- **Ссылка:** https://1c-syntax.github.io/bsl-language-server/en/features/McpMode

### C. Синтаксис-помощники и RAG по справке/метаданным

#### 11. Antonio1C/1c-syntax-helper-mcp
- **Что:** MCP-сервер поиска по синтаксису 1С: парсит `shcntx_ru.hbk`, индексирует в Elasticsearch, отдаёт через FastAPI.
- **Транспорт:** HTTP (`:8000/mcp`); Docker (Elasticsearch + сервер).
- **Что отдаёт:** `search_1c_syntax` (функции/методы/объекты), `get_1c_function_details`, `get_1c_object_info`. MCP 2025-06-18.
- **Зрелость:** средняя (MIT, основа для нескольких форков).
- **Язык:** Python (FastAPI).
- **Ссылка:** https://github.com/Antonio1C/1c-syntax-helper-mcp

#### 12. IElagin/mcp-1c-syntax-assistant
- **Что:** форк Antonio1C: карточка элемента фиксированной формы (сигнатура, параметры с типами/обязательностью, возврат, контексты, версия, пример) вместо свободного текста; прямое чтение `.hbk` без 7-Zip.
- **Транспорт:** HTTP (`:8000/health`), Docker + Elasticsearch.
- **Что отдаёт:** 4 инструмента — `find_1c_help`, `get_1c_element`, `list_1c_object_members`, `get_1c_article`.
- **Зрелость:** проработанный (v2.4.3, ~23 491 документ, eval-замер качества поиска).
- **Язык:** Python.
- **Ссылка:** https://github.com/IElagin/mcp-1c-syntax-assistant

#### 13. axel-avb/mcp-1c-help
- **Что:** read-only MCP-сервер справки по синтаксису 1С (FastMCP), гибридный поиск BM25 + kNN (эмбеддинги) с RRF-слиянием.
- **Транспорт:** streamable-http (`:8002/mcp`), внешний Elasticsearch, опциональный embedding-сервис.
- **Что отдаёт:** 5 инструментов — `find_1c_help`, `get_quick_reference`, `get_syntax_info`, `list_object_members`, `search_by_context`.
- **Зрелость:** рабочее решение (Recall@5 ≈ 50% на семантическом индексе); read-only.
- **Язык:** Python (FastMCP).
- **Ссылка:** https://github.com/axel-avb/mcp-1c-help

#### 14. AzeevAN/mcp-1c
- **Что:** Python MCP-сервер-справочник: метаданные нескольких конфигураций + синтаксис платформы (с учётом версии конфигурации) + язык запросов.
- **Транспорт:** streamable-http и stdio; Docker (357MB).
- **Что отдаёт:** 10 инструментов — разрешение формулировки в точное имя объекта, поиск процедур по дампу кода, сигнатуры, структура объекта, `get_syntax` с версией платформы, таблица замен для старых платформ. Точность поиска 90–97%.
- **Зрелость:** проработанный (benchmark-стенд, P@k/MRR).
- **Язык:** Python.
- **Ссылка:** https://glama.ai/mcp/servers/AzeevAN/mcp-1c

#### 15. MCP 1C RAG Server (Infostart)
- **Что:** комплексное решение семантического поиска по всей конфигурации (метаданные + BSL-код + документация) с интеграцией в AI-ассистенты через MCP.
- **Транспорт:** HTTP JSON-RPC 2.0 (`:9000/mcp`).
- **Что отдаёт:** семантический поиск по коду/метаданным, анализ зависимостей процедур, 4 инструмента синтакс-помощника (глобальные функции, методы, свойства, детали синтаксиса).
- **Зрелость:** средняя (стек: Qdrant, embedding-сервис, Docker; 8 GB RAM).
- **Язык:** Python + Qdrant.
- **Ссылка:** https://infostart.ru/1c/tools/2616279/

#### 16. smirnov0ser/mcp1c_metadata
- **Что:** Docker-сервис, индексирующий JSON-выгрузку метаданных (из обработки ВыгрузитьМетаданныевJson) для быстрого поиска объектов.
- **Транспорт:** HTTP (`:8000/mcp`), Docker Hub-образ.
- **Что отдаёт:** единственный инструмент `metadatasearch` — поиск объекта по полному имени (`Справочники.Номенклатура`), поддержка нескольких конфигураций.
- **Зрелость:** узкий инструмент, низкая.
- **Язык:** Python.
- **Ссылка:** https://github.com/smirnov0ser/mcp1c_metadata

### D. Прочие / специализированные

#### 17. WSSAWER/1c_debug_mcp
- **Что:** standalone MCP-сервер для отладки работающей базы 1С через HTTP RDBG-сервер платформы (точки останова, стек вызовов, локальные переменные, вычисление выражений, шаги).
- **Транспорт:** stdio; MCP Control Center может публиковать через HTTP Gate.
- **Что отдаёт:** инструменты отладки (`check_debug_server`, `connect_debugger`, `set_breakpoints`, `get_call_stack`, `get_local_variables`, `evaluate_expression`, `continue_or_step`).
- **Зрелость:** нишевый; протокол RDBG адаптирован из MIT-проекта DevTool1C.
- **Язык:** Node.js.
- **Ссылка:** https://github.com/WSSAWER/1c_debug_mcp

#### 18. MCP-сервер для текстов модулей 1С (VendaMax / MCP_1C_modules)
- **Что:** Node.js MCP-сервер, читающий файлы выгрузки конфигурации (`DumpConfigToFiles`: BSL, TXT), чтобы AI видел код модулей (метаданные стандартные серверы не отдают).
- **Транспорт:** stdio (Cursor `mcp.json`, env `MODULES_DIR`).
- **Что отдаёт:** поиск строк, чтение процедур, анализ логики по коду выгрузки.
- **Зрелость:** простая демо-реализация (статья написана с помощью Grok).
- **Язык:** Node.js.
- **Ссылка:** https://infostart.ru/1c/tools/2458900/

#### 19. «Компакт MCP для 1С» (Infostart)
- **Что:** НЕ полноценный MCP-сервер — чат-обёртка над AI-моделями (через шлюз aitunnel), которая «донасыщает» запрос сведениями о конфигурации и правилами программирования 1С.
- **Транспорт:** чат (не MCP-протокол).
- **Что отдаёт:** генерация кода 1С с учётом компактной структуры метаданных (полная структура УНФ/Бухгалтерии не влезает в лимит токенов).
- **Зрелость:** личный инструмент, эксперимент.
- **Язык:** н/д.
- **Ссылка:** https://infostart.ru/1c/tools/2732462/

#### 20. iflow-mcp (alexmiawat-1c_mcp_server_own)
- **Что:** Python-прокси — адаптация/форк `vladimir-kharin/1c_mcp` (явно указан источник) с добавлением OAuth2 (password grant / authorization code + PKCE).
- **Транспорт:** stdio и HTTP (`/mcp`, `/sse`, `/health`, `/info`).
- **Что отдаёт:** список объектов метаданных, структура объекта, предопределённые элементы справочников.
- **Зрелость:** форк для личных нужд, MIT.
- **Язык:** Python.
- **Ссылка:** https://glama.ai/mcp/servers/iflow-mcp/alexmiawat-1c_mcp_server_own

---

## Сравнительная таблица

| # | Решение | Тип | Транспорт | Что отдаёт | Зрелость | Язык | Лицензия/доступ | Источник |
|---|---|---|---|---|---|---|---|---|
| 1 | vladimir-kharin/1c_mcp | сервер внутри 1С (расширение) + Python-прокси | HTTP `/hs/mcp/`; stdio/HTTP (прокси, OAuth2) | метаданные, кастомные tools/resources/prompts | высокая (v1.6.1, 2026-08) | 1С BSL + Python | MIT | github.com/vladimir-kharin/1c_mcp |
| 2 | artesk/1C_MCP_metadata | расширение 1С + PowerShell-мост | HTTP-сервис 1С; stdio (PowerShell) | структура метаданных, поиск объектов, валидация запросов | низкая (без обновлений >1 года) | 1С BSL + PowerShell | не указана | github.com/artesk/1C_MCP_metadata |
| 3 | ROCTUP/1c-mcp-toolkit | сервер (встр. HTTP в обработке) + Python-прокси | streamable-http `/mcp`, REST, Docker | метаданные, данные, execute_query | средняя-высокая (активен) | 1С BSL + Python | открытый | github.com/ROCTUP/1c-mcp-toolkit |
| 4 | DitriXNew/MCP-DB-Client (http1c) | native DLL (MCP-движок) + 1С | Streamable HTTP + SSE `/mcp` | полный MCP (tools/resources/prompts), RAG-поиск | высокая (полный протокол, security) | C++17 + 1С BSL + Rust | открытый | github.com/DitriXNew/MCP-DB-Client |
| 5 | INFATON MCP Server v2.1 | расширение 1С + Node-обёртка | HTTP (Basic Auth) + stdio | 51 инструмент (данные, метаданные, создание, CDC, аудит) | высокая (v2.1, 14.05.2026) | 1С BSL + Node.js | open-source | infostart.ru/1c/articles/2712602/ |
| 6 | RCS-kz/mcp-1c | коммерческий сервер (Python) + расширение .cfe | stdio/HTTP | чтение/создание документов, проводки | коммерческий (Solo free / Pro $20) | Python + 1С | проприетарная (PyArmor) | github.com/rcs-kz/mcp-1c |
| 7 | ARQA MCP Server | коммерческий on-premise сервер | HTTP, TLS | создание документов, поиск/выборка | коммерческий | 1С-расширение | проприетарная | arqa.cc/ru/mcp-server |
| 8 | DitriXNew/EDT-MCP | сервер-плагин 1C:EDT | Streamable HTTP + SSE (`:8765`) | проекты, BSL-анализ, рефакторинг, валидация запросов | высокая (e2e-тесты) | Java | открытый | github.com/DitriXNew/EDT-MCP |
| 9 | yellow-hammer/mcp-1c-platform-tools | MCP-сервер для VS Code/Cursor | stdio + IPC TCP `127.0.0.1:40241` | команды 1c-platform-tools (env, project_init…) | средняя (v0.1.5) | TypeScript/Node.js | MIT | github.com/yellow-hammer/mcp-1c-platform-tools |
| 10 | 1c-syntax/bsl-language-server (MCP mode) | MCP-режим LSP-сервера | stdio / SSE / streamable | анализ кода BSL/OneScript | высокая (зрелый LSP) | Java | открытый | 1c-syntax.github.io/bsl-language-server |
| 11 | Antonio1C/1c-syntax-helper-mcp | синтаксис-помощник (hbk→ES) | HTTP `:8000/mcp`, Docker | поиск по синтаксису (3 tools) | средняя (MIT, основа форков) | Python (FastAPI) | MIT | github.com/Antonio1C/1c-syntax-helper-mcp |
| 12 | IElagin/mcp-1c-syntax-assistant | синтаксис-помощник (форк) | HTTP, Docker + ES | карточка элемента (4 tools) | проработанный (v2.4.3) | Python | MIT (на базе) | github.com/IElagin/mcp-1c-syntax-assistant |
| 13 | axel-avb/mcp-1c-help | синтаксис-помощник (гибридный поиск) | streamable-http `:8002`, ES | 5 tools, BM25+kNN | рабочее (read-only) | Python (FastMCP) | открытый | github.com/axel-avb/mcp-1c-help |
| 14 | AzeevAN/mcp-1c | справочник (метаданные+синтаксис+запросы) | streamable-http + stdio, Docker | 10 tools, поиск 90–97% | проработанный (benchmark) | Python | открытый | glama.ai/mcp/servers/AzeevAN/mcp-1c |
| 15 | MCP 1C RAG Server | семантический поиск по конфигурации | HTTP `:9000/mcp` | RAG-поиск + 4 синтакс-tools | средняя | Python + Qdrant | открытый | infostart.ru/1c/tools/2616279/ |
| 16 | smirnov0ser/mcp1c_metadata | поиск по JSON-выгрузке метаданных | HTTP `:8000/mcp`, Docker | 1 tool (metadatasearch) | низкая | Python | открытый | github.com/smirnov0ser/mcp1c_metadata |
| 17 | WSSAWER/1c_debug_mcp | отладочный сервер (RDBG) | stdio | отладка: брейкпоинты, стек, переменные | нишевый | Node.js | MIT (частично) | github.com/WSSAWER/1c_debug_mcp |
| 18 | VendaMax/MCP_1C_modules | сервер по файлам выгрузки кода | stdio | тексты модулей (BSL) из выгрузки | демо | Node.js | открытый | infostart.ru/1c/tools/2458900/ |
| 19 | «Компакт MCP для 1С» | чат-обёртка (НЕ MCP-сервер) | чат (aitunnel) | генерация кода с учётом метаданных | эксперимент | н/д | н/д | infostart.ru/1c/tools/2732462/ |
| 20 | iflow-mcp 1c_mcp_server_own | Python-прокси (форк 1c_mcp) | stdio/HTTP + OAuth2 | метаданные, предопределённые элементы | форк (MIT) | Python | MIT | glama.ai/mcp/servers/iflow-mcp/... |

### Примечание о ложном срабатывании
`blockparty-global/1s-mcp` (npm `@one-source/mcp`) **НЕ является решением для 1С:Предприятие** — это MCP-сервер для блокчейн-платформы OneSource («1s» = OneSource, 30+ инструментов для EVM-сетей). Включено в поисковую выдачу по запросу «1s-mcp», но к 1С отношения не имеет. Также не относится к 1С прокси-агрегатор `slzcdhd/1mcp` и агрегатор `1mcp-app/agent` — это общие MCP-инфраструктурные инструменты без специфики 1С.

---

## Ключевые статьи Infostart (контекст и инструкции)

- **Владимир Харин. «1С и AI от хайпа к практике. Создаем MCP-сервер…»** — фундаментальная статья про архитектуру MCP для 1С (расширение + HTTP-сервис + Python-прокси, stdio/HTTP, OAuth2-проброс). https://infostart.ru/1c/articles/2514331/ + видео https://infostart.ru/video/w2514370/
- **«Cursor + MCP для 1С: работающая инструкция с картинками»** — практический цикл подключения Cursor через «1C: Platform Tools» + «1C: Platform Tools MCP» (IPC 127.0.0.1:40241, env_status, packagedef). https://infostart.ru/1c/articles/2800301/ (предыдущая: 2795749)
- **«MCP для 1С: когда ИИ начинает видеть структуру и данные»** — обзор практического применения 1C MCP Toolkit (ROCTUP). https://infostart.ru/1c/articles/2735285/
- **«1C Syntax MCP Server: Когда твой код не превращается в тыкву»** — подключение синтакс-MCP к OpenCode. https://infostart.ru/1c/articles/2657968/

---

## Источники

1. https://github.com/vladimir-kharin/1c_mcp | vladimir-kharin/1c_mcp — фреймворк MCP-серверов в 1С (расширение + Python-прокси) | community | 2026-08 (v1.6.1)
2. https://github.com/artesk/1C_MCP_metadata | artesk/1C_MCP_metadata — MCP-сервер метаданных (расширение + PowerShell) | community | 2024 (без обновлений >1 года)
3. https://github.com/ROCTUP/1c-mcp-toolkit | ROCTUP/1c-mcp-toolkit — MCP и REST API сервер для метаданных и данных 1С | community | 2026
4. https://github.com/DitriXNew/MCP-DB-Client | DitriXNew/MCP-DB-Client (http1c) — MCP-фреймворк на native DLL для 1С | community | 2026
5. https://github.com/DitriXNew/EDT-MCP | DitriXNew/EDT-MCP — MCP-сервер для 1C:EDT | community | 2026
6. https://github.com/yellow-hammer/mcp-1c-platform-tools | yellow-hammer/mcp-1c-platform-tools — MCP для команд 1c-platform-tools | community | 2026-02 (v0.1.5)
7. https://github.com/rcs-kz/mcp-1c | RCS-kz/mcp-1c — production MCP-сервер для 1С (коммерческий) | other | 2026
8. https://github.com/Antonio1C/1c-syntax-helper-mcp | Antonio1C/1c-syntax-helper-mcp — MCP-сервер поиска по синтаксису 1С | community | 2025
9. https://github.com/IElagin/mcp-1c-syntax-assistant | IElagin/mcp-1c-syntax-assistant — MCP-сервер синтаксис-справки (форк) | community | 2026
10. https://github.com/axel-avb/mcp-1c-help | axel-avb/mcp-1c-help — read-only MCP справки по синтаксису (гибридный поиск) | community | 2026
11. https://github.com/WSSAWER/1c_debug_mcp | WSSAWER/1c_debug_mcp — MCP-сервер отладки 1С через RDBG | community | 2026
12. https://github.com/smirnov0ser/mcp1c_metadata | smirnov0ser/mcp1c_metadata — Docker-сервис поиска по JSON-выгрузке метаданных | community | 2026
13. https://1c-syntax.github.io/bsl-language-server/en/features/McpMode | BSL Language Server — MCP mode | official (сообщество 1c-syntax) | 2026
14. https://infostart.ru/1c/articles/2712602/ | INFATON MCP Server v2.1 — 51 инструмент для AI-интеграции с 1С | secondary-industry | 14.05.2026
15. https://infostart.ru/1c/articles/2514331/ | «1С и AI от хайпа к практике. Создаем MCP-сервер…» (В. Харин) | secondary-industry | 2025
16. https://infostart.ru/1c/articles/2800301/ | «Cursor + MCP для 1С: работающая инструкция с картинками» | secondary-industry | 25.08.2025
17. https://infostart.ru/1c/articles/2735285/ | «MCP для 1С: когда ИИ начинает видеть структуру и данные» | secondary-industry | 2026
18. https://infostart.ru/1c/tools/2616279/ | MCP 1C RAG Server — семантический поиск по конфигурации | secondary-industry | 2026
19. https://infostart.ru/1c/tools/2458900/ | MCP-сервер для доступа к текстам модулей 1С (Node.js) | secondary-industry | 2025
20. https://infostart.ru/1c/tools/2732462/ | «Компакт MCP для 1С» | secondary-industry | 2026
21. https://infostart.ru/1c/articles/2657968/ | «1C Syntax MCP Server: Когда твой код не превращается в тыкву» | secondary-industry | 2025
22. https://arqa.cc/ru/mcp-server | ARQA — MCP Сервер для 1С (коммерческий) | other | 2026
23. https://glama.ai/mcp/servers/AzeevAN/mcp-1c | AzeevAN/mcp-1c — справочник метаданных+синтаксиса (10 tools) | community | 2026
24. https://glama.ai/mcp/servers/iflow-mcp/alexmiawat-1c_mcp_server_own | iflow-mcp — Python-прокси с OAuth2 (форк 1c_mcp) | community | 2026

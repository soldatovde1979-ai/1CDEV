# База знаний: MCP + 1С — исследование

**AS_OF (дата сборки):** 2026-10-14
**Среда:** Windows + VS Code + агент в VS Code + 1С:Предприятие
**Задача:** развернуть и подключить MCP-сервис, дающий агенту информацию о конфигурации 1С (выгрузка/загрузка, желательно автоматизированная), правила и язык разработки 1С. Это исследование и сбор базы знаний, НЕ проектирование.

---

## Ключевые выводы

- MCP — открытый протокол (JSON-RPC 2.0 + транспорт), текущая ревизия спецификации 2026-07-28; примитивы tools/resources/prompts (серверные) и roots/sampling/elicitation (клиентские, часть deprecated). Официальные SDK: TypeScript, Python, C#, Go, Rust (Tier 1), Java, Ruby (Tier 2), Kotlin, Swift, PHP (Tier 3).
- В VS Code MCP-серверы подключаются через разные агенты-расширения: Cline, Roo Code, Continue, GitHub Copilot (агентный режим), Cody, Cursor. Ключевая ловушка — разные имена ключа конфигурации (`servers` у VS Code/Cody vs `mcpServers` у Cline/Roo/Cursor/Copilot CLI) и разные написания транспорта.
- Выгрузку/загрузку конфигурации 1С можно автоматизировать: пакетный режим 1cv8 (/DumpConfigToFiles и др.), EDT + 1cedtcli/ring, 1С:ГитКонвертер, gitsync, v8unpack, OneScript, COM-соединение. Для «git-дружелюбного» хранения конфигурации — формат выгрузки в файлы (XML) + gitsync/EDT.
- Готовые MCP-решения для 1С существуют, но единого стандарта нет: ~20 проектов, 7 архитектурных подходов (расширение+HTTP-сервис /hs/mcp/, .epf с HTTP-сервером, native DLL, серверы по файлам выгрузки, EDT-MCP, синтаксис-помощники, коммерческие). Самые зрелые: vladimir-kharin/1c_mcp, ROCTUP/1c-mcp-toolkit, DitriXNew (http1c + EDT-MCP).
- Правила разработки 1С: де-факто стандарт сообщества — BSL Language Server (1c-syntax, 130+ диагностик, есть MCP-режим); официальный норматив — 1C-Company/v8-code-style (для 1C:EDT) и its.1c.ru/db/v8std. Репозиторий silverbulleters/standards удалён — его содержимое влилось в 1c-syntax.
- Язык 1С (BSL): официальная документация на ИТС; каноническая грамматика — 1c-syntax (TextMate + ANTLR4). Прямой мост 1С↔MCP — bsl-language-server в режиме `--mcp`.

---

## Индекс файлов

| Файл | Тема | Источников |
|---|---|---|
| [01-mcp-protocol.md](01-mcp-protocol.md) | Протокол MCP: архитектура, примитивы, транспорт, SDK, версии | 17 |
| [02-mcp-vscode.md](02-mcp-vscode.md) | MCP-клиенты/агенты в VS Code и их конфигурация | 12 |
| [03-1c-config-export.md](03-1c-config-export.md) | Выгрузка/загрузка конфигурации 1С и автоматизация | 15 |
| [04-1c-mcp-solutions.md](04-1c-mcp-solutions.md) | Существующие MCP-решения для 1С (сравнительная таблица) | 24 |
| [05-1c-dev-standards.md](05-1c-dev-standards.md) | Правила/стандарты разработки 1С на GitHub (таблица для выбора) | 17 |
| [06-1c-language-bsl.md](06-1c-language-bsl.md) | Язык 1С (BSL) и его инструменты | 11 |
| [SKILL.md](SKILL.md) | Сам скилл исследования (адаптирован под этот проект) | — |

---

## Сквозные находки и пробелы (gap)

- Единого/официального MCP-сервера от фирмы 1С нет — вся экосистема сообщества.
- Нет единого стандарта авторизации в 1С-MCP-серверах (Basic-Auth vs OAuth2, прокси-прокладки).
- Функциональность (метаданные / данные / выполнение кода / отладка) размазана по разным проектам, единого универсального сервера нет.
- Репозитории `silverbulleters/standards` и `1c-syntax/bsl` из первоначальных предположений не существуют (404) — стандарты влиты в 1c-syntax, грамматика языка в 1c-syntax/1c-syntax (TextMate) и bsl-parser (ANTLR4).
- gitsync живёт в `oscript-library/gitsync` (не silverbulleters); v8unpack — `e8tools/v8unpack` (C++) и `saby-integration/v8unpack` (Python).
- Прямой кандидат на мост 1С↔MCP для этого проекта: bsl-language-server (режим `--mcp`) + один из серверов метаданных по файлам выгрузки.

---

## Сводный реестр источников — 92 уникальных

Формат: `№. URL — название (тип, дата) [файл]`.

1. https://modelcontextprotocol.io — Официальный сайт MCP (документация и спецификация) (official, 2024 (постоянно обновляется)) [01]
2. https://modelcontextprotocol.io/specification/2026-07-28/architecture — Спецификация: Архитектура (ревизия 2026-07-28) (official, 2026-07-28) [01]
3. https://modelcontextprotocol.io/specification/2026-07-28/basic/transports — Спецификация: Транспорты (official, 2026-07-28) [01]
4. https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning — Спецификация: Версионирование и совместимость (official, 2026-07-28) [01]
5. https://modelcontextprotocol.io/specification/2026-07-28/changelog — Спецификация: Changelog ревизии (official, 2026-07-28) [01]
6. https://modelcontextprotocol.io/docs/2026-07-28/sdk — Список официальных SDK и их уровней (tiers) (official, 2026-07-28) [01]
7. https://github.com/modelcontextprotocol/modelcontextprotocol — Репозиторий спецификации MCP (schema + docs + blog) (official, 2024 (репозиторий создан 2024-09-24)) [01]
8. https://github.com/modelcontextprotocol/typescript-sdk — Официальный TypeScript SDK (official, 2024 (запущен с анонсом)) [01]
9. https://github.com/modelcontextprotocol/python-sdk — Официальный Python SDK (official, 2024 (запущен с анонсом)) [01]
10. https://github.com/modelcontextprotocol/java-sdk — Официальный Java SDK (совместно со Spring AI) (official, 2025) [01]
11. https://github.com/modelcontextprotocol/kotlin-sdk — Официальный Kotlin SDK (совместно с JetBrains) (official, 2025) [01]
12. https://github.com/modelcontextprotocol/csharp-sdk — Официальный C# SDK (совместно с Microsoft) (official, 2025) [01]
13. https://anthropic.com/news/model-context-protocol — Анонс MCP от Anthropic («Introducing the Model Context Protocol») (official, 2024-11-25) [01]
14. https://blog.modelcontextprotocol.io/posts/2026-07-28 — Официальный анонс ревизии 2026-07-28 (official, 2026-07-28) [01]
15. https://devblogs.microsoft.com/dotnet/announcing-v20-of-the-official-mcp-csharp-sdk/ — Microsoft DevBlogs: анонс v2.0 C# SDK под ревизию 2026-07-28 (secondary-industry, 2026-07) [01]
16. https://blog.cloudflare.com/mcp-v2/ — Cloudflare: «The next generation of MCP» (разбор stateless-ревизии) (secondary-industry, 2026-07) [01]
17. https://en.wikipedia.org/wiki/Model_Context_Protocol — Википедия: обзорная статья о MCP (secondary-industry, 2024 (постоянно обновляется)) [01]
18. https://docs.cline.bot/mcp/mcp-overview — Cline — MCP (обзор и настройка серверов) (official, 2026-10-14 (дата обращения)) [02]
19. https://docs.roocode.com/features/mcp/using-mcp-in-roo — Roo Code — Using MCP in Roo (глобальная/проектная конфигурация, транспорты) (official, 2026-10-14 (дата обращения)) [02]
20. https://docs.continue.dev/customize/deep-dives/mcp — Continue — How to Set Up Model Context Protocol (MCP) (official, 2026-10-14 (дата обращения)) [02]
21. https://code.visualstudio.com/docs/agents/reference/mcp-configuration — VS Code — MCP configuration reference (форматы `servers`/`mcpServers`, транспорты) (official, 2026-10-14 (дата обращения)) [02]
22. https://code.visualstudio.com/docs/copilot/customization/mcp-servers — VS Code — GitHub Copilot MCP servers (агентный режим) (official, 2026-10-14 (дата обращения)) [02]
23. https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/configure-mcp-servers — GitHub Docs — Configure MCP servers for your repository (cloud agent / code review) (official, 2026-10-14 (дата обращения)) [02]
24. https://sourcegraph.com/docs/cody/capabilities/mcp — Sourcegraph — Cody MCP (official, 2026-10-14 (дата обращения)) [02]
25. https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-cline.md — GitHub MCP Server — installation guide: Cline (пример streamableHttp) (official, 2026-10-14 (дата обращения)) [02]
26. https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-roo-code.md — GitHub MCP Server — installation guide: Roo Code (пример streamable-http) (official, 2026-10-14 (дата обращения)) [02]
27. https://preprints.org/manuscript/202506.0821/v1 — A Comparative Featureset Analysis of Agentic IDE Tools (MCP-поддержка Copilot/Cursor/Cody/Cline и др.) (academic, 2025-06) [02]
28. https://mcpnav.dev/learn/add-mcp-server-to-cursor — mcpnav — Add an MCP server to Cursor (файлы, транспорты, интерполяция) (community, 2026-10-14 (дата обращения)) [02]
29. https://stackmcp.dev/blog/setup-mcp-servers-continue — StackMCP — How to Set Up MCP Servers in Continue.dev (YAML vs JSON) (secondary-industry, 2026-10-14 (дата обращения)) [02]
30. https://v8.1c.ru/platforma/vygruzka-i-zagruzka-konfiguracii-v-fayly/ — Выгрузка и загрузка конфигурации в файлы (платформа 1С:Предприятие) (official, н/д (актуально для 8.3.x)) [03]
31. https://kb.1ci.com/1C_Enterprise_Platform/Guides/Administrator_Guides/1C_Enterprise_8.3.24_Administrator_Guide/Appendix_7._The_parameters_of_the_command_line_to_launch_1C_Enterprise/7.4._Running_Designer_in_batch_mode/7.4.4._Configurations_and_extensions — Руководство администратора 8.3.24, прил. 7.4.4 — параметры командной строки (DumpCfg/LoadCfg/DumpConfigToFiles/LoadConfigFromFiles) (official, 2024) [03]
32. https://its.1c.ru/db/content/edtdoc/src/topics/t000454.html — EDT: Руководство разработчика (создание ИБ, загрузка .cf/.dt, выгрузка/загрузка XML) (official, н/д) [03]
33. https://edt.1c.ru/dev/edt/2024.2/apidocs/com/_1c/g5/v8/dt/platform/services/core/runtimes/execution/impl/IRuntimeExecutionCommandOptions.html — EDT API 2024.2 — IRuntimeExecutionCommandOptions (DUMP_CONFIG_TO_FILES, LOAD_CONFIGURATION_FROM_XML, LOAD_CONFIGURATION_FROM_CF) (official, 2024) [03]
34. https://its.1c.ru/db/metod8dev/content/5937/hdoc — 1С:ГитКонвертер — конвертация истории хранилища в Git + переход на EDT (official, н/д) [03]
35. https://its.1c.ru/db/content/edtdoc/src/topics/i061.html — EDT: Руководство — конфигурация 1С:ГитКонвертер (этапы конвертации, Git LFS) (official, н/д) [03]
36. https://github.com/1C-Company/GitConverter — 1C-Company/GitConverter — официальный репозиторий 1С:ГитКонвертер (official, 2017–2024 (активно)) [03]
37. https://github.com/oscript-library/gitsync — gitsync — синхронизация хранилища 1С с git (бывш. silverbulleters/gitsync, MPL 2.0) (community, создан 2016, активен) [03]
38. https://github.com/EvilBeaver/OneScript — OneScript — кроссплатформенная VM скриптов на языке 1С (community, активен (v2.x, .NET 8)) [03]
39. https://github.com/e8tools/v8unpack — e8tools/v8unpack — gcc-порт оригинального v8Unpack (Denis Demidov) (community, н/д (последний релиз под 8.3.16)) [03]
40. https://github.com/saby-integration/v8unpack — saby-integration/v8unpack — Python-утилита человекочитаемой распаковки cf/cfe/epf (community, 2019–2024 (активно)) [03]
41. https://infostart.ru/1c/articles/903269/ — Инфостарт: Синхронизация хранилища 1С и git-репозитория с OneScript и Gitsync (secondary-industry, 2018) [03]
42. https://infostart.ru/1c/articles/2438038/ — Инфостарт: Работаем с GIT без хранилища и EDT (ibcmd/1cedtcli, gitsync) (secondary-industry, 2024) [03]
43. https://oscript.io/learn — OneScript: Что такое OneScript (официальный сайт) (community, н/д (актуально)) [03]
44. https://releases.1c.ru/total — Сайт релизов 1С (дистрибутивы платформы, конфигураций, EDT) (official, н/д (постоянно обновляется)) [03]
45. https://github.com/vladimir-kharin/1c_mcp — vladimir-kharin/1c_mcp — фреймворк MCP-серверов в 1С (расширение + Python-прокси) (community, 2026-08 (v1.6.1)) [04]
46. https://github.com/artesk/1C_MCP_metadata — artesk/1C_MCP_metadata — MCP-сервер метаданных (расширение + PowerShell) (community, 2024 (без обновлений >1 года)) [04]
47. https://github.com/ROCTUP/1c-mcp-toolkit — ROCTUP/1c-mcp-toolkit — MCP и REST API сервер для метаданных и данных 1С (community, 2026) [04]
48. https://github.com/DitriXNew/MCP-DB-Client — DitriXNew/MCP-DB-Client (http1c) — MCP-фреймворк на native DLL для 1С (community, 2026) [04]
49. https://github.com/DitriXNew/EDT-MCP — DitriXNew/EDT-MCP — MCP-сервер для 1C:EDT (community, 2026) [04]
50. https://github.com/yellow-hammer/mcp-1c-platform-tools — yellow-hammer/mcp-1c-platform-tools — MCP для команд 1c-platform-tools (community, 2026-02 (v0.1.5)) [04]
51. https://github.com/rcs-kz/mcp-1c — RCS-kz/mcp-1c — production MCP-сервер для 1С (коммерческий) (other, 2026) [04]
52. https://github.com/Antonio1C/1c-syntax-helper-mcp — Antonio1C/1c-syntax-helper-mcp — MCP-сервер поиска по синтаксису 1С (community, 2025) [04]
53. https://github.com/IElagin/mcp-1c-syntax-assistant — IElagin/mcp-1c-syntax-assistant — MCP-сервер синтаксис-справки (форк) (community, 2026) [04]
54. https://github.com/axel-avb/mcp-1c-help — axel-avb/mcp-1c-help — read-only MCP справки по синтаксису (гибридный поиск) (community, 2026) [04]
55. https://github.com/WSSAWER/1c_debug_mcp — WSSAWER/1c_debug_mcp — MCP-сервер отладки 1С через RDBG (community, 2026) [04]
56. https://github.com/smirnov0ser/mcp1c_metadata — smirnov0ser/mcp1c_metadata — Docker-сервис поиска по JSON-выгрузке метаданных (community, 2026) [04]
57. https://1c-syntax.github.io/bsl-language-server/en/features/McpMode — BSL Language Server — MCP mode (official (сообщество 1c-syntax), 2026) [04]
58. https://infostart.ru/1c/articles/2712602/ — INFATON MCP Server v2.1 — 51 инструмент для AI-интеграции с 1С (secondary-industry, 14.05.2026) [04]
59. https://infostart.ru/1c/articles/2514331/ — «1С и AI от хайпа к практике. Создаем MCP-сервер…» (В. Харин) (secondary-industry, 2025) [04]
60. https://infostart.ru/1c/articles/2800301/ — «Cursor + MCP для 1С: работающая инструкция с картинками» (secondary-industry, 25.08.2025) [04]
61. https://infostart.ru/1c/articles/2735285/ — «MCP для 1С: когда ИИ начинает видеть структуру и данные» (secondary-industry, 2026) [04]
62. https://infostart.ru/1c/tools/2616279/ — MCP 1C RAG Server — семантический поиск по конфигурации (secondary-industry, 2026) [04]
63. https://infostart.ru/1c/tools/2458900/ — MCP-сервер для доступа к текстам модулей 1С (Node.js) (secondary-industry, 2025) [04]
64. https://infostart.ru/1c/tools/2732462/ — «Компакт MCP для 1С» (secondary-industry, 2026) [04]
65. https://infostart.ru/1c/articles/2657968/ — «1C Syntax MCP Server: Когда твой код не превращается в тыкву» (secondary-industry, 2025) [04]
66. https://arqa.cc/ru/mcp-server — ARQA — MCP Сервер для 1С (коммерческий) (other, 2026) [04]
67. https://glama.ai/mcp/servers/AzeevAN/mcp-1c — AzeevAN/mcp-1c — справочник метаданных+синтаксиса (10 tools) (community, 2026) [04]
68. https://glama.ai/mcp/servers/iflow-mcp/alexmiawat-1c_mcp_server_own — iflow-mcp — Python-прокси с OAuth2 (форк 1c_mcp) (community, 2026) [04]
69. https://github.com/1c-syntax/bsl-language-server — BSL Language Server — реализация LSP для 1C (BSL), диагностики стандарта (community (1c-syntax), 2026-10-13) [05]
70. https://github.com/1c-syntax/vsc-language-1c-bsl — VS Code: Language 1C (BSL) — синтаксис + интеграция BSL LS (community (1c-syntax), 2026-10-09) [05]
71. https://github.com/1c-syntax/sonar-bsl-plugin-community — SonarQube 1C (BSL) Community Plugin (community (1c-syntax), 2026-08-27) [05]
72. https://github.com/1c-syntax/bsl-parser — ANTLR-грамматики BSL и SDBL (community (1c-syntax), 2026-10-13) [05]
73. https://github.com/1c-syntax/mdclasses — Разбор метаданных конфигураций 1С (community (1c-syntax), 2026-10-12) [05]
74. https://github.com/1c-syntax/claude-code-bsl-lsp — Claude Code plugin — BSL Language Server LSP для агента (community (1c-syntax), 2026-04-22) [05]
75. https://github.com/1C-Company/v8-code-style — «1С:Стандарты разработки V8» — плагин 1C:EDT (official (фирма 1С), 2026-10-13) [05]
76. https://github.com/1C-Company/dt-project-checks — 1C:EDT Project checks — структурная целостность (official (фирма 1С), 2026-09-29) [05]
77. https://its.1c.ru/db/v8std — Стандарты и методики разработки (ИТС 1С) (official (фирма 1С), постоянно) [05]
78. https://github.com/Pr-Mex/vanessa-automation — BDD в 1С — сценарное тестирование (Gherkin) (community, 2026-10-06) [05]
79. https://github.com/vanessa-opensource/add — Vanessa-ADD — TDD/BDD для 1С (community, 2024-12-20) [05]
80. https://github.com/xDrivenDevelopment/precommit1c — Precommit1C — git pre-commit хуки для 1С (community, 2021-07-05 (заморожен)) [05]
81. https://github.com/otymko/phoenixbsl — Phoenix BSL — анализ/форматирование в Конфигураторе (community, 2024-10-23) [05]
82. https://github.com/otymko/bslls-connector-for-edt — Коннектор BSL LS для 1C:EDT (community, 2026-02-07) [05]
83. https://github.com/kuzyara/CodeStyle1C — Соглашения по стилю кода 1С (community, 2026-08-28) [05]
84. https://github.com/ones-devguide/dev-rules — Набор практик и стандартов для разработки в 1С (secondary-industry, 2025-12-29) [05]
85. https://github.com/silverbulleters/standards — Стандарты разработки SilverBulleters (перенесены в 1c-syntax) (community (historical), 404 — репозиторий удалён/перенесён) [05]
86. https://its.1c.ru/db/v83doc — 1С:Предприятие 8.3. Документация (актуальная версия) (official, 2026-10-14) [06]
87. https://its.1c.ru/db/v838doc/bookmark/dev/TI000000116 — Глава 4. Встроенный язык :: Руководство разработчика (official, 2026-10-14) [06]
88. https://its.1c.ru/db/content/v8323doc/src/руководство%20администратора/введение.htm — Введение (структура документации: Руководство разработчика, Описание встроенного языка, синтакс-помощник) (official, 2026-10-14) [06]
89. https://github.com/1c-syntax/1c-syntax — Зонтичный репозиторий грамматики подсветки (TextMate/YAML-tmLanguage) (community, 2026-10-14) [06]
90. https://1c-syntax.github.io/bsl-language-server — Документация BSL Language Server (capabilities, McpMode) (community, 2026-10-14) [06]
91. https://edt.1c.ru/dev — 1C:EDT public API documentation (official, 2026-10-14) [06]
92. https://edt.1c.ru/dev/ru/docs/plugins/dev/lang/bsl-model/ — Модель встроенного языка в 1C:EDT (Xtext) (official, 2026-10-14) [06]


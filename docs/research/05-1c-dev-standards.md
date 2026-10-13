# Правила и стандарты разработки 1С — обзор репозиториев и сравнительная таблица

> Исследование для выбора базы правил/стандартов при подключении MCP к 1С (Windows + VS Code + агент + 1С:Предприятие).
> Дата сбора данных: 2026-10-14. Звёзды и даты последней активности получены через GitHub REST API (реальные значения на дату сбора).

## Ключевой вывод

Правила разработки 1С существуют в двух «слоях», которые на практике объединяются:

1. **Официальные стандарты и методики фирмы «1С»** — `its.1c.ru/db/v8std` (текст стандартов) + плагин **1C-Company/v8-code-style** («1С:Стандарты разработки V8» для 1C:EDT), который автоматизирует их проверку.
2. **Стандарт сообщества (де-факто)** — реализован как набор диагностик в **BSL Language Server** (`1c-syntax/bsl-language-server`, диагностики с тегом `STANDARD`). Именно его тянут за собой VS Code, SonarQube, линтеры и MCP-интеграции.

⚠️ **Важно:** репозиторий `github.com/silverbulleters/standards`, указанный в задании как первоисточник, **больше не существует** (GitHub API возвращает `404 Not Found`). Стандарты SilverBulleters были перенесены в организацию `1c-syntax` и влились в диагностики BSL Language Server (тег `STANDARD`), а параллельно существует официальный стандарт фирмы «1С» на ИТС. Ниже это отражено в таблице и списке источников.

---

## Сравнительная таблица (для выбора)

| Репозиторий / набор правил | Что даёт | Язык | Активность (звёзды / последний коммит) | Лицензия | Как применять / установить |
|---|---|---|---|---|---|
| **1c-syntax/bsl-language-server** (BSL Language Server) | Ядро стандарта сообщества: 130+ диагностик качества кода (в т.ч. с тегом `STANDARD`), форматирование, переходы к определению, quick-fixes, когнитивная/цикломатическая сложность. Режимы: LSP, batch-анализ (CI), MCP-сервер | Java | ~490 ⭐ / 2026-10-13 (активно) | LGPL-3.0 (COPYING.md) | Скачать бинарник с Releases, добавить в PATH; конфиг `.bsl-language-server.json`; CLI `java -jar bsl-language-server.jar analyze/format`; подключается через LSP-плагины |
| **1C-Company/v8-code-style** («1С:Стандарты разработки V8») | Официальный плагин фирмы «1С»: автоматические проверки соответствия стандартам/методикам 1С (имена общих модулей, типизация кода, пустые методы, транзакции и т.д.). Входит в дистрибутив 1C:EDT 2021.3+ | Java (Eclipse/EDT) | ~241 ⭐ / 2026-10-13 (активно) | EPL-2.0 | Встроен в 1C:EDT 2021.3+; отдельно — p2-репозиторий для установки в EDT. Только для EDT, не для VS Code |
| **its.1c.ru/db/v8std** (Стандарты и методики разработки, ИТС) | Первоисточник правил: тексты стандартов фирмы «1С» (именование, исключения, транзакции, запросы и пр.). Человекочитаемый норматив | Русский (документ) | Официально, постоянно обновляется | проприетарный доступ (ИТС) | Читать/цитировать как источник правил; автоматизируется через v8-code-style |
| **1c-syntax/sonar-bsl-plugin-community** | SonarQube-плагин для 1С/OneScript: подсветка, метрики (ncloc, CPD, цикломатическая сложность), диагностики BSL LS как правила, импорт JSON-отчётов | Java | ~264 ⭐ / 2026-08-27 (активно) | LGPL-3.0 | Скачать jar с Releases → `$SONARQUBE_HOME/extensions/plugins` → перезапуск; анализ через `sonar-scanner` + `sonar-project.properties` |
| **1c-syntax/vsc-language-1c-bsl** | Официальное расширение VS Code для 1С: синтаксис + интеграция BSL Language Server (диагностики, форматирование, навигация) | TypeScript | ~193 ⭐ / 2026-10-09 (активно) | MIT | Установить из VS Code Marketplace: «Language 1C (BSL)» (id: `1c-syntax.language-1c-bsl`) |
| **1c-syntax/bsl-parser** | ANTLR4-грамматики BSL и SDBL (язык запросов) — основа парсинга для BSL LS и сторонних инструментов | 1C Enterprise (ANTLR) | ~64 ⭐ / 2026-10-13 (активно) | GPL-3.0 | Библиотека для встраивания парсера; не используется напрямую при выборе правил |
| **1c-syntax/mdclasses** | Разбор метаданных конфигураций/расширений (XML) — основа для проверок структуры проекта | Java | ~67 ⭐ / 2026-10-12 (активно) | LGPL-3.0 | Библиотека; используется внутри BSL LS / SonarQube-плагина |
| **1c-syntax/claude-code-bsl-lsp** | Плагин Claude Code: подключает BSL LS как LSP к AI-агенту (диагностики, форматирование, автозагрузка бинарника). **Релевантно для связки агент+MCP** | — (обёртка) | ~35 ⭐ / 2026-04-22 | MIT | `claude /plugin marketplace add 1c-syntax/claude-code-bsl-lsp` → `/plugin install bsl-language-server@bsl-language-server` |
| **Pr-Mex/vanessa-automation** | Сценарное/BDD-тестирование на Gherkin (`.feature`), отчёты Allure, автодокументация. Не «правила стиля», а правила проверки поведения | 1C Enterprise | ~718 ⭐ / 2026-10-06 (активно) | BSD-3-Clause | `opm install vanessa-automation` (OneScript) или готовые epf-обработки с Releases |
| **vanessa-opensource/add** (Vanessa-ADD) | TDD/BDD-инструментарий качества: юнит-тесты, дымовые тесты, поведенческие сценарии. Наследник xUnitFor1C + Vanessa-Behavior | 1C Enterprise | ~383 ⭐ / 2024-12-20 | MPL-2.0 | `opm install add`; сборка epf через OneScript + 1С:Предприятие |
| **xDrivenDevelopment/precommit1c** | Git pre-commit хуки: разборка/сборка внешних обработок (epf→xml), синтаксис-контроль, кодировка UTF-8, проверки структуры модулей | 1C Enterprise | ~246 ⭐ / 2021-07-05 (заморожен) | Apache-2.0 | `opm install precommit1c` + `.pre-commit-config.yaml`; старый, частично вытеснен BSL LS |
| **otymko/phoenixbsl** | Анализ и форматирование кода 1С прямо в Конфигураторе (Ctrl+I анализ, Ctrl+K формат), на базе BSL LS | Java | ~317 ⭐ / 2024-10-23 | Apache-2.0 | msi из Releases (Windows, запуск от администратора) или `java -jar phoenix-*.jar` (Java 11+) |
| **otymko/bslls-connector-for-edt** | Подключает BSL LS (128+ диагностик) в среду 1C:EDT — аналог v8-code-style, но на диагностиках сообщества | Java | ~133 ⭐ / 2026-02-07 (активно) | MIT | Update site `https://otymko.github.io/bslls-connector-for-edt/update/.../latest/` в EDT |
| **kuzyara/CodeStyle1C** | Соглашения по стилю кода 1С (префиксы переменных, именование, транзакции, запросы, перевод имён в латиницу). Практическое руководство, ссылается на v8std и BSL LS | Markdown | ~100 ⭐ / 2026-08-28 | нет (без лицензии) | Читать как чек-лист для code-review; тренажёр codereview1c.ru |
| **ones-devguide/dev-rules** | Живой регламент разработки: процессы, код-ревью, библиотека проектных решений (PSL), теория. Готовый шаблон командного регламента | Markdown (docusaurus) | ~77 ⭐ / 2025-12-29 | CC-BY-SA-4.0 | Клонировать/форкнуть как основу регламента команды; сайт ones-devguide.github.io/dev-rules |
| **1C-Company/dt-project-checks** | Официальные проверки структурной целостности проектов 1C:EDT (метаданные, формы, пути данных) | Java (Eclipse/EDT) | ~29 ⭐ / 2026-09-29 | EPL-2.0 | Встроен в 1C:EDT; отдельный p2-репозиторий |

---

## Как выбрать (рекомендации под контекст задачи)

- **Стандарт как «источник истины» для кода** → BSL Language Server (`1c-syntax/bsl-language-server`): самый живой, машинопроверяемый, уже имеет MCP-режим и LSP-интеграции для VS Code.
- **Если целевая IDE — VS Code** → `vsc-language-1c-bsl` (тянет BSL LS автоматически). Плагин `v8-code-style` тут **не работает** (он только для 1C:EDT).
- **Если нужна проверка в CI/SonarQube** → `sonar-bsl-plugin-community` (бесплатный) или коммерческий 1C (BSL) Plugin от SilverBulleters.
- **Официальный норматив + EDT** → `v8-code-style` + `its.1c.ru/db/v8std`.
- **Для связки «агент/MCP»** → BSL LS уже умеет `--mcp`; готовый адаптер для Claude Code — `claude-code-bsl-lsp`.
- **Правила поведения/приёмка** (не стиль кода) → `vanessa-automation` (BDD) / `vanessa-opensource/add` (TDD+BDD).
- **Командный регламент (люди)** → `ones-devguide/dev-rules` (шаблон) + `kuzyara/CodeStyle1C` (чек-лист ревью).

---

## Источники

| № | URL | Название | Тип | Дата (активность/доступ) |
|---|---|---|---|---|
| 1 | https://github.com/1c-syntax/bsl-language-server | BSL Language Server — реализация LSP для 1C (BSL), диагностики стандарта | community (1c-syntax) | 2026-10-13 |
| 2 | https://github.com/1c-syntax/vsc-language-1c-bsl | VS Code: Language 1C (BSL) — синтаксис + интеграция BSL LS | community (1c-syntax) | 2026-10-09 |
| 3 | https://github.com/1c-syntax/sonar-bsl-plugin-community | SonarQube 1C (BSL) Community Plugin | community (1c-syntax) | 2026-08-27 |
| 4 | https://github.com/1c-syntax/bsl-parser | ANTLR-грамматики BSL и SDBL | community (1c-syntax) | 2026-10-13 |
| 5 | https://github.com/1c-syntax/mdclasses | Разбор метаданных конфигураций 1С | community (1c-syntax) | 2026-10-12 |
| 6 | https://github.com/1c-syntax/claude-code-bsl-lsp | Claude Code plugin — BSL Language Server LSP для агента | community (1c-syntax) | 2026-04-22 |
| 7 | https://github.com/1C-Company/v8-code-style | «1С:Стандарты разработки V8» — плагин 1C:EDT | official (фирма 1С) | 2026-10-13 |
| 8 | https://github.com/1C-Company/dt-project-checks | 1C:EDT Project checks — структурная целостность | official (фирма 1С) | 2026-09-29 |
| 9 | https://its.1c.ru/db/v8std | Стандарты и методики разработки (ИТС 1С) | official (фирма 1С) | постоянно |
| 10 | https://github.com/Pr-Mex/vanessa-automation | BDD в 1С — сценарное тестирование (Gherkin) | community | 2026-10-06 |
| 11 | https://github.com/vanessa-opensource/add | Vanessa-ADD — TDD/BDD для 1С | community | 2024-12-20 |
| 12 | https://github.com/xDrivenDevelopment/precommit1c | Precommit1C — git pre-commit хуки для 1С | community | 2021-07-05 (заморожен) |
| 13 | https://github.com/otymko/phoenixbsl | Phoenix BSL — анализ/форматирование в Конфигураторе | community | 2024-10-23 |
| 14 | https://github.com/otymko/bslls-connector-for-edt | Коннектор BSL LS для 1C:EDT | community | 2026-02-07 |
| 15 | https://github.com/kuzyara/CodeStyle1C | Соглашения по стилю кода 1С | community | 2026-08-28 |
| 16 | https://github.com/ones-devguide/dev-rules | Набор практик и стандартов для разработки в 1С | secondary-industry | 2025-12-29 |
| 17 | https://github.com/silverbulleters/standards | Стандарты разработки SilverBulleters (перенесены в 1c-syntax) | community (historical) | 404 — репозиторий удалён/перенесён |

**Итого источников: 17** (≥8 требуемых). Все URL проверены через GitHub API и/или поисковую выдачу; метрики (звёзды, даты push, лицензии) — реальные значения с GitHub REST API на 2026-10-14.

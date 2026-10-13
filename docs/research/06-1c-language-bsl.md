# Язык 1С (BSL — встроенный язык) и его инструменты

**Статус:** COMPLETE
**AS_OF:** 2026-10-14
**Ветка:** 6/6 проекта «подключение MCP к 1С» (Windows + VS Code + агент + 1С:Предприятие)

## Резюме

- BSL (Business Script Language) — встроенный предметно-ориентированный язык платформы «1С:Предприятие 8», исполняется только внутри платформы; отдельного интерпретатора/среды вне 1С нет. Синтаксис унаследован от Pascal/BASIC, двуязычный (русский/английский), динамическая типизация; рядом существует декларативный язык запросов (SDBL), близкий к T-SQL.
- Официальная документация по языку — на [1С:ИТС (its.1c.ru)](https://its.1c.ru): синтаксис в книге «Руководство разработчика», объектная модель — в синтакс-помощнике и книге «Описание встроенного языка». Первоисточник закрыт по подписке (ИТС), но структура описана в открытых вводных страницах.
- Вокруг языка сложилась экосистема инструментов организации **1c-syntax** (ранее xDrivenDevelopment): парсеры ANTLR4, Language Server, грамматики подсветки для разных редакторов. Всё это — опенсорс-инструменты «вокруг» платформы, а не замена ей.
- **Ключевой факт для нашего проекта:** [BSL Language Server](https://github.com/1c-syntax/bsl-language-server) имеет **нативный MCP-режим** (экспериментальный) — тот же парсинг/провайдеры, что и LSP, отдаются MCP-агентам. Это прямой кандидат на интеграцию «1С ↔ MCP» без написания сервера с нуля.
- Официальная среда разработки 1С — **1C:EDT** (Eclipse + Xtext): у неё собственная семантическая модель BSL и плагин стандартов [v8-code-style](https://github.com/1C-Company/v8-code-style). Инструменты 1c-syntax (VS Code + BSL LS) — дополняющая, а не конкурирующая ветка (для разработчиков, работающих вне EDT).

---

## 1. Официальная документация по встроенному языку

Официальный первоисточник — информационная система [1С:ИТС](https://its.1c.ru), где публикуется документация по каждой версии платформы. Корень актуальной документации — [its.1c.ru/db/v83doc](https://its.1c.ru/db/v83doc).

Структура официального описания языка (подтверждено вводной страницей документации 8.3.23, [its.1c.ru/db/content/v8323doc](https://its.1c.ru/db/content/v8323doc/src/%D1%80%D1%83%D0%BA%D0%BE%D0%B2%D0%BE%D0%B4%D1%81%D1%82%D0%B2%D0%BE%20%D0%B0%D0%B4%D0%BC%D0%B8%D0%BD%D0%B8%D1%81%D1%82%D1%80%D0%B0%D1%82%D0%BE%D1%80%D0%B0/%D0%B2%D0%B2%D0%B5%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5.htm)):

- **Синтаксис встроенного языка и языка запросов** — книга «1С:Предприятие 8.3. Руководство разработчика».
- **Описание объектной модели** — включено в поставку в электронном виде: справка конфигуратора и **синтакс-помощник**; также отдельной книгой «1С:Предприятие 8.3. Описание встроенного языка».
- Непосредственно раздел «Глава 4. Встроенный язык» Руководства разработчика: [its.1c.ru/db/v838doc/bookmark/dev/TI000000116](https://its.1c.ru/db/v838doc/bookmark/dev/TI000000116).

**Синтакс-помощник (СП)** — встроенная в конфигуратор справочная система по всем объектам, методам, свойствам и константам платформы и конфигурации; открывается `Ctrl+Shift+F1`, из меню «Сервис → Синтакс-помощник» или по `F1` на ключевом слове в модуле ([обзор СП](https://sectorsystem.by/info/articles/blok-programmista-1s-novichok/sintaks-pomoshchnik-vash-glavnyy-spravochnik-kak-im-polzovatsya-effektivno/)).

**Важное ограничение для MCP-контекста:** полное содержание ИТС — под подпиской (`semi-public`). Для программного доступа к справке сообщество разбирает файлы `.hbk` синтакс-помощника (см. [bsl-help-toc-parser](https://github.com/1c-syntax/bsl-help-toc-parser) — парсер оглавления `shcntx_root.hbk`; формат — «скобкофайл» из чанков). Существуют и MCP-серверы, отдающие справку платформы из `shcntx_ru.hbk` (например `bsl-context`, [aitools1c.dev](https://aitools1c.dev/categories/platform-bsl-help/)), но это уже сторонние решения.

---

## 2. Синтаксис и описание языка на GitHub

Задача указывает репозиторий `github.com/1c-syntax/bsl` как «репозиторий описаний языка в markdown». **Проверено: такого репозитория нет** — `GET https://api.github.com/repos/1c-syntax/bsl` возвращает HTTP 404 ([проверка API](https://api.github.com/repos/1c-syntax/bsl)). В организации 1c-syntax репозиториев с именем ровно `bsl` не существует (есть `bsl-parser`, `bsl-language-server`, `bsl-context`, `bsl-common-library` и др.).

Фактическое описание синтаксиса живёт в зонтичном репозитории **[1c-syntax/1c-syntax](https://github.com/1c-syntax/1c-syntax)** (бывший `xDrivenDevelopment/1c-syntax`): это грамматика языка в формате TextMate (файл `1c.YAML-tmLanguage`), из которой генерируются пакеты подсветки для Sublime Text, Atom и VS Code. MIT, создан 2015-12-03, ~142 ⭐. Описание формата — [в README](https://github.com/1c-syntax/1c-syntax).

Программное описание грамматики/семантики языка сосредоточено в двух репозиториях:

- **bsl-parser** — грамматики ANTLR4 (лексер + парсер), фактически машиночитаемое описание языка (см. раздел 4).
- **bsl-language-server** — документация возможностей языка и диагностик (см. раздел 3), включая каталог [capabilities](https://1c-syntax.github.io/bsl-language-server/en/capabilities).

Отдельно существует **OneScript** ([oscript.io](http://oscript.io)) — независимая открытая реализация/рантайм языка, совместимая с BSL по синтаксису; используется инструментами 1c-syntax как второй целевой язык (расширение `.os`). Для него есть [tree-sitter-onescript](https://github.com/1c-syntax/tree-sitter-onescript).

**Вывод:** пункт задачи следует скорректировать — описания языка «в markdown» как отдельного репозитория нет; каноническое описание синтаксиса — TextMate-грамматика `1c-syntax/1c-syntax` + ANTLR4-грамматики `bsl-parser`.

---

## 3. BSL Language Server

[github.com/1c-syntax/bsl-language-server](https://github.com/1c-syntax/bsl-language-server) — реализация [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) для 1C (BSL) — 1С:Предприятие 8 и OneScript. Написан на **Java** (Spring), лицензия LGPL-3.0, ~490 ⭐, создан 2018-12-31, активная разработка. Документация: [1c-syntax.github.io/bsl-language-server](https://1c-syntax.github.io/bsl-language-server).

**Режимы работы (CLI)** — из [docs/en/index.md](https://github.com/1c-syntax/bsl-language-server/blob/develop/docs/en/index.md):

| Команда | Назначение |
|---|---|
| `lsp` (по умолчанию) | LSP-сервер поверх stdio |
| `websocket` (`-w`) | LSP поверх WebSocket (порт по умолчанию 8025) |
| `analyze` (`-a`) | Пакетный анализ → диагностики (для CI/CD) |
| `format` (`-f`) | Форматирование файлов |
| `mcp` | **MCP-сервер** (см. ниже) |
| `version` (`-v`) | Версия |

Конфигурация — файл `.bsl-language-server.json` в корне проекта (`-c` / `--configuration`).

**Возможности (LSP):** автодополнение с выводом типов, переход к определению (внутри модуля и между модулями конфигурации), переход к реализациям (`&Интерфейс`/`&Реализует` для OneScript-классов), поиск ссылок, hover-документация, сигнатурная подсказка, диагностики, code actions/quick fixes, иерархия типов, форматирование ([каталог capabilities](https://1c-syntax.github.io/bsl-language-server/en/capabilities)).

**MCP-режим** ([features/McpMode.md](https://github.com/1c-syntax/bsl-language-server/blob/develop/docs/en/features/McpMode.md)) — помечен как **экспериментальный** (построен на Spring AI 2.0, milestone). Работает поверх того же движка парсинга/провайдеров, что и LSP. Способы запуска:

```sh
# автономный MCP-сервер (транспорт: stdio / sse / streamable)
java -jar bsl-language-server.jar mcp                       # stdio
java -jar bsl-language-server.jar mcp --protocol sse --server.port=8080
java -jar bsl-language-server.jar mcp --protocol streamable --server.port=8080

# MCP рядом с LSP (stdio) / LSP-over-websocket
java -jar bsl-language-server.jar --mcp --server.port=8080
java -jar bsl-language-server.jar websocket --mcp --server.port=8080
```

Рабочие каталоги (workspace folders) регистрируются инструментами `register_workspace_folder` / `list_workspace_folders` / `unregister_workspace_folder`; анализ отвечает только внутри зарегистрированных каталогов (корень проекта — каталог с исходниками `src/cf` конфигурации или OneScript-проектом и, при наличии, `.bsl-language-server.json`). Есть MCP-tools `type_info`, `global_member_info`, `global_member_search` (не привязаны к файлу, требуют явного `workspaceFolder`).

> Это самый релевантный для нашего проекта компонент: готовый MCP-сервер, понимающий BSL, без необходимости самому парсить язык. Ограничение — статус «experimental» и привязка к конкретной ревизии MCP-протокола (SDK говорит на ревизии `2025-11-25`; roots-механизм помечен deprecated в ревизии `2026-07-28`).

---

## 4. Парсеры (bsl-parser)

[github.com/1c-syntax/bsl-parser](https://github.com/1c-syntax/bsl-parser) — «Коллекция парсеров языка 1С (BSL) в формате ANTLR4». GPL-3.0, ~64 ⭐, создан 2018-12-31.

Содержит три грамматики ([README](https://github.com/1c-syntax/bsl-parser)):

1. **Парсер языка 1С (BSL)** — правила лексера и грамматики на основе BNF/JFlex из репозитория [IntelliJ Idea OneScript Support](https://github.com/dmpas/idea-onescript) авторства Сергея Батанова (`@dmpas`).
2. **Парсер языка запросов 1С (SDBL)**.
3. **Парсер описаний методов** (комментарии-описания методов).

Сопутствующие библиотеки: [bsl-parser-core](https://github.com/1c-syntax/bsl-parser-core) (ядерная библиотека) и [bsl-parser-testing](https://github.com/1c-syntax/bsl-parser-testing) (тестирование парсеров). Используется форк ANTLR от 1c-syntax (свой ATN UUID — для совместимости генерируемых классов между `bsl-parser` и `bsl-language-server`, [см. комментарий](https://github.com/1c-syntax/bsl-help-toc-parser/blob/master/build.gradle.kts)). Сгенерированные классы — `BSLParser`, `BSLMethodDescriptionParser` и др. ([javadoc](https://1c-syntax.github.io/bsl-parser/dev/javadoc/com/github/_1c_syntax/bsl/parser/BSLParser.html)).

Роль в экосистеме: `bsl-parser` — нижний слой; на него опирается `bsl-language-server` (диагностики, навигация, рефакторинги) и сторонние анализаторы (например, SonarQube-плагин [sonar-bsl-plugin-community](https://github.com/1c-syntax/sonar-bsl-plugin-community)).

---

## 5. Подсветка и поддержка в редакторах

Каноническая грамматика подсветки — `1c-syntax/1c-syntax` (TextMate). Из неё развёрнуты пакеты под каждый редактор:

| Редактор | Репозиторий | Примечание |
|---|---|---|
| VS Code | [vsc-language-1c-bsl](https://github.com/1c-syntax/vsc-language-1c-bsl) | расширение **Language 1C (BSL)**, publisher `1c-syntax` |
| Sublime Text | [sublime-language-1c-bsl](https://github.com/1c-syntax/sublime-language-1c-bsl) | Package Control |
| Atom | [atom-language-1c-bsl](https://github.com/1c-syntax/atom-language-1c-bsl) | — |
| Vim | [vim-language-1c-bsl](https://github.com/andreevlex/vim-language-1c-bsl) | — |
| IntelliJ | [intellij-language-1c-bsl](https://github.com/1c-syntax/intellij-language-1c-bsl) | плагин IntelliJ Platform |
| CodeMirror 6 | [codemirror-lang-bsl](https://github.com/1c-syntax/codemirror-lang-bsl) | грамматика Lezer + встроенная подсветка SDBL |

Также: `pygments-bsl` (Pygments-лексер), `KDE-bsl-syntax` (KDE XML), подсветка для GitLab/GitHub (грамматика 1c-syntax принята в [github-linguist](https://github.com/github-linguist/linguist/pull/7096)).

**VS Code — расширение «Language 1C (BSL)»** ([vsc-language-1c-bsl](https://github.com/1c-syntax/vsc-language-1c-bsl)): MIT, ~193 ⭐, создано 2015-12-08, TypeScript. Публикуется на [Marketplace](https://marketplace.visualstudio.com/items?itemName=xDrivenDevelopment.language-1c-bsl) (publisher `1c-syntax`, id `xDrivenDevelopment.language-1c-bsl`, текущая версия 2.1.1 по [package.json](https://github.com/1c-syntax/vsc-language-1c-bsl/blob/master/package.json)). Делает две вещи:

1. подсветка синтаксиса `*.bsl` / `*.os` (TextMate-грамматика) и языка запросов;
2. интеграция с BSL Language Server через LSP — настройки `language-1c-bsl.languageServerEnabled` (по умолчанию `true`) и `language-1c-bsl.languageServerConfiguration` (путь к `.bsl-language-server.json`) из [package.json](https://github.com/1c-syntax/vsc-language-1c-bsl/blob/master/package.json).

Для других сред есть плагины, работающие через LSP того же сервера (например [claude-code-bsl-lsp](https://github.com/1c-syntax/claude-code-bsl-lsp) — плагин Claude Code).

---

## 6. Связь с EDT (1C:Enterprise Development Tools)

**1C:EDT** — официальная современная среда разработки приложений 1С на базе Eclipse ([edt.1c.ru](https://edt.1c.ru); публичный API — [edt.1c.ru/dev](https://edt.1c.ru/dev)). Кроссплатформенна (Windows/Linux/macOS), интегрирована с Git, плагинной архитектурой.

У EDT **собственная** модель встроенного языка, независимая от ANTLR4-инструментов 1c-syntax:

- Грамматика и семантическая модель описаны в [«Модель встроенного языка»](https://edt.1c.ru/dev/ru/docs/plugins/dev/lang/bsl-model/) (документация EDT public API). Модель построена на Xtext; корневой элемент `Module`, элементы `Method`/`Procedure`/`Function`, `ImplicitVariable`, `Expression`, `Statement` и т. д. в пространстве `com._1c.g5.v8.dt.bsl.model`. Доступ к node-модели — через сервис `org.eclipse.xtext.nodemodel.util.NodeModelUtils`.

- Стандарты разработки 1С оформлены как Eclipse-плагин **[1C-Company/v8-code-style](https://github.com/1C-Company/v8-code-style)** («1С:Стандарты разработки V8»): EPL-2.0, ~241 ⭐, создан 2021-02-09, входит в дистрибутив `1C:EDT 2021.3+` и не требует отдельной установки.

**Соотношение двух экосистем:**

- EDT (Eclipse + Xtext + v8-code-style) — «официальная» линия 1С: полная среда с семантической моделью, рефакторингами и проверками стандартов.
- 1c-syntax (ANTLR4 `bsl-parser` + `bsl-language-server` + TextMate-грамматики) — «сообщественная» линия, дающая поддержку языка в легковесных редакторах (VS Code и др.) и, что важно для нас, **MCP/LSP-интерфейс к языку вне Eclipse**.
- Модели не совпадают и не делят грамматику; инструменты работают параллельно. Для MCP-интеграции на рабочем месте «Windows + VS Code + агент» практичнее линия 1c-syntax (BSL LS с MCP-режимом), а EDT-модель полезна как справочник по официальной семантике языка.

---

## Сравнительная таблица инструментов языка

| Инструмент | Назначение | Как работает | Зрелость |
|---|---|---|---|
| **bsl-parser** (1c-syntax, ANTLR4) | Парсинг BSL, языка запросов (SDBL) и описаний методов | Грамматики ANTLR4 (.g4): лексер + парсер → дерево разбора; правила на основе BNF/JFlex из IntelliJ OneScript (dmpas); форк ANTLR 1c-syntax | Зрелый: с 2018, GPL-3.0, ~64 ⭐, активная поддержка |
| **bsl-language-server** (1c-syntax, Java/Spring) | LSP-сервер: диагностики, навигация, автодополнение, рефакторинги, форматирование; **+ MCP-режим** | Работает поверх bsl-parser; режимы `lsp`/`websocket`/`analyze`/`format`/`mcp`; конфиг `.bsl-language-server.json` | Зрелый: с 2018, LGPL-3.0, ~490 ⭐; MCP-режим — экспериментальный (Spring AI 2.0) |
| **1c-syntax/1c-syntax** (TextMate-грамматика) | Источник грамматики подсветки языка | Единый `1c.YAML-tmLanguage` → tmLanguage-пакеты для Sublime/Atom/VS Code | Зрелый: с 2015, MIT, ~142 ⭐ |
| **vsc-language-1c-bsl** (VS Code) | Подсветка + LSP-интеграция в VS Code | TextMate-грамматика + LSP-клиент к BSL LS; `*.bsl`/`*.os`; настройки `language-1c-bsl.*` | Зрелый: с 2015, MIT, ~193 ⭐, v2.1.1 |
| **1C:EDT (модель BSL)** (1С, Eclipse) | Официальная среда разработки; семантическая модель языка | Eclipse + Xtext-грамматика; модель `com._1c.g5.v8.dt.bsl.model`; доступ через `NodeModelUtils` | Официальный продукт 1С, зрелый, коммерческая поддержка |
| **v8-code-style** (1C-Company) | Проверка стандартов разработки 1С в EDT | Eclipse-плагин, p2-репозиторий; входит в EDT 2021.3+ | Официальный: EPL-2.0, ~241 ⭐ |
| **codemirror-lang-bsl** (1c-syntax) | Подсветка в веб-редакторах (CodeMirror 6) + встроенный SDBL | Грамматика Lezer; опирается на bsl-parser и TextMate-грамматику | Молодой: создан 2026-05, MIT, ~3 ⭐ |
| **bsl-help-toc-parser** (1c-syntax) | Разбор оглавления синтакс-помощника платформы | Парсит `shcntx_root.hbk` («скобкофайл» чанков) | Нишевый, рабочий |
| **OneScript** (oscript.io) | Независимый рантайм BSL-совместимого языка | Отдельный интерпретатор для `.os`; второй целевой язык инструментов 1c-syntax | Зрелый, самостоятельный проект |

---

## Источники

1. https://its.1c.ru/db/v83doc | 1С:Предприятие 8.3. Документация (актуальная версия) | official | 2026-10-14
2. https://its.1c.ru/db/v838doc/bookmark/dev/TI000000116 | Глава 4. Встроенный язык :: Руководство разработчика | official | 2026-10-14
3. https://its.1c.ru/db/content/v8323doc/src/руководство%20администратора/введение.htm | Введение (структура документации: Руководство разработчика, Описание встроенного языка, синтакс-помощник) | official | 2026-10-14
4. https://github.com/1c-syntax/1c-syntax | Зонтичный репозиторий грамматики подсветки (TextMate/YAML-tmLanguage) | community | 2026-10-14
5. https://github.com/1c-syntax/bsl-parser | Коллекция парсеров BSL/SDBL/описаний методов (ANTLR4) | community | 2026-10-14
6. https://github.com/1c-syntax/bsl-language-server | BSL Language Server (LSP, MCP-режим) | community | 2026-10-14
7. https://1c-syntax.github.io/bsl-language-server | Документация BSL Language Server (capabilities, McpMode) | community | 2026-10-14
8. https://github.com/1c-syntax/vsc-language-1c-bsl | VS Code-расширение Language 1C (BSL) | community | 2026-10-14
9. https://edt.1c.ru/dev | 1C:EDT public API documentation | official | 2026-10-14
10. https://edt.1c.ru/dev/ru/docs/plugins/dev/lang/bsl-model/ | Модель встроенного языка в 1C:EDT (Xtext) | official | 2026-10-14
11. https://github.com/1C-Company/v8-code-style | 1С:Стандарты разработки V8 (плагин 1C:EDT) | official | 2026-10-14

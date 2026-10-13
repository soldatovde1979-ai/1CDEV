# MCP-серверы в VS Code: как подключаются к агентам/расширениям

> Исследовательский срез (не проектирование). Дата сбора: 2026-10-14.
> Цель: понять, какие агенты/расширения VS Code умеют работать как MCP-клиенты, где у каждого хранится конфигурация MCP-серверов и какие транспорты поддерживаются.

## Ключевые выводы

- **Все крупные агенты для VS Code — это MCP-клиенты** (подключают внешние MCP-серверы как инструменты), а не MCP-серверы. Исключение/особенность: **Roo Code** умеет ещё и *создавать* MCP-серверы по запросу.
- **Три транспорта** являются де-факто стандартом у всех: **stdio** (локальный процесс), **Streamable HTTP** (рекомендуемый для новых remote-серверов), **SSE** (legacy).
- **Главная ловушка переносимости конфигураций** — топ-ключ JSON:
  - VS Code / Cody / GitHub Copilot (в VS Code) → `"servers"`
  - Cline / Roo Code / Cursor / Claude Desktop / Copilot CLI / Copilot cloud agent → `"mcpServers"`
  - Continue → вообще не объект, а YAML-массив с полем `name`.
- **Тип поля transport** тоже не унифицирован: Cline пишет `"streamableHttp"` (camelCase), Roo Code и GitHub — `"streamable-http"` (с дефисом), VS Code/Cody — `"http"`.
- **MCP в VS Code доступен только в режиме Agent** (Copilot Chat → Agent), у Cody — только Agent mode, у Continue — только agent mode.
- VS Code получил нативную поддержку MCP в **1.99 (апрель 2025)**; пользовательский OAuth clientId — с **1.123 (июнь 2026)**.
- Файлы конфигурации: VS Code — `.vscode/mcp.json` (workspace) / user profile; Cline — `cline_mcp_settings.json`; Roo Code — глобальный `mcp_settings.json` + проектный `.roo/mcp.json`; Continue — `~/.continue/config.yaml` + `.continue/mcpServers/`; Cursor — `.cursor/mcp.json` + `~/.cursor/mcp.json`.

---

## Разбор по расширениям/агентам

### 1. Cline (docs.cline.bot)

- **Роль:** MCP-клиент. Подключает MCP-серверы как инструменты агента.
- **Транспорты:** stdio (локальный), Streamable HTTP (рекомендуется), SSE (legacy).
- **Где конфигурация:**
  - IDE-расширение: файл `cline_mcp_settings.json` (открывается из UI: панель Cline → иконка «MCP Servers» → вкладка **Configure** → кнопка **Configure MCP Servers**).
  - CLI: `~/.cline/mcp.json` (в новых версиях `~/.cline/data/settings/cline_mcp_settings.json`); wizard `cline mcp` (add/edit/enable/disable/delete), листинг `cline config mcp --json`.
- **Как подключать:**
  - Локально (stdio): объект `command` + `args` + `env` + `autoApprove`.
  - Remote: вкладка **Remote Servers** → имя, URL, тип (Streamable HTTP / SSE) → Add Server; либо в JSON запись с `"type": "streamableHttp"` (camelCase!) и `"url"` + `"headers"`.
  - Marketplace: one-click установка каталога MCP-серверов.
- **Особенность:** если тип remote не указан явно как `streamableHttp`, Cline откатывается на SSE — и endpoint Streamable-HTTP даст `405`.

### 2. Roo Code (docs.roocode.com)

- **Роль:** MCP-клиент **и** умеет генерировать/создавать MCP-серверы («Enable MCP Server Creation»).
- **Транспорты:** stdio, Streamable HTTP (`"type": "streamable-http"`, с дефисом), SSE (legacy).
- **Где конфигурация (два уровня):**
  1. **Global:** `mcp_settings.json` (через VS Code settings) — действует во всех воркспейсах.
  2. **Project:** `.roo/mcp.json` в корне проекта (коммитится в VCS, приоритетнее глобального при совпадении имени).
- **Как подключать:** UI → иконка MCP → **Edit Global MCP** / **Edit Project MCP**. JSON с объектом `mcpServers`.
- **Параметры (stdio):** `command`, `args`, `cwd`, `env`, `alwaysAllow`, `disabled`, `timeout` (1–3600 с), `watchPaths`, `disabledTools`; интерполяция `${env:VAR}`.
- **Windows:** для npx-серверов рекомендуют `"command": "cmd", "args": ["/c", "npx", "-y", ...]`.

### 3. Continue (docs.continue.dev)

- **Роль:** MCP-клиент. MCP работает **только в agent mode**.
- **Транспорты:** `sse`, `stdio`, `streamable-http` (поле `type`).
- **Где конфигурация (YAML, не JSON!):**
  - Глобальный `~/.continue/config.yaml` (Windows: `%USERPROFILE%\.continue\config.yaml`) — секция `mcpServers` как **массив** (с `-`), у сервера поле `name`.
  - Воркспейс: каталог `.continue/mcpServers/` с файлами `*.yaml` (блоки с метаданными `name`, `version`, `schema: v1`) или `mcp.json` (совместимость с JSON-конфигами из Claude/Cursor/Cline).
- **Как подключать:** добавить сервер в YAML → перезагрузить («Continue: Reload»).
- **Особенность:** НЕ поддерживает project-level конфиг в общем смысле; `mcpServers` — массив, а не объект (частая причина ошибок при копировании из Cursor/Cline).

### 4. GitHub Copilot в VS Code (code.visualstudio.com)

- **Роль:** MCP-клиент. Требует VS Code **1.99+** и **Agent mode** (не Ask/Edit).
- **Транспорты:** stdio, `http` (VS Code сначала пробует Streamable HTTP, затем откатывается на SSE), `sse`.
- **Где конфигурация (два формата):**
  - **VS Code format** — `.vscode/mcp.json` (workspace) или user profile (`MCP: Open User Configuration`); топ-ключ **`servers`**.
  - **Portable format** — `.mcp.json` в корне воркспейса или `~/.copilot/mcp-config.json`; топ-ключ `mcpServers` (читается напрямую Agent Host).
- **Как подключать:** Command Palette → **MCP: Add Server** (мастер: тип stdio/HTTP-SSE, URL, имя, scope Workspace/Global); либо `code --add-mcp '...'` (CLI); либо `vscode:mcp/install?<json>` install-link.
- **Поля:** `type`, `url`, `headers`, `oauth` (clientId), `inputs`/`${input:...}` для секретов, `sandboxEnabled`; HTTP может идти и на unix-socket / Windows named pipe (`pipe:///pipe/...`).
- **Copilot CLI / cloud coding agent (docs.github.com):** repo-level конфиг на GitHub.com (Settings → Copilot → MCP servers), топ-ключ `mcpServers`, `type`: `local`/`stdio`/`http`/`sse`, поле `tools` (allowlist); CLI хранит в `~/.copilot/mcp-config.json`, команды `copilot mcp add/list`.

### 5. Cody (Sourcegraph) (sourcegraph.com/docs/cody/capabilities/mcp)

- **Роль:** MCP-клиент. Реализован поверх OpenCtx. MCP-инструменты доступны **только в Agent mode** (не в обычном чате/автодополнении).
- **Транспорты:** stdio, http, sse.
- **Где конфигурация:** `.vscode/mcp.json` (workspace) или user scope (Command Palette → **MCP: Open User Configuration**); топ-ключ **`servers`** (не `mcpServers`).
- **Как подключать:** установить расширение Cody → создать/дополнить `.vscode/mcp.json` → переключить Cody в Agent mode → инструменты появляются в tool picker.
- **Особенность:** плагин для JetBrains использует другой путь конфигурации, чем VS Code-расширение.

### 6. Cursor (другие MCP-клиенты; вторичные источники)

- **Роль:** MCP-клиент (во всех режимах, включая Composer/agent).
- **Транспорты:** stdio, SSE, Streamable HTTP (с OAuth).
- **Где конфигурация:** `.cursor/mcp.json` (project) и `~/.cursor/mcp.json` (global); при совпадении имён project-конфиг приоритетнее; топ-ключ `mcpServers`.
- **Как подключать:** UI, Marketplace (one-click + OAuth), deeplink `cursor://anysphere.cursor-deeplink/mcp/install?...`, ручной `mcp.json`.
- **Интерполяция:** `${env:NAME}`, `${userHome}`, `${workspaceFolder}`, `${workspaceFolderBasename}`, `${pathSeparator}`.

---

## Сравнительная таблица

| Расширение/агент | Поддержка MCP | Транспорты | Где конфигурация | Как подключать |
|---|---|---|---|---|
| **Cline** | клиент | stdio, Streamable HTTP, SSE (legacy) | `cline_mcp_settings.json`; CLI: `~/.cline/mcp.json`; Marketplace | UI «MCP Servers» → Configure / Remote Servers / Marketplace; CLI `cline mcp`; JSON `mcpServers`, remote `"type": "streamableHttp"` (camelCase) |
| **Roo Code** | клиент + создание серверов | stdio, Streamable HTTP (`"streamable-http"`), SSE | Global `mcp_settings.json` + Project `.roo/mcp.json` (приоритет project) | UI → Edit Global/Project MCP; JSON `mcpServers`; `${env:VAR}`; Windows через `cmd /c npx` |
| **Continue** | клиент (agent mode) | stdio, sse, streamable-http | `~/.continue/config.yaml` (YAML-массив) + `.continue/mcpServers/*.yaml|mcp.json` | секция `mcpServers` в YAML (массив с `name`) → «Continue: Reload» |
| **GitHub Copilot (VS Code)** | клиент (agent mode, VS Code 1.99+) | stdio, http (fallback на SSE), sse | `.vscode/mcp.json` (workspace) / user profile; portable `.mcp.json`, `~/.copilot/mcp-config.json` | «MCP: Add Server» мастер, `code --add-mcp`, `vscode:mcp/install`; VS Code-ключ `servers` |
| **GitHub Copilot (CLI / cloud agent)** | клиент | local, stdio, http, sse | repo-level на GitHub.com (Settings → Copilot → MCP); CLI `~/.copilot/mcp-config.json` | JSON `mcpServers`, поле `tools`; `copilot mcp add` |
| **Cody (Sourcegraph)** | клиент (agent mode, через OpenCtx) | stdio, http, sse | `.vscode/mcp.json` (workspace) / user scope; ключ `servers` | создать `.vscode/mcp.json` → Agent mode → tool picker |
| **Cursor** | клиент | stdio, SSE, Streamable HTTP (OAuth) | `.cursor/mcp.json` (project) + `~/.cursor/mcp.json` (global); ключ `mcpServers` | UI, Marketplace, deeplink, ручной `mcp.json`; `${env:...}` |

---

## Источники

1. https://docs.cline.bot/mcp/mcp-overview | Cline — MCP (обзор и настройка серверов) | official | 2026-10-14 (дата обращения)
2. https://docs.roocode.com/features/mcp/using-mcp-in-roo | Roo Code — Using MCP in Roo (глобальная/проектная конфигурация, транспорты) | official | 2026-10-14 (дата обращения)
3. https://docs.continue.dev/customize/deep-dives/mcp | Continue — How to Set Up Model Context Protocol (MCP) | official | 2026-10-14 (дата обращения)
4. https://code.visualstudio.com/docs/agents/reference/mcp-configuration | VS Code — MCP configuration reference (форматы `servers`/`mcpServers`, транспорты) | official | 2026-10-14 (дата обращения)
5. https://code.visualstudio.com/docs/copilot/customization/mcp-servers | VS Code — GitHub Copilot MCP servers (агентный режим) | official | 2026-10-14 (дата обращения)
6. https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/configure-mcp-servers | GitHub Docs — Configure MCP servers for your repository (cloud agent / code review) | official | 2026-10-14 (дата обращения)
7. https://sourcegraph.com/docs/cody/capabilities/mcp | Sourcegraph — Cody MCP | official | 2026-10-14 (дата обращения)
8. https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-cline.md | GitHub MCP Server — installation guide: Cline (пример streamableHttp) | official | 2026-10-14 (дата обращения)
9. https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-roo-code.md | GitHub MCP Server — installation guide: Roo Code (пример streamable-http) | official | 2026-10-14 (дата обращения)
10. https://preprints.org/manuscript/202506.0821/v1 | A Comparative Featureset Analysis of Agentic IDE Tools (MCP-поддержка Copilot/Cursor/Cody/Cline и др.) | academic | 2025-06
11. https://mcpnav.dev/learn/add-mcp-server-to-cursor | mcpnav — Add an MCP server to Cursor (файлы, транспорты, интерполяция) | community | 2026-10-14 (дата обращения)
12. https://stackmcp.dev/blog/setup-mcp-servers-continue | StackMCP — How to Set Up MCP Servers in Continue.dev (YAML vs JSON) | secondary-industry | 2026-10-14 (дата обращения)

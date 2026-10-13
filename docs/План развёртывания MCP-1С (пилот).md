# План: AI-ассистированная разработка 1С — развёртывание MCP (пилот)

**Описание:** Настройка связки «автоматическая выгрузка конфигурации в git + MCP-сервер
метаданных + BSL Language Server (язык и правила) + агент в VS Code». Без Docker, без
платных компонентов (кроме модели DeepSeek). Основано на манифесте
`Архитектурный Манифест v1.0 MCP-интеграция 1С.md` и базе знаний `research/` (92 источника).

**Инструменты:** 1С:Конфигуратор, VS Code, Cline или Roo Code, Python 3.11+,
BSL Language Server (Java 17+), MCP Python SDK, git. (Phase 2: mcp-1c, Apache/IIS.)

**Время:** 2–3 часа на ядро (Этапы 1–5); +1–2 часа на Phase 2 (Этапы 6–7).

**Исполнители:** [Р] — ты (GUI: Конфигуратор, VS Code, браузер). [А] — агент (команды,
правка файлов, git).

**Лицензии:** всё бесплатно/open-source, кроме DeepSeek API (по токенам) и платформы 1С
(лицензия уже есть). ai_rules_1c — без лицензии (см. Этап 7).

---

## Этап 0. Проверка окружения [Р]+[А]

1. [Р] Узнай версию платформы: Конфигуратор → «?» → «О программе». Нужна 8.3.16+
   (рекомендовано 8.3.24+). Запиши номер — пригодится на Этапе 1 и 6.
2. [А] Проверь инструменты (агент выполняет):

```bash
python --version        # нужно 3.11+
java -version           # нужно 17+ (Temurin 21 — оптимально)
git --version
code --version          # VS Code 1.99+
```

Если Java нет или младше 17 — установи Eclipse Temurin 21 (zip-дистрибутив с adoptium.net,
распаковать в `C:\Java\jdk-21`, не менять системный PATH).

---

## Этап 1. Автоматизация выгрузки конфигурации [Р]+[А]

Создаём источник истины — выгрузку конфигурации в XML, коммитимую в git.

1. [А] Создай структуру каталогов и git-репозиторий:

```bash
mkdir -p C:/Projects/1C/src/cf C:/Projects/1C/work-ib C:/Projects/1C/logs
cd C:/Projects/1C && git init -b main && echo "# 1C config export" > README.md && git add -A && git commit -m "init"
```

2. [А] Создай сценарий `C:\Projects\1C\tools\export-config.cmd` (путь к 1cv8 подставь по
   версии из Этапа 0):

```bat
@echo off
set BIN=C:\Program Files\1cv8\8.3.24.xxxx\bin\1cv8.exe
"%BIN%" DESIGNER /IBConnectionString "File=C:\Projects\1C\work-ib" /DisableStartupDialogs ^
  /DumpConfigToFiles "C:\Projects\1C\src\cf" -Format иерархический -update ^
  /Out "C:\Projects\1C\logs\dump.log"
if %errorlevel% neq 0 exit /b %errorlevel%
cd /d "C:\Projects\1C\src"
git add -A && git commit -m "config: automated export" || exit /b 0
```

3. [Р] Первичная выгрузка: Конфигуратор → `Конфигурация → Выгрузить конфигурацию в файлы...`
   в `C:\Projects\1C\src\cf`. (Первый раз — вручную; дальше — автоматически.) Успех —
   папка заполнилась XML-файлами (`Documents`, `Catalogs`, `CommonModules`…).

> **Инфо:** XML-выгрузка — единственный формат, пригодный для построчного diff и git.
> Файл `.cf` — бинарный контейнер, в git его коммитить бессмысленно.

4. [А] Закоммить первичную выгрузку, затем создай задачу в Windows Task Scheduler
   (ежедневно, ночь) на `export-config.cmd`.

---

## Этап 2. MCP-сервер «1c-config» (метаданные + код) [А]

1. [А] Установи SDK и создай сервер `C:\Tools\1c-config\server.py`:

```bash
pip install mcp
mkdir C:/Tools/1c-config
```

Скелет (FastMCP, read-only, пути только внутри `CF_DIR`):

```python
from mcp.server.fastmcp import FastMCP
import os, subprocess

CF_DIR = os.environ.get("ONEC_CF_DIR", r"C:\Projects\1C\src\cf")
mcp = FastMCP("1c-config")

@mcp.tool()
def export_status() -> str:
    out = subprocess.run(["git", "-C", os.path.dirname(CF_DIR), "log", "-1",
                          "--format=%h %ci %s"], capture_output=True, text=True).stdout.strip()
    return f"export dir={CF_DIR}; last commit={out}"

@mcp.tool()
def list_metadata_objects(type: str | None = None) -> list[str]:
    ...  # обход CF_DIR/Configuration, чтение *.xml

@mcp.tool()
def get_object_metadata(full_name: str) -> str: ...
@mcp.tool()
def get_module_text(full_name: str, module: str) -> str: ...
@mcp.tool()
def search_objects(query: str) -> list[str]: ...
@mcp.tool()
def search_in_modules(pattern: str, max_hits: int = 50) -> list[str]: ...
@mcp.tool()
def list_subsystems() -> list[str]: ...

if __name__ == "__main__":
    mcp.run()  # stdio
```

2. [А] Проверь: `export_status` должен вернуть последний git-коммит выгрузки.
   (Полный список инструментов и сигнатур — в манифесте §5.2.)

---

## Этап 3. BSL Language Server — язык и правила [А]

1. [А] Скачай `bsl-language-server.jar` (релизы `1c-syntax/bsl-language-server`) в
   `C:\Tools\bslls\`, проверь запуск:

```bash
java -jar C:/Tools/bslls/bsl-language-server.jar mcp --protocol stdio
# сервер должен ответить на initialize (JSON-RPC). Ctrl+C — выход.
```

> **Инфо:** это нативный MCP-режим BSL LS — тот же движок, что у LSP (парсинг + 100+
> диагностик стандарта, тег `STANDARD`). Отдельный мост `mcp-bsl-lsp-bridge` не нужен.

2. [Р] Для подсветки в редакторе установи расширение VS Code **Language 1C (BSL)**
   (1c-syntax) — оно поднимет BSL LS как LSP и подсветит `.bsl`-файлы.

---

## Этап 4. Агент и подключение серверов [Р]+[А]

1. [Р] Установи **Cline** (или **Roo Code** — для команды: его `.roo/mcp.json`
   коммитится в git и подхватывается каждым). Открой `C:\Projects\1C` как папку проекта.
2. [А] Создай `.roo/mcp.json` (для Roo) или `cline_mcp_settings.json` (для Cline):

```json
{
  "mcpServers": {
    "1c-config": {
      "command": "python",
      "args": ["C:/Tools/1c-config/server.py"],
      "env": { "ONEC_CF_DIR": "C:/Projects/1C/src/cf" }
    },
    "bsl-language-server": {
      "command": "java",
      "args": ["-jar", "C:/Tools/bslls/bsl-language-server.jar", "mcp", "--protocol", "stdio"]
    }
  }
}
```

> **Ловушка:** ключ верхнего уровня — `mcpServers` (Cline/Roo/Cursor). У VS Code
> Copilot/Cody — `servers`. Для наших локальных stdio-серверов транспорт не указывается.

---

## Этап 5. Сквозная проверка [Р]

1. [Р] Перезапусти VS Code (`Reload Window`), открой панель агента, проверь, что
   `1c-config` и `bsl-language-server` появились в списке MCP-серверов.
2. [Р] В чате агента спроси:
   - «Сколько справочников в конфигурации?» → должен вызвать `list_metadata_objects`,
     а не отвечать по памяти.
   - «Как устроен документ РеализацияТоваровУслуг?» → `get_object_metadata`.
   - «Какие ошибки видит BSL LS в этом модуле?» → диагностика, а не общие слова.
3. [Р] Убедись, что `export_status` показывает актуальную дату последней выгрузки.

---

## Этап 6 (Phase 2). Live-доступ к базе: mcp-1c [Р]+[А]

1. [Р] Скачай `mcp-1c_windows_amd64.zip` с `github.com/feenlace/mcp-1c/releases/latest`
   (один `.exe`, 25–40 МБ, без зависимостей). Сохрани в `C:\Tools\mcp-1c\`.
2. [А] Установи расширение в базу (база должна быть закрыта в Конфигураторе/клиенте):

```bash
C:/Tools/mcp-1c/mcp-1c.exe --install "C:\путь\к\базе" [--platform-version 8.3.XX]
```

3. [Р] Опубликуй HTTP-сервис: Конфигуратор → `Администрирование → Публикация на
   веб-сервере` (Apache/IIS), имя публикации `mcp1c`, отметка расширения `MCP_HTTPService`.
4. [Р] **Обязательно закрой сервис аутентификацией** (Basic + TLS) и ограничь доступ
   localhost/внутренним сегментом — иначе структура базы доступна всем, кто достучится
   до порта. Учётные данные в конфиг — через env, не в открытом виде.
5. [А] Добавь сервер в `.roo/mcp.json` (блок `"1c"` — см. манифест §5.5) и проверь:

```bash
curl http://localhost/mcp1c/hs/mcp-1c/version
```

---

## Этап 7 (Phase 2). Правила и токен-экономия [Р]+[А]

1. [Р] Скачай полнотекстовые файлы правил из `github.com/comol/ai_rules_1c`
   (`content/rules`). Внимание: часть файлов — только заголовки (текст у них в Docker
   MCP-сервере), брать лишь полнотекстовые (`module-structure.md` и др.).
   [А] Разложи их в `C:\Projects\1C\.roo\rules-code\` — Roo Code подхватит автоматически.
   > **Лицензия:** у `ai_rules_1c` её нет — перед использованием в дирекции уточни право.
2. (Опционально, крупные конфигурации) [А] Установи `rlm-tools-bsl` (Dach-Coin, MIT,
   Python 3.10+) для токен-экономного анализа кода (SQLite-индекс методов, `git_search`).
   Подключи вторым сервером `rlm-bsl` (Streamable HTTP или stdio).

---

## Что это даёт и чего не делает

**Даёт:** агент видит состав метаданных и тексты модулей из выгрузки (без касания живой
базы), контролирует код через BSL LS (язык + правила), в Phase 2 — читает живую базу
через mcp-1c и опирается на перенесённые правила.

**Не делает (в этом пилоте):** запись в базу (загрузка конфигурации/выполнение кода —
отдельный защищённый контур, манифест §8); централизованный HTTP-контур с OAuth2 для
команды (Phase 3); полный набор правил ai_rules_1c без их Docker-сервера.

---

## Лицензии и стоимость (пилот)

| Компонент | Лицензия | Плата |
|---|---|---|
| Python SDK (mcp/FastMCP) | MIT | бесплатно |
| BSL Language Server | LGPL-3.0 | бесплатно |
| mcp-1c (feenlace) | MIT | бесплатно |
| rlm-tools-bsl (Dach-Coin) | MIT | бесплатно |
| VS Code, git, Java (Temurin), Python, Apache/IIS | свободные | бесплатно |
| Cline / Roo Code | Roo open-source / Cline freemium | бесплатно со своим API-ключом |
| ai_rules_1c (comol) | **нет лицензии** | уточнить |
| DeepSeek API | проприетарный | платно (по токенам) |

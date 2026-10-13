# План: развёртывание MCP-интеграции 1С для AI-агента в VS Code (редакция Claude, пилот)

**Описание:** Пошаговое развёртывание связки «автоматическая выгрузка конфигурации в git (ветки base/main) + свой read-only MCP-сервер `1c-config` (метаданные, код, поиск, диагностики BSL LS) + нативный MCP BSL LS как дополнение + Zoo/Roo Code с правилами» в изолированном внутреннем контуре Windows, без Docker и без прав администратора.

**Опирается на:** `Архитектурный Манифест Claude_v1.md` (мажор 1). Номера разделов `§N` ниже — разделы этого манифеста.

**Инструменты:** 1С:Конфигуратор (пакетный режим `1cv8 DESIGNER`), PowerShell 5.1, git, Python 3.11+, Eclipse Temurin JDK 21, BSL Language Server, MCP Python SDK, VS Code, Zoo Code / Roo Code, расширение Language 1C (BSL), Яндекс Почта (перенос файлов), Claude Code (разработка сервера в репозитории 1CDEV).

**Время:** Ф0–Ф1 — 2–3 ч (плюс 25–45 мин полной выгрузки ERP); Ф2 — 1–2 рабочих дня работы агента; Ф3 — 1 ч логистики; Ф4–Ф6 — 2–3 ч.

**Исполнители:** `[Р]` — человек (GUI, браузер, почта, ввод пароля, любые действия во внешнем контуре); `[А]` — агент (Zoo Code во внутреннем контуре или Claude Code в репозитории 1CDEV — указано в шаге).

**Машины:** «ПК-внутр» — рабочий ПК внутреннего контура; «ПК-внеш» — машина с интернетом (Windows x64). Перед шагом с командами проверь машину: `$env:COMPUTERNAME`.

**Шаги:** Версия платформы → Инвентаризация → Сеть и диск → Переменные окружения → Репозиторий и worktree → Скрипт выгрузки → Первая выгрузка → Проверка -update → Вливание base → Расписание → Разработка 1c-config → Сборка набора → Перенос → Распаковка → Установка сервера → BSL LS → Конфиги → mcp.json → Расширение VS Code → Правила агента → Приёмка → Обратный контур

---

## Фаза 0. Проверки Go/No-Go

### 0.1 Версия платформы и путь к 1cv8.exe [Р]

Версия нужна для пути к `1cv8.exe` и для проверки поддержки `-update` и частичной загрузки. Машина: ПК-внутр.

Конфигуратор → «Справка» → «О программе». Запиши номер версии платформы (вида `8.3.24.1691`) и путь вида `C:\Program Files\1cv8\8.3.24.1691\bin\1cv8.exe`. Ниже 8.3.16 — стоп: план требует пересмотра.

> **Инфо:** версия платформы (движок) и версия конфигурации (ERP, УТ) — разные числа. Нужна первая.

### 0.2 Инвентаризация инструментов, диска и путей [А]

Агент (Zoo Code) собирает факты, от которых зависят следующие фазы: версии python, git и java, свободное место, политику выполнения скриптов, поддержку длинных путей. Команда ничего не меняет — только печатает.

```powershell
$env:COMPUTERNAME
python --version; py -0p 2>$null
git --version
& "C:\Java\jdk-21\bin\java.exe" -version 2>&1 | Select-Object -First 1
java -version 2>&1 | Select-Object -First 1
code --version | Select-Object -First 1
Get-PSDrive C | Select-Object @{n='FreeGB';e={[math]::Round($_.Free/1GB,1)}}
Get-ExecutionPolicy -List
(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' -ErrorAction SilentlyContinue).LongPathsEnabled
Test-Path C:\ ; try { New-Item -ItemType Directory C:\1C -Force | Out-Null; 'C:\1C — создаётся' } catch { 'C:\1C — запрещено политикой' }
```

Что смотреть: Python 3.11+ (номер минора запиши — по нему собирается набор колёс в шаге 3.1); git есть; FreeGB ≥ 6; `C:\1C` создаётся. Если `C:\1C` запрещён — корень `C:\Projects\1C`; далее везде подставляй его вместо `C:\1C` (вопрос В-4). Если `ExecutionPolicy` для `MachinePolicy`/`UserPolicy` равна `AllSigned` или `Restricted` — стоп, скрипты `.ps1` запускать нельзя; нужно решение с администратором.

> **Инфо:** `py -0p` — список всех установленных Python с путями (лаунчер Windows). `2>&1` у `java -version` нужен, потому что java печатает версию в поток ошибок. `LongPathsEnabled = 1` — Windows разрешает пути длиннее 260 символов; если пусто или 0, git справится сам (`core.longpaths`), а Python и Проводник — нет, поэтому корень и держим коротким. `Get-ExecutionPolicy -List` показывает политику на всех уровнях: уровни `MachinePolicy`/`UserPolicy` задаются групповой политикой и перекрывают `-ExecutionPolicy Bypass`.

### 0.3 Доступность внешних ресурсов из контура [А]

Решает, что качать напрямую, а что везти через внешний контур. Команда делает HEAD-запросы и печатает «ok» или ошибку для каждого адреса.

```powershell
'https://github.com','https://objects.githubusercontent.com','https://api.github.com',
'https://pypi.org/simple/mcp/','https://files.pythonhosted.org','https://api.adoptium.net',
'https://marketplace.visualstudio.com' | ForEach-Object {
  try { Invoke-WebRequest $_ -Method Head -UseBasicParsing -TimeoutSec 10 | Out-Null; "ok    $_" }
  catch { "FAIL  $_  ($($_.Exception.Message))" }
}
```

Если всё `ok` — Ф3 можно пройти прямо на ПК-внутр без почты. Если хоть один `FAIL` — Ф3 по полной схеме через ПК-внеш (ожидаемый вариант).

> **Инфо:** `-UseBasicParsing` отключает разбор HTML через Internet Explorer — в PowerShell 5.1 без него запрос может упасть на машинах без настроенного IE. `objects.githubusercontent.com` — домен, с которого реально скачиваются файлы релизов GitHub; без него страница релиза открывается, а скачивание — нет.

---

## Фаза 1. Репозиторий и автоматическая выгрузка

### 1.1 Переменные окружения для выгрузки [Р]

Скрипт выгрузки берёт путь к платформе, ИБ и учётку из переменных пользователя — их нет ни в git, ни в `mcp.json`. Пароль вводишь ты сам, агенту его не сообщай. Машина: ПК-внутр, PowerShell.

```powershell
setx ONEC_BIN "C:\Program Files\1cv8\8.3.24.1691\bin\1cv8.exe"
setx ONEC_IB_PATH "C:\Bases\ERP_DEV"          # файловая ИБ разработки
# или для серверной ИБ вместо строки выше:  setx ONEC_IB_SERVER "srv1c\erp_dev"
setx ONEC_USER "Выгрузка"                     # пропусти обе строки, если вход через ОС (/WA+)
setx ONEC_PASSWORD "<пароль>"
```

Каждая строка печатает «УСПЕШНО: указанное значение сохранено». Новые значения видны только в **новых** окнах PowerShell и после перезапуска VS Code.

> **Инфо:** `setx` записывает переменную в профиль пользователя (реестр `HKCU\Environment`) навсегда, без прав администратора; текущее окно её не видит. Пароль хранится там открытым текстом — для общего ПК лучше вход через ОС (вопрос В-2). Пользователю 1С для выгрузки достаточно права «Администрирование».

### 1.2 Репозиторий, настройки git и worktree base [А]

Создаём репозиторий с двумя рабочими копиями: `erp` (ветка `main`, для VS Code и агента) и `erp-base` (ветка `base`, туда пишет только выгрузка) — §4.5. Машина: ПК-внутр.

```powershell
$ErrorActionPreference = 'Stop'
$Root = 'C:\1C'
New-Item -ItemType Directory -Force "$Root\erp\scripts", "$Root\logs", "$Root\tools" | Out-Null
git init -b main "$Root\erp"
$cfg = @{ 'core.longpaths'='true'; 'core.quotepath'='false'; 'core.autocrlf'='false';
          'core.safecrlf'='false'; 'feature.manyFiles'='true' }
$cfg.GetEnumerator() | ForEach-Object { git -C "$Root\erp" config $_.Key $_.Value }
Set-Content "$Root\erp\.gitattributes" "* -text`n*.bin binary" -Encoding Ascii
Set-Content "$Root\erp\.gitignore" "src/cf/ConfigDumpInfo.xml" -Encoding Ascii
git -C "$Root\erp" add -A
git -C "$Root\erp" commit -q -m "init: настройки репозитория выгрузки"
git -C "$Root\erp" branch base
git -C "$Root\erp" worktree add "$Root\erp-base" base
git -C "$Root\erp" worktree list
```

Успех — последняя команда печатает две строки: `C:/1C/erp … [main]` и `C:/1C/erp-base … [base]`. Если `git init` пишет «Reinitialized existing» — репозиторий уже был; проверь `git -C C:\1C\erp log --oneline`, прежде чем продолжать.

> **Инфо:** `worktree add` создаёт вторую рабочую папку того же репозитория на другой ветке — общая история, разные файлы на диске. `erp-base` лежит вне `erp`, поэтому VS Code и агент её не видят. `* -text` запрещает git менять переводы строк в выгрузке; `feature.manyFiles` включает формат индекса и кэш, рассчитанные на сотню тысяч файлов. `ConfigDumpInfo.xml` — служебный файл платформы для инкрементальной выгрузки; в git он даёт только шум.

### 1.3 Скрипт выгрузки export-config.ps1 [А]

Скрипт реализует алгоритм §4.3: lock, авто-восстановление после прерванного запуска, `update` с откатом на `full`, коммит только при изменениях, статус `last-dump.json` при любом исходе. Агент сохраняет файл `C:\1C\erp\scripts\export-config.ps1` в кодировке **UTF-8 с BOM** (без BOM PowerShell 5.1 испортит кириллицу).

```powershell
# export-config.ps1 — выгрузка конфигурации 1С в ветку base
# VERSION = "1.0"; контракт last-dump.json schema=1 (Манифест Claude_v1 §5.3)
[CmdletBinding()]
param(
    [switch]$Full,                  # принудительная полная выгрузка
    [int]$IfOlderThanHours = 0      # >0: выгружать, только если последняя успешная старше N часов
)
$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

$Root       = if ($env:ONEC_ROOT) { $env:ONEC_ROOT } else { 'C:\1C' }
$BaseTree   = Join-Path $Root 'erp-base'
$CfDir      = Join-Path $BaseTree 'src\cf'
$LogDir     = Join-Path $Root 'logs'
$StatusFile = Join-Path $LogDir 'last-dump.json'
$LockFile   = Join-Path $LogDir 'export.lock'

function Invoke-Git {
    # stderr git не превращаем в исключение (PS 5.1 + Stop), проверяем код возврата
    $prev = $ErrorActionPreference; $ErrorActionPreference = 'Continue'
    try { $out = & git -C $BaseTree @args 2>&1 | ForEach-Object { "$_" } }
    finally { $ErrorActionPreference = $prev }
    if ($LASTEXITCODE -ne 0) { throw "git $($args -join ' ') -> код $LASTEXITCODE`n$($out -join "`n")" }
    return $out
}

function Write-Status([hashtable]$s) {
    $s['schema'] = 1
    ($s | ConvertTo-Json -Depth 3) | Set-Content -Path $StatusFile -Encoding UTF8
}

function Clear-CfDir {
    # через git: корректно с длинными путями; удалённые в ИБ объекты исчезнут
    Invoke-Git rm -r -q --ignore-unmatch -- src/cf | Out-Null
    Invoke-Git clean -fdxq -- src/cf | Out-Null
    New-Item -ItemType Directory -Force -Path $CfDir | Out-Null
}

function Invoke-Dump([string]$Mode, [string]$Log) {
    $a = @('DESIGNER', $script:IbArg) + $script:AuthArgs + @(
        '/DisableStartupDialogs', '/DisableStartupMessages',
        "/Out `"$Log`"", "/DumpConfigToFiles `"$CfDir`"", '-Format Hierarchical')
    if ($Mode -eq 'update') { $a += '-update' }
    $p = Start-Process -FilePath $env:ONEC_BIN -ArgumentList ($a -join ' ') -Wait -PassThru -WindowStyle Hidden
    return $p.ExitCode
}

New-Item -ItemType Directory -Force -Path $LogDir | Out-Null
$prevStatus = $null
if (Test-Path $StatusFile) {
    try { $prevStatus = Get-Content $StatusFile -Raw -Encoding UTF8 | ConvertFrom-Json } catch { $prevStatus = $null }
}
$prevOk = if ($prevStatus -and $prevStatus.PSObject.Properties['last_ok']) { $prevStatus.last_ok } else { $null }

if ($IfOlderThanHours -gt 0 -and $prevOk -and ((Get-Date) - [datetime]$prevOk).TotalHours -lt $IfOlderThanHours) {
    Write-Host "Выгрузка свежая (last_ok = $prevOk) — пропуск."
    exit 0
}

try { $lock = [System.IO.File]::Open($LockFile, 'OpenOrCreate', 'ReadWrite', 'None') }
catch { Write-Host 'Выгрузка уже выполняется — выход.'; exit 0 }

$started = Get-Date
$stamp   = $started.ToString('yyyyMMdd-HHmmss')
$mode    = 'n/a'
$log     = Join-Path $LogDir "export-$stamp.log"
try {
    if (-not $env:ONEC_BIN -or -not (Test-Path $env:ONEC_BIN)) { throw "Не найден 1cv8.exe (ONEC_BIN = '$env:ONEC_BIN')" }
    if     ($env:ONEC_IB_SERVER) { $script:IbArg = "/S `"$env:ONEC_IB_SERVER`"" }
    elseif ($env:ONEC_IB_PATH)   { $script:IbArg = "/F `"$env:ONEC_IB_PATH`"" }
    else { throw 'Не задана ONEC_IB_PATH (файловая ИБ) или ONEC_IB_SERVER (сервер\база)' }
    $script:AuthArgs = if ($env:ONEC_USER) { @("/N `"$env:ONEC_USER`"", "/P `"$env:ONEC_PASSWORD`"") } else { @('/WA+') }

    if (Invoke-Git status --porcelain -- src/cf) {
        Write-Warning 'В erp-base остались следы прерванной выгрузки — откат и полная выгрузка.'
        Invoke-Git reset -q --hard HEAD | Out-Null
        Invoke-Git clean -fdxq -- src/cf | Out-Null
        $Full = $true
    }
    $mode = if ($Full -or -not (Test-Path (Join-Path $CfDir 'ConfigDumpInfo.xml'))) { 'full' } else { 'update' }
    if ($mode -eq 'full') { Clear-CfDir }

    $log  = Join-Path $LogDir "export-$stamp-$mode.log"
    $code = Invoke-Dump $mode $log
    if ($code -ne 0 -and $mode -eq 'update') {
        Write-Warning "Инкрементальная выгрузка: код $code — повтор полной."
        $mode = 'full'; Clear-CfDir
        $log  = Join-Path $LogDir "export-$stamp-$mode.log"
        $code = Invoke-Dump $mode $log
    }
    if ($code -ne 0) { throw "1cv8 завершился с кодом $code, лог: $log" }

    Invoke-Git add -A -- src/cf | Out-Null
    $changed = @(Invoke-Git diff --cached --name-only -- src/cf).Count
    if ($changed -gt 0) {
        Invoke-Git -c 'user.name=1C export' -c 'user.email=export@localhost' commit -q -m "config: выгрузка $mode $stamp ($changed файлов)" | Out-Null
    }
    $commit = @(Invoke-Git rev-parse --short HEAD)[0]
    $now = (Get-Date).ToString('o')
    Write-Status @{ status='ok'; mode=$mode; started=$started.ToString('o'); finished=$now; last_ok=$now;
                    changed_files=$changed; commit=$commit; log=$log; message=$null }
    Write-Host "OK: режим $mode, изменено файлов: $changed, коммит $commit"
}
catch {
    Write-Status @{ status='error'; mode=$mode; started=$started.ToString('o'); finished=(Get-Date).ToString('o');
                    last_ok=$prevOk; changed_files=0; commit=$null; log=$log; message="$_" }
    throw
}
finally {
    $lock.Dispose()
    Get-ChildItem $LogDir -Filter 'export-*.log' |
        Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-30) } | Remove-Item -Force
}
```

Файл создаётся молча. Проверка синтаксиса без запуска выгрузки: `powershell -NoProfile -Command "[void][ScriptBlock]::Create((Get-Content C:\1C\erp\scripts\export-config.ps1 -Raw))"` — тишина значит, что синтаксис верный.

> **Инфо:** `Set-StrictMode -Version Latest` превращает обращение к неинициализированной переменной в ошибку. `Start-Process -Wait -PassThru` нужен потому, что `1cv8.exe` — оконное приложение: при запуске через `&` PowerShell не ждёт его завершения и не видит код возврата. `-WindowStyle Hidden` скрывает окно Конфигуратора. Ключи 1С: `DESIGNER` — режим Конфигуратора; `/F` и `/S` — файловая и серверная ИБ; `/N` и `/P` — пользователь и пароль; `/WA+` — вход через учётку Windows; `/DisableStartupDialogs` и `/DisableStartupMessages` подавляют окна, которые иначе повесят пакетный запуск; `/Out` — файл журнала; `/DumpConfigToFiles <каталог>` — выгрузка в XML; `-Format Hierarchical` — иерархический формат (английское значение — без проблем с кодировкой); `-update` — только изменённые объекты. Блокировка: файл открывается с `FileShare None`, второй экземпляр получает ошибку открытия и выходит. **Деструктивные действия:** `git reset --hard` и `git clean -fdx` выполняются только в `erp-base` — этот worktree принадлежит скрипту, вручную там ничего не правится.

### 1.4 Первая полная выгрузка [А]

Первая выгрузка — сразу через скрипт, а не через меню Конфигуратора: так заодно проверяется весь конвейер. На ERP это 25–45 минут. Агент запускает в **новом** окне PowerShell (там видны переменные из 1.1).

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File C:\1C\erp\scripts\export-config.ps1 -Full
Get-Content C:\1C\logs\last-dump.json
(Get-ChildItem C:\1C\erp-base\src\cf -Recurse -File).Count
git -C C:\1C\erp-base log --oneline -3
```

Успех — строка `OK: режим full, изменено файлов: N…`, в `last-dump.json` `"status": "ok"`, в `log` новый коммит `config: выгрузка full …`. Ошибка — смотри файл из поля `log` в `last-dump.json`. Типовые причины: неверный пароль, ИБ заблокирована монопольно, неверный путь к ИБ.

> **Инфо:** `-ExecutionPolicy Bypass` снимает политику только для этого запуска и не действует, если политика задана GPO (шаг 0.2). `-NoProfile` не загружает профиль пользователя — запуск воспроизводимее и быстрее.

### 1.5 Проверка инкрементальной выгрузки и параллельной работы [Р]

Контрольная точка Go/No-Go для `-update` и для работы при открытом Конфигураторе. Открой Конфигуратор ИБ разработки и **не закрывай** его. В любом общем модуле добавь строку-комментарий `// проверка выгрузки`, сохрани конфигурацию (Ctrl+S; «Обновить конфигурацию БД» не нужно).

Результат проверит следующий шаг.

### 1.6 Повторная выгрузка: проверка результата [А]

Скрипт должен сам выбрать режим `update`, отработать за минуты и закоммитить ровно изменённый модуль.

```powershell
Measure-Command { powershell -NoProfile -ExecutionPolicy Bypass -File C:\1C\erp\scripts\export-config.ps1 } | Select-Object TotalMinutes
git -C C:\1C\erp-base show --stat --oneline HEAD
```

Успех — `режим update`, время 1–5 мин, в `--stat` один `Module.bsl` (плюс, возможно, служебный `Configuration.xml`). Если 1cv8 вернул ошибку блокировки при открытом Конфигураторе — выгрузку можно делать только при закрытом Конфигураторе; ночное расписание это покрывает, а задачу `folderOpen` (шаг 1.9) включать не стоит. После проверки удали тестовый комментарий в Конфигураторе и сохрани.

### 1.7 Вливание base в main [А]

Агент и VS Code работают в `C:\1C\erp` (ветка `main`), а выгрузка пока только в `base`. Вливание приносит картину в рабочую копию.

```powershell
git -C C:\1C\erp merge base --no-edit
(Get-ChildItem C:\1C\erp\src\cf\Catalogs -Filter *.xml).Count
```

Успех — `merge` пишет `Merge made by…` (или `Fast-forward`), счётчик печатает число справочников. Конфликт при первом вливании невозможен; в дальнейшем конфликт — сигнал, что модуль менялся и в базе, и в `main` (§12.6).

### 1.8 Задачи VS Code [А]

Ручной запуск выгрузки, вливание и автозапуск при открытии папки — одна кнопка вместо команд. Агент создаёт `C:\1C\erp\.vscode\tasks.json`.

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "1С: выгрузить конфигурацию",
      "type": "shell",
      "command": "powershell -NoProfile -ExecutionPolicy Bypass -File \"${workspaceFolder}\\scripts\\export-config.ps1\"",
      "problemMatcher": []
    },
    {
      "label": "1С: выгрузить, если старше 12 ч",
      "type": "shell",
      "command": "powershell -NoProfile -ExecutionPolicy Bypass -File \"${workspaceFolder}\\scripts\\export-config.ps1\" -IfOlderThanHours 12",
      "runOptions": { "runOn": "folderOpen" },
      "problemMatcher": []
    },
    {
      "label": "1С: влить base в main",
      "type": "shell",
      "command": "git merge base --no-edit",
      "problemMatcher": []
    }
  ]
}
```

Проверка — `Ctrl+Shift+P` → «Tasks: Run Task» показывает три задачи. Автозапуск при открытии папки VS Code выполнит только после разрешения: «Tasks: Manage Automatic Tasks» → «Allow Automatic Tasks» (делаешь ты).

> **Инфо:** `${workspaceFolder}` — папка, открытая в VS Code (`C:\1C\erp`). `runOn: folderOpen` запускает задачу при каждом открытии проекта; благодаря `-IfOlderThanHours 12` повторные открытия в течение дня ничего не делают. `problemMatcher: []` отключает разбор вывода на ошибки компилятора — здесь он не нужен.

### 1.9 Ночное расписание [А]

Ночная выгрузка в 02:30 с догоняющим запуском, если ПК был выключен. Регистрируется под текущим пользователем, права администратора не нужны.

```powershell
$action   = New-ScheduledTaskAction -Execute 'powershell.exe' `
            -Argument '-NoProfile -ExecutionPolicy Bypass -File "C:\1C\erp\scripts\export-config.ps1"'
$trigger  = New-ScheduledTaskTrigger -Daily -At '02:30'
$settings = New-ScheduledTaskSettingsSet -StartWhenAvailable -MultipleInstances IgnoreNew `
            -ExecutionTimeLimit (New-TimeSpan -Hours 2)
Register-ScheduledTask -TaskPath '\1C\' -TaskName 'ExportConfig' -Action $action -Trigger $trigger -Settings $settings
Start-ScheduledTask -TaskPath '\1C\' -TaskName 'ExportConfig'
Get-ScheduledTaskInfo -TaskPath '\1C\' -TaskName 'ExportConfig' | Select-Object LastRunTime, LastTaskResult
```

Успех — `LastTaskResult` равен `0` через минуту-две после `Start-ScheduledTask` (выгрузка без изменений). `Access denied` при регистрации — Планировщик закрыт политикой; тогда остаётся задача `folderOpen` из 1.8, этого достаточно.

> **Инфо:** `-StartWhenAvailable` — если в 02:30 ПК был выключен, задача запустится при ближайшей возможности. `-MultipleInstances IgnoreNew` — не запускать второй экземпляр, пока идёт первый (дублирует lock скрипта). `-ExecutionTimeLimit 2h` — аварийная остановка зависшей выгрузки. Задача выполняется, только пока пользователь вошёл в систему; режим «без входа» требует хранить пароль Windows в задаче — для пилота не нужен.

---

## Фаза 2. Разработка сервера 1c-config

### 2.1 Каркас, контракты и тесты [А]

Исполнитель — Claude Code в репозитории 1CDEV, каталог `mcp-1c-config/` (раскладка — §8). Сначала контрактный тест и синтетическая мини-выгрузка, потом код. Это фиксирует инструменты §5.4 до реализации.

```bash
cd mcp-1c-config
python -m venv .venv && . .venv/bin/activate
pip install -e ".[test]"
pytest -q tests/test_server_contract.py   # до реализации — падает, это ожидаемо
```

Тест падает с перечнем отсутствующих инструментов — это нормальная стартовая точка: реализация доводит его до зелёного.

### 2.2 Реализация по модулям [А]

Порядок: `settings` → `safety` → `layout` → `index` → `metadata` → `bsl` → `search` → `status` → `limits` → `diagnostics` → `server` → `probe` → `__main__ --selftest`. После каждого модуля — его тесты. Директивы — §16.1. Claude Code в 1CDEV.

```bash
pytest -q
ruff check src tests
```

Успех — все тесты зелёные, линтер без замечаний. Тесты `diagnostics` с реальным jar помечены `@pytest.mark.bslls` и пропускаются, если не задана переменная `ONEC_BSLLS_JAR`.

### 2.3 Фиксация зависимостей и эталонов [А]

Закрепляем точные версии для офлайн-набора и кладём эталоны скриптов и шаблонов в `deploy/` (§8). Claude Code в 1CDEV.

```bash
pip freeze --exclude-editable > requirements.lock
git add mcp-1c-config deploy && git commit -m "1c-config 1.0.0: сервер, тесты, шаблоны развёртывания"
```

Успех — `requirements.lock` содержит `mcp==<версия>` и все транзитивные зависимости с точными версиями. Колёса под Windows собираются уже на ПК-внеш (шаг 3.1): `pywin32` и бинарные колёса зависят от ОС.

---

## Фаза 3. Офлайн-набор и перенос

### 3.1 Сборка набора на внешней машине [Р]

Машина: ПК-внеш, Windows x64, Python **того же минора**, что на ПК-внутр (шаг 0.2), — иначе бинарные колёса (`pydantic-core`, `pywin32`) не встанут. Скрипт `deploy/build-kit.ps1` из репозитория 1CDEV скачивает и проверяет всё, что нужно контуру. Запускаешь ты: на ПК-внеш агента нет.

```powershell
$ErrorActionPreference = 'Stop'
$Kit = "$env:USERPROFILE\Desktop\1c-kit"; $Py = 'py -3.11'      # минор как на ПК-внутр
New-Item -ItemType Directory -Force "$Kit\wheels" | Out-Null
git clone --depth 1 https://github.com/soldatovde1979-ai/1CDEV "$Kit\src"
Invoke-Expression "$Py -m pip download -r `"$Kit\src\mcp-1c-config\requirements.lock`" -d `"$Kit\wheels`""
Invoke-Expression "$Py -m pip wheel `"$Kit\src\mcp-1c-config`" --no-deps -w `"$Kit\wheels`""
Invoke-WebRequest 'https://api.adoptium.net/v3/binary/latest/21/ga/windows/x64/jdk/hotspot/normal/eclipse?project=jdk' -OutFile "$Kit\jdk-21.zip" -UseBasicParsing
$rel = Invoke-RestMethod 'https://api.github.com/repos/1c-syntax/bsl-language-server/releases/latest'
$jar = $rel.assets | Where-Object name -like '*-exec.jar' | Select-Object -First 1
Invoke-WebRequest $jar.browser_download_url -OutFile "$Kit\$($jar.name)" -UseBasicParsing
"bsl-language-server $($rel.tag_name)" | Set-Content "$Kit\VERSIONS.txt"
Get-ChildItem $Kit -Recurse -File -Exclude SHA256SUMS.txt | Where-Object FullName -notlike "$Kit\src\*" |
  Get-FileHash -Algorithm SHA256 | ForEach-Object { "$($_.Hash)  $($_.Path.Substring($Kit.Length+1))" } |
  Set-Content "$Kit\SHA256SUMS.txt"
Get-ChildItem $Kit -File | Select-Object Name, @{n='MB';e={[math]::Round($_.Length/1MB,1)}}
```

Успех — в `1c-kit` лежат `wheels\` (с `onec_config-1.0.0-py3-none-any.whl`), `jdk-21.zip` (~190 МБ), `bsl-language-server-<ver>-exec.jar`, `VERSIONS.txt`, `SHA256SUMS.txt`. Если `pip download` пишет «No matching distribution» — минор Python на ПК-внеш не совпадает с ПК-внутр.

> **Инфо:** `pip download -r` скачивает ровно закреплённые версии со всеми зависимостями под текущие ОС и Python — поэтому набор собирается на такой же Windows. `pip wheel --no-deps` собирает колесо самого сервера без повторного скачивания зависимостей. API Adoptium отдаёт последний GA-релиз JDK 21 в zip. `*-exec.jar` — самодостаточный jar BSL LS со всеми зависимостями. `SHA256SUMS.txt` позволит проверить, что файлы не повредились при пересылке. Если на ПК-внутр нет VS Code-расширения Language 1C (BSL), его `.vsix` скачивается со страницы расширения в Marketplace («Download Extension»).

### 3.2 Перенос во внутренний контур [Р]

Яндекс Почта пропускает вложения примерно до 25 МБ, всё крупнее превращает в ссылку на Диск, а Диск может быть закрыт в контуре. Поэтому крупное режем на тома.

На ПК-внеш: 7-Zip → «Добавить к архиву», формат 7z, «Разбить на тома» `20M` — для `jdk-21.zip` и jar. Остальное (`wheels`, `VERSIONS.txt`, `SHA256SUMS.txt`, `deploy\`) — одним архивом. Отправь письма самому себе; на ПК-внутр сохрани все вложения в `C:\Temp\1c-kit\` и распакуй (открыть `.7z.001` — 7-Zip соберёт тома).

### 3.3 Проверка целостности и раскладка [А]

Агент сверяет контрольные суммы и раскладывает компоненты по местам из §5.1. Машина: ПК-внутр.

```powershell
$ErrorActionPreference = 'Stop'
$Kit = 'C:\Temp\1c-kit'
Get-Content "$Kit\SHA256SUMS.txt" | ForEach-Object {
  $h, $f = $_ -split '  ', 2
  if ((Get-FileHash "$Kit\$f" -Algorithm SHA256).Hash -ne $h) { throw "Хэш не совпал: $f" }
}
'Все хэши совпали'
Expand-Archive "$Kit\jdk-21.zip" C:\1C\tools\jdk-tmp -Force
Move-Item (Get-ChildItem C:\1C\tools\jdk-tmp -Directory | Select-Object -First 1).FullName C:\1C\tools\jdk-21
Remove-Item C:\1C\tools\jdk-tmp
New-Item -ItemType Directory -Force C:\1C\tools\bslls | Out-Null
Copy-Item (Get-ChildItem $Kit -Filter '*-exec.jar').FullName C:\1C\tools\bslls\bsl-language-server.jar
Copy-Item "$Kit\VERSIONS.txt" C:\1C\tools\VERSIONS.txt
& C:\1C\tools\jdk-21\bin\java.exe -version
```

Успех — «Все хэши совпали» и `openjdk version "21…`. «Хэш не совпал» — файл повреждён при пересылке, перешли его заново.

> **Инфо:** zip Temurin содержит одну папку вида `jdk-21.0.x+y`; её перемещение в `jdk-21` даёт стабильный путь, на который ссылаются `mcp.json` и сервер. Если в `C:\1C\tools\jdk-21` уже был JDK, `Move-Item` упадёт — тогда сначала сравни версии и реши, заменять ли.

---

## Фаза 4. Установка и подключение

### 4.1 Установка сервера 1c-config в venv [А]

Изолированное окружение Python только для сервера, установка строго из локальных колёс. Машина: ПК-внутр.

```powershell
python -m venv C:\1C\tools\1c-config\.venv
C:\1C\tools\1c-config\.venv\Scripts\python.exe -m pip install --no-index --find-links C:\Temp\1c-kit\wheels onec-config==1.0.0
C:\1C\tools\1c-config\.venv\Scripts\python.exe -c "import onec_config; print(onec_config.__version__, onec_config.API_VERSION)"
```

Успех — последняя строка печатает `1.0.0 1`. Ошибка `No matching distribution` — колёса собраны под другой минор Python (шаг 3.1).

> **Инфо:** `--no-index` запрещает обращаться к PyPI, `--find-links` указывает папку с колёсами — установка полностью офлайн. venv не трогает системный Python и PATH.

### 4.2 Самопроверка сервера на реальной выгрузке [А]

Go/No-Go для карты формата §5.2: `--selftest` строит индекс по реальной выгрузке, сверяет число объектов каждого вида с числом `*.xml` в каталогах и прогоняет по одному вызову каждого инструмента.

```powershell
$env:ONEC_CF_DIR = 'C:\1C\erp\src\cf'; $env:ONEC_REPO_DIR = 'C:\1C\erp'
$env:ONEC_DUMP_STATUS = 'C:\1C\logs\last-dump.json'
$env:ONEC_JAVA = 'C:\1C\tools\jdk-21\bin\java.exe'; $env:ONEC_BSLLS_JAR = 'C:\1C\tools\bslls\bsl-language-server.jar'
C:\1C\tools\1c-config\.venv\Scripts\python.exe -m onec_config --selftest
```

Успех — таблица «вид → объектов в индексе → файлов на диске» без расхождений, время каждого инструмента в пределах §13 и итог `SELFTEST OK`. Расхождение по виду означает ошибку в `layout.py`: вывод selftest отправь в сессию Claude Code (1CDEV) для исправления, дальше не иди.

### 4.3 BSL Language Server: версия, анализ, MCP-инструменты [А]

Три проверки: jar запускается на JDK 21; `analyze` работает (на нём стоит `check_module`); нативный MCP-режим отдаёт инструменты. Последняя — Go/No-Go для сервера `bsl-ls` в `mcp.json`.

```powershell
$J = 'C:\1C\tools\jdk-21\bin\java.exe'; $Jar = 'C:\1C\tools\bslls\bsl-language-server.jar'
& $J -jar $Jar --version
& $J -jar $Jar analyze --help | Select-Object -First 30
C:\1C\tools\1c-config\.venv\Scripts\python.exe -m onec_config.probe $J -Xmx4g -jar $Jar mcp --protocol stdio
```

Успех — номер версии; справка `analyze` с ключами `--srcDir`, `--reporter`, `--outputDir` (если ключи называются иначе, это ошибка `diagnostics.py`: передай справку в 1CDEV); `probe` печатает список инструментов, включая `register_workspace_folder` и `global_member_info`. Если `probe` упал или инструментов нет — в шаге 4.5 ставь `"disabled": true` для `bsl-ls`: слой «правила» всё равно закрыт через `check_module`.

> **Инфо:** `probe` — утилита из пакета `onec_config`: запускает любой stdio MCP-сервер, выполняет `initialize` и `tools/list`, печатает имена и описания инструментов и закрывает процесс. `-Xmx4g` — предел памяти JVM; индексация ERP в MCP-режиме требует 2–4 ГБ. Зафиксируй версию BSL LS в `C:\1C\tools\VERSIONS.txt`, если её там нет (вопрос В-6).

### 4.4 Конфигурации проекта: BSL LS, .rooignore, настройки VS Code [А]

Общий конфиг диагностик для `check_module` и для расширения VS Code (одинаковые правила у агента и человека), исключения для агента (§10) и для поиска VS Code. Агент создаёт файлы в `C:\1C\erp`.

`.bsl-language-server.json`:

```json
{
  "$schema": "https://1c-syntax.github.io/bsl-language-server/configuration/schema.json",
  "language": "ru",
  "diagnostics": { "computeTrigger": "onSave", "skipSupport": "withSupportLocked" }
}
```

`.rooignore`:

```gitignore
**/*.bin
**/Ext/Template*
**/Ext/Picture/**
**/ConfigDumpInfo.xml
.env
*.log
```

`.vscode/settings.json`:

```json
{
  "search.exclude": { "**/*.bin": true, "**/Ext/Picture/**": true, "**/Ext/Template*": true },
  "files.watcherExclude": { "**/src/cf/**/Ext/Template*": true, "**/src/cf/**/Ext/Picture/**": true },
  "files.encoding": "utf8bom"
}
```

Файлы создаются молча; проверка — `git -C C:\1C\erp status` показывает три новых файла.

> **Инфо:** `skipSupport: withSupportLocked` отключает замечания в объектах на замке поддержки поставщика — в типовой ERP их тысячи, и это не твой код. `files.encoding: utf8bom` — новые `.bsl` VS Code сохраняет с BOM, как ожидает платформа. Если в разделе настроек Zoo/Roo включена индексация кодовой базы (Codebase Indexing), выключи её для этого проекта: 100 тыс. файлов индексироваться будут часами.

### 4.5 Подключение серверов в .roo/mcp.json [А]

Регистрация обоих серверов у агента — блок из §14 с абсолютными путями. Агент создаёт `C:\1C\erp\.roo\mcp.json`; если файл уже есть — дописывает блоки **внутрь существующего** `mcpServers`, второй ключ `mcpServers` не создаёт.

```json
{
  "mcpServers": {
    "1c-config": {
      "command": "C:/1C/tools/1c-config/.venv/Scripts/python.exe",
      "args": ["-m", "onec_config"],
      "env": {
        "ONEC_CF_DIR": "C:/1C/erp/src/cf",
        "ONEC_REPO_DIR": "C:/1C/erp",
        "ONEC_DUMP_STATUS": "C:/1C/logs/last-dump.json",
        "ONEC_JAVA": "C:/1C/tools/jdk-21/bin/java.exe",
        "ONEC_BSLLS_JAR": "C:/1C/tools/bslls/bsl-language-server.jar",
        "ONEC_BSLLS_CONFIG": "C:/1C/erp/.bsl-language-server.json",
        "ONEC_BSLLS_JAVA_OPTS": "-Xmx2g"
      },
      "timeout": 120,
      "alwaysAllow": ["config_status", "list_objects", "find_object", "describe_object",
                      "list_module_methods", "read_module", "search_code", "check_module"]
    },
    "bsl-ls": {
      "command": "C:/1C/tools/jdk-21/bin/java.exe",
      "args": ["-Xmx4g", "-jar", "C:/1C/tools/bslls/bsl-language-server.jar", "mcp", "--protocol", "stdio"],
      "timeout": 120,
      "disabled": false
    }
  }
}
```

Проверка — `python -m json.tool C:\1C\erp\.roo\mcp.json` печатает файл без ошибки разбора. `"disabled": true` у `bsl-ls` — если шаг 4.3 не прошёл.

> **Инфо:** `alwaysAllow` — инструменты, которые агент вызывает без подтверждения; здесь только read-only. `timeout` — сколько секунд агент ждёт ответа инструмента (по умолчанию у Roo 60, у `check_module` холодный старт JVM бывает дольше). Паролей в файле нет и не будет: учётка 1С нужна только скрипту выгрузки.

### 4.6 Расширение Language 1C (BSL) в VS Code [Р]

Подсветка и панель Problems для тебя, на том же jar и JDK, что у агента.

Установи «Language 1C (BSL)» (издатель 1c-syntax): Marketplace или «Extensions» → «…» → «Install from VSIX…» с файлом из набора. В настройках расширения укажи путь к Java `C:\1C\tools\jdk-21\bin\java.exe` и внешний jar `C:\1C\tools\bslls\bsl-language-server.jar` (названия параметров смотри в описании расширения: они менялись между версиями). Открой любой `Module.bsl` — появились подсветка и замечания в Problems.

### 4.7 Перезапуск и проверка подключения [Р]

`Ctrl+Shift+P` → «Developer: Reload Window». Открой панель Zoo/Roo Code → значок MCP-серверов. Должны быть зелёными `1c-config` (8 инструментов) и `bsl-ls` (если включён). Красный индикатор — открой лог сервера в той же панели и передай текст агенту.

---

## Фаза 5. Правила агента

### 5.1 Файлы правил .roo/rules [А]

Свои короткие тексты правил (§9, §16.2) — лицензионно чистые и написанные под наши инструменты. Агент создаёт три файла в `C:\1C\erp\.roo\rules\`; тексты берутся из `deploy/templates/rules/` набора.

| Файл | Содержание |
|---|---|
| `00-project.md` | Проект, раскладка `src/cf`, ветки `base`/`main`, `full_name` на русском |
| `10-mcp-tools.md` | Девять правил §16.2 дословно |
| `20-bsl-standards.md` | Выжимка стандартов своими словами: области `ПрограммныйИнтерфейс`/`СлужебныйПрограммныйИнтерфейс`/`СлужебныеПроцедурыИФункции`; строки пользователю через `НСтр`; запросы не в цикле; `Попытка…Исключение` с `ЗаписьЖурналаРегистрации` и `ПодробноеПредставлениеОшибки`; минимум серверных вызовов, `&НаСервереБезКонтекста` где можно; без `Сообщить` в серверном коде |

```powershell
New-Item -ItemType Directory -Force C:\1C\erp\.roo\rules | Out-Null
Copy-Item C:\Temp\1c-kit\deploy\templates\rules\*.md C:\1C\erp\.roo\rules\
Get-ChildItem C:\1C\erp\.roo\rules
```

Успех — три файла в списке. Zoo/Roo Code подхватывает `.roo/rules/*.md` во всех режимах без отдельного включения.

### 5.2 Проверка, что правила работают [Р]

В чате агента: «Что ты обязан сделать после правки модуля?» Успех — ответ «вызвать `check_module`…» со ссылкой на правило, а не общие слова. Затем: «Добавь реквизит в документ X» — агент должен отказаться править XML и описать, что сделать в Конфигураторе.

### 5.3 (Опционально, Ф2) ai_rules_1c [Р]

Только после подтверждения права использования (у репозитория нет лицензии) и только полнотекстовые файлы — их список в `plan_cloude.md`, Этап 6. Класть в `.roo/rules-code/` (только режим Code), чтобы не раздувать промпт остальных режимов.

---

## Фаза 6. Приёмка

### 6.1 Контрольный набор вопросов [Р]

Задай агенту 10 вопросов, подставив реальные объекты своей конфигурации. Ожидаемый инструмент виден в чате, ответ сверяется командой из шага 6.2 или в Конфигураторе.

| # | Вопрос | Ожидаемый инструмент | Сверка |
|---|---|---|---|
| 1 | Какая конфигурация выгружена и когда? | `config_status` | `last-dump.json` |
| 2 | Сколько справочников в конфигурации? | `list_objects` | счётчик файлов |
| 3 | Реквизиты и табличные части документа X | `describe_object` | Конфигуратор |
| 4 | Экспортные методы общего модуля Y | `list_module_methods` | Конфигуратор |
| 5 | Покажи ОбработкаПроведения документа X | `read_module(method)` | Конфигуратор |
| 6 | Где вызывается метод Y.Z? | `search_code` | `git grep -c` |
| 7 | Найди объекты про «контрагент» по синониму | `find_object` | — |
| 8 | Проверь модуль Y на стандарты | `check_module` | панель Problems |
| 9 | Сигнатура глобальной функции `СтрНайти` | `global_member_info` (`bsl-ls`) | синтакс-помощник |
| 10 | Опиши документ НесуществующийДокумент | `find_object` → «не найден» | **без выдумки — обязательно** |

Критерий: не меньше 9 из 10 ответов через инструменты; вопрос 10 — строго «не найден». Если `bsl-ls` выключен, вопрос 9 не учитывается и критерий — 8 из 9.

### 6.2 Сверочные команды [А]

Независимая проверка ответов агента по файлам, без MCP.

```powershell
(Get-ChildItem C:\1C\erp\src\cf\Catalogs -Filter *.xml).Count
git -C C:\1C\erp grep -c -F "<ИмяМетода>(" -- src/cf
Get-Content C:\1C\logs\last-dump.json
```

Числа совпадают с ответами агента на вопросы 2, 6 и 1 — приёмка инструментов пройдена.

### 6.3 Цикл правки [Р]

Сквозной сценарий §12.2: попроси агента внести небольшую правку в модуль (например, добавить проверку заполнения). Агент правит `.bsl` и вызывает `check_module`. Сделай `git diff` и закоммить в `main`, перенеси правку в Конфигуратор (копированием модуля или шагом 7.2), сохрани, запусти задачу «1С: выгрузить», затем «1С: влить base в main». Успех — вливание прошло **без конфликта**, `git -C C:\1C\erp diff base` по этому модулю пуст.

---

## Фаза 7. Обратный контур (опционально, Ф1.5)

### 7.1 Скрипт load-changes.ps1 [А]

Частичная загрузка в **ИБ разработки** только тех файлов, которые есть в `main`, но ещё не попали в `base`. По умолчанию только показывает список (dry-run); запись — ключом `-Apply`, запускает человек. Агент сохраняет `C:\1C\erp\scripts\load-changes.ps1` (UTF-8 с BOM).

```powershell
# load-changes.ps1 — частичная загрузка изменений main в ИБ разработки
# VERSION = "1.0". По умолчанию dry-run. Запись в ИБ — только с -Apply.
[CmdletBinding()]
param([switch]$Apply, [switch]$UpdateDB)
$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

$Root  = if ($env:ONEC_ROOT) { $env:ONEC_ROOT } else { 'C:\1C' }
$Repo  = Join-Path $Root 'erp'
$CfDir = Join-Path $Repo 'src\cf'
$Log   = Join-Path $Root ("logs\load-{0}.log" -f (Get-Date -Format 'yyyyMMdd-HHmmss'))

if (git -C $Repo status --porcelain -- src/cf) { throw 'В main есть незакоммиченные изменения src/cf — сначала review и коммит.' }
$deleted = @(git -C $Repo -c core.quotepath=false diff --name-only --diff-filter=D base...HEAD -- src/cf)
if ($deleted.Count) { throw "Удаления объектов загрузкой не переносятся — сделай в Конфигураторе:`n$($deleted -join "`n")" }
$files = @(git -C $Repo -c core.quotepath=false diff --name-only --diff-filter=AM base...HEAD -- src/cf)
if (-not $files.Count) { Write-Host 'Нет изменений относительно base.'; exit 0 }

Write-Host "К загрузке ($($files.Count)):"; $files | ForEach-Object { Write-Host "  $_" }
if (-not $Apply) { Write-Host 'Dry-run: в ИБ ничего не записано. Для загрузки: -Apply [-UpdateDB]'; exit 0 }

$list = [IO.Path]::GetTempFileName()
$files | ForEach-Object { Join-Path $Repo ($_ -replace '/', '\') } | Set-Content $list -Encoding UTF8
$ib   = if ($env:ONEC_IB_SERVER) { "/S `"$env:ONEC_IB_SERVER`"" } else { "/F `"$env:ONEC_IB_PATH`"" }
$auth = if ($env:ONEC_USER) { "/N `"$env:ONEC_USER`" /P `"$env:ONEC_PASSWORD`"" } else { '/WA+' }
$cmd  = "DESIGNER $ib $auth /DisableStartupDialogs /DisableStartupMessages /Out `"$Log`" " +
        "/LoadConfigFromFiles `"$CfDir`" -listFile `"$list`" -Format Hierarchical -partial -updateConfigDumpInfo"
if ($UpdateDB) { $cmd += ' /UpdateDBCfg' }
try {
    $p = Start-Process -FilePath $env:ONEC_BIN -ArgumentList $cmd -Wait -PassThru -WindowStyle Hidden
    if ($p.ExitCode -ne 0) { throw "1cv8 завершился с кодом $($p.ExitCode), лог: $Log" }
    Write-Host "Загружено. Лог: $Log"
} finally { Remove-Item $list -Force }
```

Проверка без записи: `powershell -NoProfile -ExecutionPolicy Bypass -File C:\1C\erp\scripts\load-changes.ps1` печатает список файлов и «Dry-run: в ИБ ничего не записано».

> **Инфо:** `base...HEAD` (три точки) — изменения `main` от общей точки с `base`, то есть ровно то, чего ещё нет в ИБ. `--diff-filter=AM` — только добавленные и изменённые файлы; удаления скрипт отказывается переносить. Ключи 1С: `/LoadConfigFromFiles <каталог>` — загрузка из XML; `-listFile` — загружать только файлы из списка; `-partial` — частичная загрузка; `-updateConfigDumpInfo` — обновить служебный файл выгрузки; `/UpdateDBCfg` — сразу обновить конфигурацию базы данных (нужен монопольный доступ к ИБ). Формат путей в `-listFile` (абсолютные или от каталога) проверить на своей версии платформы по первому запуску [GO]. Если ИБ подключена к хранилищу, загружаемые объекты должны быть захвачены — иначе загрузка завершится ошибкой (вопрос В-1). **Предупреждение:** `-Apply` изменяет основную конфигурацию ИБ разработки; запускать только на ИБ разработки, никогда на рабочей.

### 7.2 Первая загрузка [Р]

Сделай копию ИБ разработки (для файловой — копия каталога ИБ при закрытых сеансах), затем в новом окне PowerShell:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File C:\1C\erp\scripts\load-changes.ps1 -Apply
```

Открой Конфигуратор: изменённый модуль содержит правку из `main`; основная конфигурация помечена изменённой. «Обновить конфигурацию БД» — вручную (F7) или повторный запуск с `-UpdateDB`. Ошибка — лог из вывода, ИБ восстанавливается из копии.

---

## Дальше: Ф2 и Ф3

Ф2 (расширение) и Ф3 (команда) — по §2 и §17 манифеста: live-доступ только к обезличенной копии ИБ (выбор между `1c_mcp` и `mcp-1c` — после проверки зрелости), `rlm-tools-bsl` при нехватке скорости поиска на ERP, CI-анализ `bsl-language-server analyze` на pull request, общий `mcp.json` с `${env:ONEC_TOOLS}`.

---

## Что это даёт и чего не делает

**Даёт:** ежедневную автоматическую картину конфигурации в git, которая не затирает правки; агента, который отвечает о метаданных и коде по фактам выгрузки, читает код по методам, ищет вызовы и проверяет каждую свою правку диагностиками BSL LS; одинаковые правила для агента и человека; полностью офлайн-установку без прав администратора.

**Не делает:** не пишет в ИБ без явного `-Apply` от человека; не правит XML метаданных и форм; не даёт агенту данные ИБ (это Ф2, только обезличенная копия); не убирает вопрос передачи кода в DeepSeek — его закрывает согласование ИБ (вопрос В-3).

# research

Консолидированный скилл для глубокого исследования тем (в первую очередь —
MCP и 1С).

SKILL.md собран из двух проверенных публичных скиллов:

- altmbr/claude-research-skill — оркестрация параллельных агентов, скелеты
  файлов, протокол «поиск → запись», восстановление зависших агентов.
  https://github.com/altmbr/claude-research-skill (MIT)
- daymade/claude-code-skills (deep-research V6) — управление источниками,
  типы и доступность источников, даты AS_OF, реестр цитирования,
  контр-рецензия, анти-галлюцинационные правила.
  https://github.com/daymade/claude-code-skills

Из daymade взята только методология; привязка к его скриптам (research_assets.py,
provider_runs.py) и вендорным маршрутам (ChatGPT/Kimi/UniFuncs) отброшена.
Скилл адаптирован под инструменты Hermes: web_search, web_extract,
delegate_task, write_file/read_file.

Проверенные, но не вошедшие кандидаты:

- kaynquang/multi-agent-research — 13 агентов, 7 фаз; избыточен, его идеи
  (уточнение вопроса, верификация источников, adversarial-review) покрыты выше.
- claude-office-skills/skills (deep-research) — методология попроще (PESTEL/SWOT),
  без управления источниками; полезна как запасной фреймворк.

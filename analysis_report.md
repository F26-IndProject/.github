# АРХИТЕКТУРНЫЙ АНАЛИЗ ПРОЕКТА LISA

> Дата: 2026-03-18
> Статус: Честная production-оценка
> Дедлайн: 5 месяцев

---

## ЧАСТЬ 4: ДЕТАЛЬНЫЙ АНАЛИЗ ПО ФАЙЛАМ

---

### ФАЙЛ: `/workspaces/WinAgent/agent.py`
**НАЗНАЧЕНИЕ:** Главный исполняемый файл Windows-агента
**СТРОК КОДА:** ~350

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✓ Singleton через lock-файл
- ✓ Heartbeat в отдельном потоке (threading.Thread)
- ✓ Случайный выбор действий по весам
- ~ `is_work_time()` существует, но не подключена к главному циклу
- ~ try/except вокруг действий есть, но не везде
- ✗ Нет ролей бухгалтер/менеджер/ИТ-инженер (есть user/admin/dev)
- ✗ Нет Humanizer (Гаусс-паузы, суточный ритм, мышь Безье)
- ✗ Нет Social graph / Event Bus
- ✗ Нет IMAP/SMTP, нет SMB/RDP
- ✗ `next_heartbeat_in: 86400` — heartbeat раз в 24 часа вместо 5 минут

**КРИТИЧЕСКИЕ ПРОБЛЕМЫ:**

```
Файл: WinAgent/agent.py (heartbeat интервал)
Текущий код: next_heartbeat_in=86400 (24ч) в ответе сервера
НО агент спит 300 сек между отправками → рассинхрон
```

```
Файл: WinAgent/client/server_api.py
BASE_URL = "http://localhost:8000/api"  ← захардкожен
Нужно: читать из .env или settings.yaml
```

**СООТВЕТСТВИЕ UML:** Частично соответствует блоку "Windows Agent" — работает как агент, но не как инжектированный в explorer.exe поток

---

### ФАЙЛ: `/workspaces/WinAgent/actions/apps.py`
**НАЗНАЧЕНИЕ:** Запуск Windows-приложений
**СТРОК КОДА:** ~80

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✓ `os.startfile()` используется для открытия файлов
- ✗ `subprocess.Popen` используется для запуска приложений — **python.exe будет родителем**
- ✗ Нет Shell.Application COM
- ✗ `open_browser` запускает браузер через Popen — нарушение требования 4
- ✗ `run_terminal_command` использует `subprocess.Popen(["powershell.exe", ...])` — прямое нарушение

**КРИТИЧЕСКИЕ ПРОБЛЕМЫ:**

```python
# Текущий код (нарушение Требования 4):
subprocess.Popen(["powershell.exe", "-Command", cmd])
# Родитель powershell.exe = python.exe ← НЕЛЬЗЯ

# Нужно:
import win32com.client
shell = win32com.client.Dispatch("Shell.Application")
shell.ShellExecute("powershell.exe", cmd, "", "open", 1)
# Родитель powershell.exe = explorer.exe ✓
```

---

### ФАЙЛ: `/workspaces/WinAgent/actions/gui.py`
**НАЗНАЧЕНИЕ:** GUI-автоматизация через pyautogui
**СТРОК КОДА:** ~30

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ~ `simulate_typing()` есть, но без опечаток и исправлений
- ✗ Нет случайной скорости печати (Гаусс)
- ✗ Нет мыши по кривой Безье
- ✗ Нет тремора ±2px
- ✗ Нет суточного ритма (×0.85 утром, ×1.35 после обеда)

---

### ФАЙЛ: `/workspaces/WinAgent/client/server_api.py`
**НАЗНАЧЕНИЕ:** HTTP-клиент к LISA Backend
**СТРОК КОДА:** ~60

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✗ `BASE_URL = "http://localhost:8000/api"` — захардкожен
- ✗ Нет `GET /api/agents/{id}/pending` — агент не получает команды
- ✗ Нет `POST /api/events` — агент не отправляет события
- ✓ `POST /api/agent_activities` — активности отправляются

---

### ФАЙЛ: `/workspaces/WinAgent/config/settings.yaml`
**НАЗНАЧЕНИЕ:** Конфигурация агента
**СТРОК КОДА:** ~15

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ~ `work_start: 09:00`, `work_end: 18:00`, `work_days: [1,2,3,4,5]` — есть
- ✗ Нет `lunch_start`, `lunch_duration`
- ✗ Нет `arrival_variance: ±20 min`
- ✗ Нет `holidays: []` / `ical_url`
- ✗ Нет `micro_break_frequency`

---

### ФАЙЛ: `/workspaces/WinAgent/roles/admin.yaml`, `dev.yaml`, `user.yaml`
**НАЗНАЧЕНИЕ:** Роли агентов
**СТРОК КОДА:** ~20-30 каждый

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✗ Нет ролей: **бухгалтер** (accountant), **менеджер** (manager), **ИТ-инженер** (it_engineer)
- ✗ Нет 10-20 URL в каждой роли (в текущих ролях 2-3 URL)
- ✗ Нет весов действий (weights)
- ✗ Нет расписания внутри YAML роли

---

### ФАЙЛ: `/workspaces/backend/app/main.py`
**НАЗНАЧЕНИЕ:** Точка входа FastAPI-приложения
**СТРОК КОДА:** ~80

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✓ CORS настроен
- ✓ Все роутеры подключены
- ~ GET `/api/dashboard/stats` подключён, но данные из БД могут быть нули при пустой БД
- ✗ Нет `GET /api/events/stream` (SSE)
- ✗ Нет роутера для `POST /api/events`
- ✗ Нет `GET /api/agents/{id}/pending`

---

### ФАЙЛ: `/workspaces/backend/app/models/models.py`
**НАЗНАЧЕНИЕ:** SQLAlchemy модели БД
**СТРОК КОДА:** ~150

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✓ Agent, AgentBuild, AgentActivity, Role, BehaviorTemplate
- ✗ Нет модели `AgentEvent` (для Event Bus)
- ✗ Нет модели `SocialGraph` (кто с кем общается)
- ✗ Нет поля `pending_commands` у Agent или отдельной таблицы `AgentCommand`
- ✗ `Servers` хранит пароль в открытом виде в БД

---

### ФАЙЛ: `/workspaces/backend/app/api/endpoints/agents.py`
**НАЗНАЧЕНИЕ:** API эндпоинты управления агентами
**СТРОК КОДА:** ~200

**КРИТИЧЕСКАЯ ПРОБЛЕМА — деплой:**
```python
# Текущий код (строки ~120-150):
config_path = f"/tmp/shared_configs/deploy_{agent_id}.json"
with open(config_path, 'w') as f:
    json.dump(deploy_config, f)
return {"status": "deploy_task_created", "config_path": config_path}
# ← просто пишет JSON файл на локальный диск сервера!

# Нужно: реальный SSH деплой через paramiko
import paramiko
ssh = paramiko.SSHClient()
ssh.connect(host, username=user, password=password)
ssh.exec_command(f"chmod +x /tmp/agent && nohup /tmp/agent &")
```

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✗ POST `/agents/generate` создаёт JSON в `/tmp/build-{id}.json` — не реальный бинарник
- ✗ POST `/agents/{id}/deploy` пишет файл в `/tmp` вместо SSH
- ✗ Нет POST `/api/events`
- ✗ Нет GET `/api/agents/{id}/pending`

---

### ФАЙЛ: `/workspaces/backend/app/api/endpoints/heartbeat.py`
**НАЗНАЧЕНИЕ:** Приём heartbeat от агентов
**СТРОК КОДА:** ~120

**КРИТИЧЕСКАЯ ПРОБЛЕМА:**
```python
# Текущий код:
"next_heartbeat_in": 86400  # 24 часа!

# Нужно:
"next_heartbeat_in": 300  # 5 минут
```

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✓ Авторизация по Bearer токену
- ✓ Создание агента при первом heartbeat
- ✓ Сохранение активности
- ✗ `next_heartbeat_in: 86400` — агент будет считаться "оффлайн" сразу
- ~ Авторизационный ключ `sk-agent-heartbeat-key-2024` захардкожен в коде

---

### ФАЙЛ: `/workspaces/backend/app/api/endpoints/builds.py`
**НАЗНАЧЕНИЕ:** API управления сборками
**СТРОК КОДА:** ~80

**КРИТИЧЕСКАЯ ПРОБЛЕМА — заглушка:**
```python
# Текущий код:
new_build = AgentBuild(
    agent_id=agent.id,
    build_status=BuildStatus.READY,  # ← СРАЗУ "готово"!
    binary_path="/tmp/fake_agent.exe",  # ← фейковый путь
    build_log="Build completed successfully"
)
# Нет реального вызова сборщика!

# Нужно: запустить реальный build pipeline
import subprocess
result = subprocess.run(
    ["python", "-m", "PyInstaller", "--onefile", "agent.py"],
    capture_output=True, timeout=300
)
```

---

### ФАЙЛ: `/workspaces/backend/app/database.py`
**НАЗНАЧЕНИЕ:** Конфигурация БД
**СТРОК КОДА:** ~20

**КРИТИЧЕСКАЯ ПРОБЛЕМА:**
```python
# Текущий код:
DATABASE_URL = "postgresql://lisa:lisa_password_2024@lisa_postgres_quick:5432/lisa_dev"
# Пароль захардкожен в коде!

# Нужно:
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://...")
```

---

### ФАЙЛ: `/workspaces/linux-agent/modified_updatable_agent.py`
**НАЗНАЧЕНИЕ:** Главный Linux-агент с auto-update
**СТРОК КОДА:** ~2000+

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✓ Бесконечный цикл `while True`
- ✓ Plugin система для приложений
- ✓ Auto-update механизм
- ✓ Mutex (один экземпляр)
- ~ xdotool для GUI — работает только на X11, не Wayland
- ✗ `HEARTBEAT_CONFIG` — интервал 86400 сек (24 часа)
- ✗ БД пароль `pass` захардкожен
- ✗ Нет IMAP/SMTP, SMB, Humanizer, Social graph, Event Bus
- ✗ Нет проверки is_work_time в главном цикле

**КРИТИЧЕСКАЯ ПРОБЛЕМА — credentials:**
```python
# Текущий код:
DATABASE_CONFIG = {
    "host": "localhost",
    "database": "lisa_dev",
    "user": "lisa",
    "password": "pass"  # ← в открытом виде!
}
HEARTBEAT_CONFIG = {
    "api_key": "sk-agent-heartbeat-key-2024"  # ← в коде!
}
```

---

### ФАЙЛ: `/workspaces/dropper-linux/LISA/Dropper/Dropper/dropper.py`
**НАЗНАЧЕНИЕ:** Linux dropper с инъекцией в процесс
**СТРОК КОДА:** ~120

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✓ `memfd_create` — fileless выполнение в памяти
- ✓ `self_delete()` — самоудаление
- ✓ Маскировка имени процесса
- ~ Цель: `qterminal` — но не `bash`/системный процесс
- ✗ Нет Windows-версии (только Linux)
- ✗ Нет `VirtualAllocEx + CreateRemoteThread` для Windows

---

### ФАЙЛ: `/workspaces/dropper-linux/LISA/Dropper/injection.c`
**НАЗНАЧЕНИЕ:** C-программа ptrace-инъекции
**СТРОК КОДА:** ~400

**СООТВЕТСТВИЕ ТРЕБОВАНИЯМ:**
- ✓ `ptrace(PTRACE_ATTACH)` реализован
- ✓ `mmap` с `PROT_EXEC` в целевом процессе
- ✓ `process_vm_writev` — запись payload
- ✓ Восстановление регистров после инъекции
- ~ Работает для Linux, нет аналога для Windows
- ✗ Не интегрирован с backend pipeline

---

### ФАЙЛ: `/workspaces/frontend/src/pages/agents.vue`
**НАЗНАЧЕНИЕ:** Список активных агентов

**КРИТИЧЕСКАЯ ПРОБЛЕМА:**
```javascript
// Сломанный фильтр:
agents.value = response.data.filter(a => a.status === false)
// status — это строка ('online'/'offline'), никогда не равна false
// Список всегда пустой!

// Нужно:
agents.value = response.data.filter(a => a.status === 'online')
```

---

### ФАЙЛ: `/workspaces/frontend/src/pages/addAgentRight.vue`
**НАЗНАЧЕНИЕ:** 4-шаговый мастер создания агента

**КРИТИЧЕСКИЕ ПРОБЛЕМЫ:**
```javascript
// Захардкоженные URL во всех .vue файлах:
const response = await axios.post('http://localhost:8000/api/roles', ...)
// В проде сломается

// Нужно: создать frontend/src/config.js:
export const API_BASE = import.meta.env.VITE_API_URL || 'http://localhost:8000/api'
```

---

## ЧАСТЬ 5: АНАЛИЗ КРИТИЧЕСКИХ ПРОБЛЕМ

### ПРОБЛЕМА 1: Process Injection Windows

**Найдено в `WinAgent/actions/apps.py`:**
```python
subprocess.Popen(["powershell.exe", "-Command", cmd])
# python.exe → родитель powershell.exe ← НАРУШЕНИЕ

subprocess.Popen([browser_path, url])
# python.exe → родитель chrome.exe ← НАРУШЕНИЕ
```

**Вывод:** python.exe будет родителем всех запущенных процессов. Нужно заменить:
```python
import win32com.client
def launch_via_shell(path, args=""):
    shell = win32com.client.Dispatch("Shell.Application")
    shell.ShellExecute(path, args, "", "open", 1)
    # Родитель = explorer.exe ✓
```

---

### ПРОБЛЕМА 2: Build Pipeline — заглушка

**`builds.py`:**
```python
new_build = AgentBuild(
    build_status=BuildStatus.READY,  # ← мгновенно "готово"
    binary_path="/tmp/fake_agent.exe",  # ← фейк
    binary_size=0,
    build_log="Build completed successfully (MVP stub)"
)
```
**Проблема:** Пользователь нажимает "Build" → видит "Ready" → скачивает несуществующий файл.

---

### ПРОБЛЕМА 3: SSH деплой не реализован

**`agents.py`:**
```python
config_path = f"/tmp/shared_configs/deploy_{agent_id}_{timestamp}.json"
os.makedirs("/tmp/shared_configs", exist_ok=True)
with open(config_path, 'w') as f:
    json.dump(deploy_config, f)
return {"status": "deploy_task_created", "config_path": config_path}
# ← просто пишет JSON файл на локальный диск сервера!
```
**Проблема:** Никакого SSH. Файл пишется в `/tmp/` контейнера, target host его никогда не получит.

---

### ПРОБЛЕМА 4: Нет Event Bus

- ✗ Нет таблицы `agent_events` в `models.py`
- ✗ Нет `POST /api/events` эндпоинта
- ✗ Нет `GET /api/agents/{id}/pending`
- ✗ Нет Social graph
- ✗ Нет SSE `GET /api/events/stream`

---

### ПРОБЛЕМА 5: Dropper — только Linux

- `dropper.py` и `dropper2uoate.py` — оба только для Linux
- `memfd_create` — Linux-специфично
- `ptrace` в `injection.c` — только Linux
- `agent_payload_embed.py` содержит ELF (не PE) бинарник
- Нет `VirtualAllocEx`, `CreateRemoteThread` для Windows

---

### ПРОБЛЕМА 6: Credentials в коде

| Файл | Проблема |
|------|----------|
| `backend/app/database.py` | `lisa_password_2024` в URL |
| `linux-agent/modified_updatable_agent.py` | `"password": "pass"` |
| `linux-agent/modified_updatable_agent.py` | `"api_key": "sk-agent-heartbeat-key-2024"` |
| `backend/app/api/endpoints/heartbeat.py` | `VALID_API_KEY = "sk-agent-heartbeat-key-2024"` |
| `WinAgent/client/server_api.py` | `BASE_URL = "http://localhost:8000/api"` |
| `WinAgent/sql/test/generate_and_run_sql.py` | `password="pass"` (psql) |
| `backend/app/models/models.py` | `Servers` хранит password в plain text |
| `linux-agent/.github/fixed_updatable_agent_builder.py` | Nuitka path `/home/slash/...` |

---

## ЧАСТЬ 6: ИТОГОВЫЕ ТАБЛИЦЫ

### ТАБЛИЦА 1: Соответствие UML блокам

| Блок UML | Статус | Файл | Проблема |
|----------|--------|------|----------|
| Frontend создаёт конфиг | ~ | `addAgentRight.vue` | Нет schedule/humanizer полей |
| Config Generator | ~ | `agents.py:generate` | Пишет JSON в /tmp, не бинарник |
| Windows Agent генерация | ✗ | — | Нет автоматической генерации |
| Dropper упаковка в .exe | ✗ | — | Только Linux, нет Windows .exe |
| CI/CD SSH деплой | ✗ | `agents.py:deploy` | Пишет JSON файл вместо SSH |
| Process Injection explorer | ✗ | — | Не реализован для Windows |
| Heartbeat подтверждение | ~ | `heartbeat.py` | Работает, но next_heartbeat_in=86400 |
| Frontend ALL IS FINE | ✗ | — | Страница не существует |
| Backend Config Generator | ~ | `agents.py` | Генерирует JSON, не компилирует |
| Backend Agent Manager | ~ | `agents.py` | CRUD есть, нет команд агентам |
| Backend Build Pipeline | ✗ | `builds.py` | Заглушка: status=READY сразу |
| Backend Deployment Manager | ✗ | `agents.py:deploy` | /tmp JSON вместо деплоя |
| Dropper Process Injection | ~ | `dropper.py` | Только Linux, memfd_create |
| Dropper Embedded Binary | ~ | `agent_payload_embed.py` | Есть (LFS), только ELF |
| Dropper Binary Converter | ✓ | `convert_code.py` | Работает, hex-конвертация ELF |
| Deployment Logic SSH | ✗ | — | Не реализовано |

---

### ТАБЛИЦА 2: Соответствие требованиям ТЗ

| Требование | Статус | Файл | Что нужно |
|------------|--------|------|-----------|
| Роль: бухгалтер | ✗ | — | Создать `accountant.yaml` |
| Роль: менеджер | ✗ | — | Создать `manager.yaml` |
| Роль: ИТ-инженер | ✗ | — | Создать `it_engineer.yaml` |
| Гибкое расписание | ~ | `settings.yaml` | Добавить lunch, variance, micro_breaks |
| is_work_time() подключена | ✗ | `agent.py` | Подключить к главному циклу |
| Обеды и перерывы | ✗ | — | Отсутствуют |
| Вариация прихода ±20 мин | ✗ | — | Отсутствует |
| Бесконечный цикл | ✓ | `agent.py` | — |
| try/except в цикле | ~ | `agent.py` | Не все действия обёрнуты |
| Изоляция процессов Win | ✗ | `apps.py` | subprocess.Popen → COM Shell |
| Изоляция процессов Linux | ~ | `linux-agent` | xdg-open + start_new_session |
| Офис: работа внутри файла | ~ | `apps.py` | Открывает, не работает внутри |
| Офис: корректное закрытие | ✗ | — | Нет закрытия приложений |
| Браузер: скролл и клики | ~ | `linux-agent` | xdotool есть, Win — нет |
| Браузер: список 10-20 сайтов | ✗ | `net.py` | Только 4 URL |
| Почта: IMAP чтение | ✗ | — | Отсутствует |
| Почта: SMTP отправка | ✗ | — | Отсутствует |
| Почта: вложения | ✗ | — | Отсутствует |
| SMB сетевая папка | ✗ | — | Отсутствует |
| SSH подключения | ✗ | — | Отсутствует в агенте |
| RDP подключения | ✗ | — | Отсутствует |
| Humanizer: паузы Гаусс | ✗ | — | Фиксированный sleep |
| Humanizer: суточный ритм | ✗ | — | Отсутствует |
| Humanizer: печать опечатки | ✗ | `gui.py` | Нет опечаток |
| Humanizer: мышь Безье | ✗ | — | Отсутствует |
| Humanizer: отвлечения 4% | ✗ | — | Отсутствует |
| Social graph | ✗ | — | Нет таблицы/конфига |
| Event Bus агент→агент | ✗ | — | Нет эндпоинта и таблицы |
| Build Pipeline реальный | ✗ | `builds.py` | Заглушка |
| SSH деплой реальный | ✗ | `agents.py` | /tmp JSON |
| Process Injection Windows | ✗ | — | Не реализован |
| Process Injection Linux | ~ | `dropper.py` | memfd_create есть |
| Автозапуск Windows | ~ | `agent.spec` | PyInstaller есть, schtasks нет |
| Автозапуск Linux | ~ | `linux-agent` | systemd упомянут, install.sh нет |
| Installer одним файлом Win | ~ | `agent.spec` | Spec есть, installer.exe нет |
| Installer одним файлом Linux | ✗ | — | install.sh не существует |
| Credentials в .env | ✗ | везде | Хардкод во всех компонентах |
| Dashboard реальные данные | ~ | `main.py` | Запрос к БД есть, может быть 0 |
| SSE лента событий | ✗ | — | Нет /api/events/stream |
| График активности | ✗ | frontend | Нет Chart.js компонента |
| POST /api/events | ✗ | — | Эндпоинт отсутствует |
| GET /api/agents/{id}/pending | ✗ | — | Эндпоинт отсутствует |

*Статусы: ✓ готово | ~ частично | ✗ нет | ⚠ сломано*

---

### ТАБЛИЦА 3: Приоритизированный список задач

| # | Задача | Файл | Сложность | Оценка |
|---|--------|------|-----------|--------|
| 1 | Исправить `next_heartbeat_in: 86400→300` | `heartbeat.py` | Низкая | 10 мин |
| 2 | Исправить фильтр в `agents.vue` | `agents.vue` | Низкая | 5 мин |
| 3 | Вынести credentials в .env | все файлы | Низкая | 2 часа |
| 4 | Создать роли: accountant, manager, it_engineer | `roles/*.yaml` | Низкая | 3 часа |
| 5 | Подключить `is_work_time()` к главному циклу | `agent.py` | Низкая | 1 час |
| 6 | Добавить 10-20 URL в каждую роль | `roles/*.yaml` | Низкая | 1 час |
| 7 | Централизовать API_BASE в frontend | `config.js` + все `.vue` | Низкая | 2 часа |
| 8 | Добавить обеды/перерывы в цикл | `agent.py`, `settings.yaml` | Средняя | 4 часа |
| 9 | Добавить POST /api/events + таблицу AgentEvent | `models.py` + endpoint | Средняя | 6 часов |
| 10 | Добавить GET /api/agents/{id}/pending | `agents.py` | Средняя | 4 часа |
| 11 | Заменить subprocess.Popen на COM Shell | `apps.py` | Средняя | 6 часов |
| 12 | Humanizer: Гаусс-паузы + суточный ритм | новый `humanizer.py` | Средняя | 1 день |
| 13 | Реализовать IMAP/SMTP | новый `actions/mail.py` | Средняя | 1 день |
| 14 | Добавить SSE `/api/events/stream` | новый `events.py` | Средняя | 1 день |
| 15 | Реальный SSH деплой через paramiko | `agents.py:deploy` | Высокая | 1 день |
| 16 | Реальный Build Pipeline (PyInstaller async) | `builds.py` | Высокая | 2 дня |
| 17 | Windows dropper через Task Scheduler | новый файл | Средняя | 1 день |
| 18 | install.sh для Linux с systemd | новый файл | Средняя | 4 часа |
| 19 | Social graph + Event Bus между агентами | `models.py` + endpoints | Высокая | 3 дня |
| 20 | График активности Chart.js на фронте | новый компонент | Средняя | 1 день |

---

## ШАГ 3: АРХИТЕКТУРНЫЙ АНАЛИЗ

### ВОПРОС 1: ПЕРЕПИСАТЬ ИЛИ ДОРАБОТАТЬ?

| Компонент | Решение | Обоснование |
|-----------|---------|-------------|
| **backend** | **Доработать** | Правильный стек, правильная структура. 5-6 точечных исправлений |
| **WinAgent** | **Доработать** | Главный цикл правильный. Переписать apps.py + добавить humanizer |
| **linux-agent** | **Рефакторинг** | Разбить 2000-строчный файл на модули. Логика правильная |
| **frontend** | **Доработать** | 80% работает. Добавить Dashboard + исправить 2 бага |
| **dropper** | **Упростить** | injection.c оставить. dropper.py упростить до Task Scheduler |

---

### ВОПРОС 2: ТЕХНОЛОГИЧЕСКИЙ СТЕК

#### WinAgent на Python
**Вердикт:** Python — правильный выбор. Цель — тестирование SIEM, не обход AV. PyInstaller `--onefile` достаточно. Переход на Go потребует 2-3 месяца переучивания.

| Стек | Оценка |
|------|--------|
| Python (текущий) | Правильный выбор |
| Go | Хорош для маскировки, плохо для GUI на Windows |
| C# | Идеален для Windows COM, требует .NET |
| PowerShell | Нативен, легко детектируется SIEM |

#### Linux Agent на Python
**Вердикт:** Python правильный выбор. Проблема не в языке — в структуре кода (2000 строк в одном файле).

#### Backend FastAPI + PostgreSQL
**Вердикт:** Правильный выбор. SQLite был бы достаточен для 3-4 агентов, но PostgreSQL уже настроен — менять нецелесообразно.

#### Frontend Vue + Vuetify
**Вердикт:** Оставить Vue. Проблемы точечные (URL, фильтр, отсутствие Dashboard), не архитектурные.

#### Dropper на Python
**Вердикт:** Python — неправильный язык для process injection. Но для данной задачи достаточно Task Scheduler как посредника (см. Вопрос 4).

---

### ВОПРОС 3: ОПТИМАЛЬНАЯ СТРУКТУРА РЕПОЗИТОРИЯ

```
/workspaces/
├── lisa-core/                    # Shared библиотека (НОВАЯ)
│   ├── lisa_core/
│   │   ├── heartbeat.py          # Общий клиент heartbeat
│   │   ├── config.py             # Загрузка и валидация конфига
│   │   ├── scheduler.py          # is_work_time(), обеды, перерывы
│   │   ├── humanizer.py          # Гаусс-паузы, суточный ритм
│   │   └── transport.py          # HTTP клиент к backend API
│   └── pyproject.toml
│
├── agents/
│   ├── common/
│   │   ├── base_agent.py         # Базовый класс: цикл, веса, ошибки
│   │   └── base_actions.py       # Абстрактные действия
│   │
│   ├── windows/                  # Текущий WinAgent/
│   │   ├── agent.py
│   │   ├── actions/
│   │   │   ├── apps.py           # COM Shell, os.startfile
│   │   │   ├── mail.py           # IMAP/SMTP
│   │   │   └── smb.py
│   │   ├── roles/
│   │   │   ├── accountant.yaml
│   │   │   ├── manager.yaml
│   │   │   └── it_engineer.yaml
│   │   └── installer/
│   │       ├── install.py        # schtasks автозапуск
│   │       └── agent.spec
│   │
│   └── linux/                    # Текущий linux-agent/
│       ├── agent.py
│       ├── plugins/
│       ├── roles/
│       └── installer/
│           └── install.sh
│
├── backend/                      # Текущий
│   └── app/
│       └── api/endpoints/
│           └── events.py         # НОВЫЙ: SSE + Event Bus
│
├── frontend/                     # Текущий
│
├── dropper/
│   ├── linux/
│   │   ├── dropper.py            # Упрощённый
│   │   └── injection.c           # Существующий
│   └── windows/                  # НОВЫЙ (опционально)
│
└── docker-compose.yml
```

**Ключевое изменение:** `lisa-core` как pip-пакет:
```bash
pip install -e /workspaces/lisa-core/
```
Heartbeat, humanizer, scheduler — в одном месте, не дублируются.

---

### ВОПРОС 4: ПРАВИЛЬНЫЙ ЛИ ПОДХОД К PROCESS ISOLATION?

#### Реальная оценка VirtualAllocEx + CreateRemoteThread (текущий план)

Проблемы:
1. Windows Defender блокирует с 2021 года — нужно отключать защиту
2. Требует `SeDebugPrivilege` — администратор обязателен
3. При сбое инъекции `explorer.exe` крашится
4. Правильная реализация: 3-6 недель у опытного C-разработчика

#### Рекомендуемый подход: Вариант А + Вариант Б

**Вариант А — Shell.Application COM (для большинства приложений):**
```python
import win32com.client
shell = win32com.client.Dispatch("Shell.Application")
shell.ShellExecute(app_path, args, "", "open", 1)
# Родитель = explorer.exe ✓
# Сложность: 2 часа
# Реалистичность для SIEM: высокая
```

**Вариант Б — Task Scheduler (для "специальных" запусков):**
```python
subprocess.run([
    'schtasks', '/Create', '/F',
    '/TN', 'ChromeUpdate',
    '/TR', f'"{chrome_path}" {url}',
    '/SC', 'ONCE',
    '/ST', (datetime.now() + timedelta(seconds=5)).strftime('%H:%M'),
    '/RU', os.getenv('USERNAME')
])
# Родитель = svchost.exe ✓
# Реалистичность: очень высокая
```

**Вывод:** Process injection с VirtualAllocEx — решение другой задачи (обход EDR). Для тестирования SIEM на паттерны поведения Shell.Application COM даёт 95% результата за 5% усилий.

---

### ВОПРОС 5: ПРАВИЛЬНЫЙ ЛИ ПОДХОД К EVENT BUS?

**Математика:** 3-4 агента, задержки реакции 5-40 минут, 10-30 событий/день.

**Вердикт: PostgreSQL polling — оптимален.**

```sql
CREATE TABLE agent_commands (
    id SERIAL PRIMARY KEY,
    agent_id VARCHAR NOT NULL,
    command_type VARCHAR NOT NULL,
    payload JSONB NOT NULL,
    scheduled_at TIMESTAMP NOT NULL,
    executed_at TIMESTAMP,
    status VARCHAR DEFAULT 'pending'
);
```

Агент делает `GET /api/agents/{id}/pending` каждые 60 секунд.

- Задержка 60 сек при минимальном интервале реакции 5 мин = погрешность 0.3%
- Redis — дополнительная инфраструктура без выигрыша
- WebSocket — проблемы за NAT при перезапуске агента
- Long polling — усложняет backend без существенного выигрыша

---

### ВОПРОС 6: ПРАВИЛЬНЫЙ ЛИ ПОДХОД К HUMANIZER?

**Рекомендуемая реализация (~80 строк, ~4 часа разработки):**

```python
# agents/common/humanizer.py
import random, time
from datetime import datetime

class Humanizer:
    def sleep(self, base_seconds: float) -> None:
        """Гаусс-распределение + суточный ритм + редкие отвлечения"""
        t = random.gauss(base_seconds, base_seconds * 0.2)
        t *= self._time_of_day_factor()
        if random.random() < 0.04:  # 4% — отвлечение
            t += random.uniform(300, 1500)
        time.sleep(max(1.0, t))

    def _time_of_day_factor(self) -> float:
        hour = datetime.now().hour
        if 10 <= hour <= 12: return 0.85   # бодрое утро
        if 14 <= hour <= 15: return 1.35   # послеобеденный спад
        return 1.0

    def type_text(self, text: str) -> None:
        for char in text:
            if random.random() < 0.02:  # 2% опечаток
                wrong = random.choice('qwertyuiopasdfghjklzxcvbnm')
                self._press_key(wrong)
                time.sleep(random.uniform(0.1, 0.3))
                self._press_key('\b')
            self._press_key(char)
            time.sleep(random.gauss(0.08, 0.03))

    def move_mouse_bezier(self, x2: int, y2: int) -> None:
        import pyautogui
        x1, y1 = pyautogui.position()
        cx1 = x1 + random.randint(-100, 100)
        cy1 = y1 + random.randint(-50, 50)
        cx2 = x2 + random.randint(-100, 100)
        cy2 = y2 + random.randint(-50, 50)
        steps = random.randint(20, 50)
        for i in range(steps + 1):
            t = i / steps
            x = (1-t)**3*x1 + 3*(1-t)**2*t*cx1 + 3*(1-t)*t**2*cx2 + t**3*x2
            y = (1-t)**3*y1 + 3*(1-t)**2*t*cy1 + 3*(1-t)*t**2*cy2 + t**3*y2
            pyautogui.moveTo(int(x) + random.randint(-2,2),
                           int(y) + random.randint(-2,2), _pause=False)
            time.sleep(0.01)
```

**По инструментам:**
- pyautogui — достаточно для данной задачи
- win32api напрямую — надёжнее, без FAILSAFE проблем
- Playwright — хуже: создаёт Chromium с --remote-debugging-port (подозрительно для SIEM)
- `os.startfile(url)` + реальный Chrome — реалистичнее Playwright для SIEM тестирования
- xdotool — хорошо на X11 (96% success, test_report.md), нужен `XDG_SESSION_TYPE=x11`

---

### ВОПРОС 7: ПРАВИЛЬНЫЙ ЛИ ПОДХОД К ДЕПЛОЮ?

**Рекомендация: Вариант А + Вариант Г**

- **install.sh / install.py** — надёжный ручной деплой для тестовой среды
- **SSH через paramiko** — автоматизированный деплой из UI

**Правильная реализация SSH деплоя:**
```python
async def deploy_agent(agent_id: str, server: Server, db: Session):
    build = db.query(AgentBuild).filter_by(
        agent_id=agent_id, build_status='ready').first()
    if not build:
        raise HTTPException(400, "No ready build found")

    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        client.connect(server.ip, username=server.login,
                      password=decrypt(server.password), timeout=30)
        sftp = client.open_sftp()
        sftp.put(build.binary_path, '/tmp/lisa_agent')
        sftp.close()
        stdin, stdout, stderr = client.exec_command(
            'chmod +x /tmp/lisa_agent && /tmp/lisa_agent --install')
        exit_code = stdout.channel.recv_exit_status()
        if exit_code != 0:
            raise Exception(stderr.read().decode())
        return {"status": "deployed"}
    finally:
        client.close()
```

---

## ШАГ 4: ПРОБЛЕМЫ В ТЕКУЩЕМ КОДЕ

### Неправильные решения (требуют исправления)

| Проблема | Файл | Время |
|----------|------|-------|
| `next_heartbeat_in: 86400` | `heartbeat.py` | 10 мин |
| `subprocess.Popen` для приложений | `apps.py` | 2 часа |
| `BuildStatus.READY` сразу | `builds.py` | 1 день |
| JSON в `/tmp` вместо SSH | `agents.py` | 4-6 часов |
| Credentials захардкожены | везде | 4 часа |
| Фильтр `status === false` | `agents.vue` | 5 мин |
| Heartbeat 86400 в linux-agent | `modified_updatable_agent.py` | 5 мин |

### Хорошие решения которые стоит сохранить

- **Plugin система** в linux-agent — JSON плагины архитектурно правильны
- **Auto-update механизм** — уникальная фича, сохранить
- **Singleton + heartbeat thread + main loop с весами** — правильный паттерн
- **PyInstaller spec файл** — правильно настроен
- **injection.c** — C реализация ptrace написана корректно
- **4-шаговый мастер** в addAgentRight.vue — UX правильный

### Излишняя сложность

1. **Два агента без общего кода** — WinAgent и linux-agent дублируют heartbeat, singleton, main loop
2. **Process injection для данной задачи** — Task Scheduler даёт 95% результата за 5% усилий
3. **Git LFS для agent_payload_embed.py** — артефакт сборки не должен быть в Git
4. **SQL скрипты с хардкод MAC** — должны быть миграции Alembic
5. **Два dropper файла** (dropper.py и dropper2uoate.py) — один должен быть удалён

### Критически недостающие вещи

| Что | Критичность | Время |
|-----|-------------|-------|
| IMAP/SMTP в агентах | Высокая | 1 день |
| `GET /api/agents/{id}/pending` | Высокая | 4 часа |
| `POST /api/events` + Event Bus | Высокая | 1 день |
| Роли: accountant, manager, it_engineer | Высокая | 3 часа |
| `is_work_time()` в главном цикле | Высокая | 2 часа |
| SSE `GET /api/events/stream` | Средняя | 6 часов |
| Dashboard с реальными данными | Средняя | 1 день |
| install.sh + systemd | Средняя | 4 часа |
| install.py + schtasks | Средняя | 4 часа |
| Humanizer модуль | Средняя | 4 часа |
| Шифрование паролей серверов в БД | Высокая | 3 часа |

---

## ШАГ 5: ОПТИМАЛЬНЫЙ ПЛАН РЕАЛИЗАЦИИ (5 месяцев)

### ЭТАП 1 (Месяц 1): Работающая основа

**Цель:** Linux агент работает, виден на дашборде, три роли настроены.

**Неделя 1: Критические фиксы**
```
- heartbeat.py: next_heartbeat_in 86400 → 300
- agents.vue: фильтр false → 'online'
- database.py, modified_updatable_agent.py: credentials → .env
- frontend/src/config.js: централизованный API_BASE
- is_work_time() подключить в linux-agent
```

**Неделя 2: Три роли**
```yaml
# accountant.yaml:
role: accountant
schedule:
  work_start: "09:00"
  work_end: "18:00"
  work_days: [1,2,3,4,5]
lunch:
  start: "13:00"
  duration_range: [30, 60]
arrival_variance_minutes: 20
micro_breaks:
  per_hour: [1, 2]
  duration_range: [5, 15]
urls:
  - "https://mail.google.com"
  - "https://docs.google.com"
  # ... 13 ещё
apps: [libreoffice, thunderbird, evince]
weights:
  open_file: 30
  browser: 25
  mail: 30
  idle: 15
```

**Неделя 3: Event Bus (минимум)**
```python
# models.py — добавить:
class AgentCommand(Base):
    id, agent_id, command_type, payload (JSON),
    scheduled_at, executed_at, status

# endpoints — добавить:
GET /api/agents/{id}/pending
POST /api/events
```

**Неделя 4: Humanizer**
```
Создать agents/common/humanizer.py (~80 строк)
Подключить в linux-agent и WinAgent
```

---

### ЭТАП 2 (Месяц 2): Windows агент + почта

**Неделя 5-6: WinAgent process isolation**
```python
# apps.py — заменить subprocess.Popen:
def launch_via_shell(path, args=""):
    shell = win32com.client.Dispatch("Shell.Application")
    shell.ShellExecute(path, args, "", "open", 1)
```
Проверка: Process Monitor → chrome.exe родитель = explorer.exe

**Неделя 6: IMAP/SMTP**
```python
# agents/common/mail.py:
class MailClient:
    def read_inbox(self, count=5) -> List[Message]: ...
    def send(self, to, subject, body, attach=None): ...
```

**Неделя 7-8: Real Build Pipeline**
```python
# builds.py — убрать заглушку:
async def build_agent_async(build_id, template_id, db):
    build.build_status = BuildStatus.BUILDING
    db.commit()
    result = subprocess.run(
        ['python', '-m', 'PyInstaller', '--onefile', 'agent.py'],
        capture_output=True, timeout=300)
    build.build_status = (BuildStatus.READY if result.returncode == 0
                         else BuildStatus.FAILED)
    db.commit()

background_tasks.add_task(build_agent_async, new_build.id, ...)
```

---

### ЭТАП 3 (Месяц 3): Взаимодействие агентов

**Неделя 9-10: Social graph и сценарии**

Три конкретных сценария:
1. Accountant сохраняет Excel → Manager читает письмо через 10-30 мин
2. IT_Engineer перезапускает сервис → Accountant видит уведомление
3. Manager создаёт задачу → Accountant открывает файл задачи

**Неделя 11: SMB**
```python
# agents/windows/actions/smb.py
def connect_share(server, share, user, password):
    shell = win32com.client.Dispatch("Shell.Application")
    shell.ShellExecute('net', f'use Z: \\\\{server}\\{share}', '', 'open', 0)
```

**Неделя 12: SSH действия для it_engineer**
```python
# agents/linux/actions/ssh_action.py
def execute_remote(host, user, password, commands):
    client = paramiko.SSHClient()
    client.connect(host, username=user, password=password)
    for cmd in commands:
        humanizer.sleep(2)
        client.exec_command(cmd)
```

---

### ЭТАП 4 (Месяц 4): Деплой + автозапуск

**install.sh:**
```bash
#!/bin/bash
set -e
read -p "IP сервера LISA Backend: " SERVER_IP
read -p "Роль агента (accountant/manager/it_engineer): " ROLE

mkdir -p ~/.config/lisa
cat > ~/.config/lisa/.env << EOF
SERVER_URL=http://${SERVER_IP}:8000
ROLE=${ROLE}
AGENT_API_KEY=$(python3 -c "import secrets; print(secrets.token_hex(32))")
EOF

cat > ~/.config/systemd/user/lisa-agent.service << EOF
[Unit]
Description=LISA User Activity Agent
After=graphical-session.target
[Service]
ExecStart=/usr/bin/python3 $(pwd)/agent.py
EnvironmentFile=%h/.config/lisa/.env
Restart=always
RestartSec=30
[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now lisa-agent
```

**install.py (Windows):**
```python
def install():
    server_ip = input("IP сервера: ")
    role = input("Роль: ")
    # Создать .env в %APPDATA%/LISA/
    # Зарегистрировать в Task Scheduler ONLOGON
    subprocess.run([
        'schtasks', '/Create', '/F',
        '/TN', 'LISAUserAgent',
        '/TR', f'"{agent_exe_path}"',
        '/SC', 'ONLOGON', '/DELAY', '0:30',
        '/RU', os.getenv('USERNAME')
    ])
```

**SSE для дашборда:**
```python
# backend/app/api/endpoints/events.py
@router.get("/events/stream")
async def event_stream():
    queue = asyncio.Queue()
    _subscribers.append(queue)
    async def generate():
        try:
            while True:
                event = await queue.get()
                yield f"data: {json.dumps(event)}\n\n"
        finally:
            _subscribers.remove(queue)
    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

### ЭТАП 5 (Месяц 5): Эксперимент + измерения

**Тестовая среда:**
- VM1: Ubuntu 22.04, роль accountant
- VM2: Windows 10, роль manager
- VM3: Ubuntu 22.04, роль it_engineer
- Wazuh/Splunk Free подключён ко всем ВМ
- Запуск на 48 часов без изменений

**Метрики:**
1. Wazuh алерты в час — цель < 1 ложного срабатывания/агент/день
2. Process tree — % запусков где родитель = explorer.exe (Windows)
3. Временной паттерн — совпадение с профилем рабочего дня
4. Email трафик — реалистичное распределение по времени
5. Количество уникальных URL в day/week ratio
6. Корреляция действий между агентами

---

## ШАГ 6: ИТОГОВЫЕ РЕКОМЕНДАЦИИ

### ТОП-5 ИЗМЕНЕНИЙ: МАКСИМУМ РЕЗУЛЬТАТА ЗА МИНИМУМ ВРЕМЕНИ

| # | Изменение | Файл | Время | Результат |
|---|-----------|------|-------|-----------|
| 1 | `next_heartbeat_in: 86400→300` | `heartbeat.py` | 10 мин | Дашборд показывает online |
| 2 | Фильтр `false→'online'` | `agents.vue` | 5 мин | Список агентов не пустой |
| 3 | Три роли YAML с 15+ URL | `roles/*.yaml` | 3 часа | Ключевое требование ТЗ |
| 4 | `is_work_time()` + обеды в цикл | `agent.py` | 2 часа | Реалистичный рабочий день |
| 5 | COM Shell вместо subprocess.Popen | `apps.py` | 2 часа | Правильный process tree |

**Итого: ~8 часов. Результат: система демонстрируема.**

### ТОП-5 ВЕЩЕЙ КОТОРЫЕ НЕ СТОИТ ДЕЛАТЬ

1. **VirtualAllocEx + CreateRemoteThread** — 3-6 недель, заблокирует Defender, не нужно для SIEM тестирования
2. **Переход с Python на Go/Rust** — 2-3 месяца переучивания без выигрыша для задачи
3. **Redis как Event Bus** — PostgreSQL polling полностью достаточен для 3-4 агентов
4. **Playwright для браузера** — `os.startfile(url)` реалистичнее: реальный Chrome с реальным трафиком
5. **Windows dropper на C с нуля** — Task Scheduler даёт правильный process tree за 20 строк Python

### ТЕХНОЛОГИЧЕСКИЕ ЗАМЕНЫ

| Что | На что | Почему |
|-----|--------|--------|
| `subprocess.Popen` для приложений | `win32com.client Shell.Application` | Родитель = explorer.exe |
| Хардкод credentials | `python-dotenv` + `.env` | Безопасность |
| Plain text пароли в БД | `cryptography.fernet.Fernet` | Безопасность |
| Фиксированный `time.sleep()` | `random.gauss(mu, sigma)` | Реалистичность |
| `BuildStatus.READY` сразу | `asyncio.create_task()` + PyInstaller | Корректность |
| JSON в `/tmp` для деплоя | `paramiko.SSHClient` | Реальный деплой |
| `http://localhost:8000` в Vue | `import.meta.env.VITE_API_URL` | Configurable |
| 2 dropper файла | 1 актуальный + удалить | Чистота кода |
| `agent_payload_embed.py` в Git | CI/CD артефакты | Правильное место |

### АРХИТЕКТУРНЫЕ ПАТТЕРНЫ

**Использовать:**
- **Strategy Pattern** — каждое действие (`OpenBrowser`, `SendMail`) — отдельный класс
- **Template Method** — `BaseAgent` с абстрактными `_launch_app()`, `_browse_url()`
- **Factory Pattern** — создание действий из YAML роли через `ActionFactory.create()`
- **Observer/Event Bus** — PostgreSQL polling (простота важнее latency)

**Антипаттерны в текущем коде:**
- **God Object** — `modified_updatable_agent.py` 2000+ строк
- **Hardcoded Magic Numbers** — `86400`, `sk-agent-heartbeat-key-2024`
- **Stub Implementation** — `BuildStatus.READY` сразу, `/tmp/fake_agent.exe`
- **Shotgun Surgery** — `localhost:8000` в 6 разных .vue файлах
- **Duplication** — heartbeat логика в двух агентах независимо

---

## ФИНАЛЬНАЯ ОЦЕНКА

### Процент готовности

| Метрика | Значение |
|---------|----------|
| По UML блокам | **25%** (4 из 16 блоков работают) |
| По требованиям ТЗ | **18%** (8 из 44 требований выполнены) |
| Критических блокеров | **6** |
| Задач < 1 часа | **4** |
| Задач ~1 день | **10** |
| Задач 3+ дней | **6** |

### Что показать на демо ПРЯМО СЕЙЧАС

- ✓ Создание агента через 4-шаговый мастер
- ✓ Linux агент реально работает (xdotool активности)
- ✓ Linux dropper с memfd_create инъекцией
- ✓ CRUD ролей и шаблонов через API
- ✓ WebSocket `/ws/agents/{id}` (ping работает)
- ✓ Страница деплоя — выбор сервера

### Что сломается на демо

- ✗ Кнопка Build — статус "Ready", бинарника нет
- ✗ Кнопка Deploy — JSON в /tmp, до target host не дойдёт
- ✗ Список агентов — фильтр `status === false` → пустой список
- ✗ Dashboard stats — нули
- ✗ Windows агент — все процессы запускает как дочерние python.exe

### Минимум для защиты диплома

1. Исправить heartbeat (10 мин) — иначе все агенты "offline"
2. Исправить фильтр в agents.vue (5 мин) — иначе пустой список
3. Реальный SSH деплой через paramiko (1 день)
4. Создать 3 роли YAML (3 часа)
5. Linux install.sh с systemd (4 часа)

### Научная новизна

1. **Плагинная архитектура агентов** — JSON плагины без перекомпиляции
2. **Auto-update через БД** — обновление шаблона без перезапуска
3. **Fileless инъекция через memfd_create** — агент без файлов на диске
4. **Social graph для координации** — Event Bus между агентами с настраиваемой задержкой
5. **Суточный humanizer** — статистически реалистичный временной профиль активности

### Финальный прогноз

При реализации этапов 1-3:
- Wazuh Sigma rules "unusual process tree" — **не сработают** (Shell.Application COM)
- Wazuh rules "after-hours activity" — **не сработают** (is_work_time)
- ML UEBA (Splunk, Sentinel) — **адаптируется** к baseline за 1-2 дня
- Статистический анализ временных рядов — **реалистично** при Гаусс-распределении
- `python.exe` в процессах — **заметен** как нестандартное приложение
- Подключение к порту 8000 — **заметно** как нестандартный трафик

**Итог:** Для задачи "генерация реалистичного UEBA-датасета для тестирования SIEM" — система будет работать при правильной реализации. Для задачи "обойти production EDR" — не предназначена.

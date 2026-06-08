# Инструкция по развёртыванию NiFi Admin

Пошаговое руководство: что нужно, как собрать, перенести и запустить панель в production (в том числе без интернета).

---

## 1. Что нужно

### На машине сборки (с интернетом)

| Компонент | Версия | Зачем |
|-----------|--------|-------|
| Python | 3.10+ | Backend, установка зависимостей в `vendor/python` |
| Node.js + npm | 18+ | Сборка frontend (`npm run build` → `frontend/dist`) |
| PowerShell или bash | — | Скрипты в `scripts/` |

### На целевом сервере (production / air-gap)

| Компонент | Версия | Зачем |
|-----------|--------|-------|
| Python | 3.10+ | **Только Python** — Node.js не нужен |
| Сеть до NiFi | HTTP | Доступ к `http://<nifi-host>:<port>/nifi-api` |
| Порт | 8000 (по умолчанию) | Панель: UI + API на одном порту |

### Что входит в архив проекта

```
nifi-admin/
├── vendor/python/      # Python-зависимости (offline)
├── frontend/dist/      # Собранный UI (обязательно для production)
├── backend/            # FastAPI-приложение
│   ├── app/
│   ├── data/           # servers.json, пороги алертов
│   └── .env.example
├── scripts/            # install / build скрипты
├── README.md
└── DEPLOY.md           # этот файл
```

> **Не нужны на сервере:** `frontend/node_modules`, исходники `frontend/src` (если уже есть `dist`).

---

## 2. Сборка архива (машина с интернетом)

### Windows

```powershell
cd d:\Projects\nifi-admin

# 1. Python-зависимости → vendor/python
.\scripts\install-deps.ps1

# 2. npm-зависимости frontend
.\scripts\install-frontend.ps1

# 3. Production-сборка UI → frontend/dist
.\scripts\build-prod.ps1

# 4. Конфиг (скопируйте и отредактируйте перед production)
copy backend\.env.example backend\.env
```

### Linux / macOS

```bash
cd /path/to/nifi-admin
chmod +x scripts/*.sh
./scripts/install-deps.sh
./scripts/install-frontend.sh
./scripts/build-prod.sh
cp backend/.env.example backend/.env
```

После сборки упакуйте папку `nifi-admin` в ZIP (без `frontend/node_modules`).

---

## 3. Перенос на целевой сервер

1. Скопируйте архив `nifi-admin-v1.0.0.zip` на сервер (USB, SCP, общая папка).
2. Распакуйте в рабочую директорию, например `C:\Apps\nifi-admin` или `/opt/nifi-admin`.
3. Убедитесь, что присутствуют:
   - `vendor/python/`
   - `frontend/dist/` (папка с `index.html` и `assets/`)
   - `backend/app/`

---

## 4. Настройка перед запуском

### 4.1. Файл `backend/.env`

Скопируйте из примера:

```powershell
copy backend\.env.example backend\.env
```

Минимальная конфигурация для production:

```env
APP_HOST=0.0.0.0
APP_PORT=8000
CORS_ORIGINS=http://localhost:8000,http://192.168.0.10:8000

# ОБЯЗАТЕЛЬНО в production:
PANEL_TOKEN=ваш-длинный-секретный-токен

# Опционально — отдельный ключ шифрования паролей NiFi в servers.json
CREDENTIALS_KEY=другой-секретный-ключ
```

| Переменная | Описание |
|------------|----------|
| `PANEL_TOKEN` | Токен входа в панель. Передаётся в заголовке `X-Panel-Token`. Без него API открыт всем. |
| `CREDENTIALS_KEY` | Ключ шифрования паролей NiFi в `servers.json`. Если пусто — используется `PANEL_TOKEN`. |
| `CORS_ORIGINS` | Разрешённые адреса браузера (через запятую). |

### 4.2. Подключение серверов NiFi

**Через UI (рекомендуется):**

1. Откройте `http://<ip-сервера>:8000`
2. При запросе введите `PANEL_TOKEN`
3. Вкладка **Серверы** → **Добавить сервер**
4. Заполните:
   - **Хост** — IP или hostname NiFi
   - **Порт** — например `8888`
   - **REST API path** — `/nifi-api`
   - **Логин / пароль** — учётная запись NiFi (если включена аутентификация)
5. Нажмите **Проверить**, затем **Сохранить**
6. **Открыть** — переход к Summary и Flow

**Вручную** — файл `backend/data/servers.json`:

```json
{
  "servers": [
    {
      "id": "a1b2c3d4-0001",
      "name": "NiFi Prod",
      "host": "192.168.0.52",
      "port": 8888,
      "base_path": "/nifi-api",
      "username": "admin",
      "password": ""
    }
  ],
  "active_server_id": "a1b2c3d4-0001"
}
```

> Пароли после сохранения через UI шифруются (`enc:...`) при заданном `PANEL_TOKEN`.

---

## 5. Запуск

### Windows (интерактивно)

```powershell
cd C:\Apps\nifi-admin

$env:PYTHONPATH = "vendor\python;backend"
$env:PANEL_TOKEN = "ваш-токен"

python -m uvicorn app.main:app --app-dir backend --host 0.0.0.0 --port 8000
```

### Linux

```bash
cd /opt/nifi-admin
export PYTHONPATH="vendor/python:backend"
export PANEL_TOKEN="ваш-токен"

python3 -m uvicorn app.main:app --app-dir backend --host 0.0.0.0 --port 8000
```

### Проверка

| URL | Ожидание |
|-----|----------|
| `http://<ip>:8000` | Страница входа / панель NiFi Admin |
| `http://<ip>:8000/api/health` | `{"status":"ok","version":"1.0.0"}` |
| `http://<ip>:8000/docs` | Swagger API |

---

## 6. Автозапуск (опционально)

### Windows — Планировщик задач или NSSM

Создайте `start-nifi-admin.ps1`:

```powershell
$env:PYTHONPATH = "C:\Apps\nifi-admin\vendor\python;C:\Apps\nifi-admin\backend"
$env:PANEL_TOKEN = "ваш-токен"
Set-Location C:\Apps\nifi-admin
python -m uvicorn app.main:app --app-dir backend --host 0.0.0.0 --port 8000
```

### Linux — systemd

Файл `/etc/systemd/system/nifi-admin.service`:

```ini
[Unit]
Description=NiFi Admin Panel
After=network.target

[Service]
Type=simple
User=nifi
WorkingDirectory=/opt/nifi-admin
Environment=PYTHONPATH=/opt/nifi-admin/vendor/python:/opt/nifi-admin/backend
Environment=PANEL_TOKEN=ваш-токен
ExecStart=/usr/bin/python3 -m uvicorn app.main:app --app-dir backend --host 0.0.0.0 --port 8000
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable nifi-admin
sudo systemctl start nifi-admin
```

---

## 7. Требования к NiFi

- Apache NiFi **1.25+** или **2.x** (проверено на 2.7.2)
- REST API доступен по HTTP: `http://<host>:<port>/nifi-api`
- Учётная запись с правами на чтение/управление (для Start/Stop Processors, empty queue и т.д.)
- Панель и NiFi должны видеть друг друга в сети

---

## 8. Безопасность

1. **Всегда** задавайте `PANEL_TOKEN` в production.
2. Не открывайте порт 8000 в интернет без reverse proxy и HTTPS.
3. Ограничьте `CORS_ORIGINS` конкретными адресами.
4. Файл `servers.json` и экспорт серверов содержат учётные данные — ограничьте доступ к `backend/data/`.
5. NiFi подключается по HTTP в доверенной сети (TLS verification отключён по дизайну offline HTTP).

---

## 9. Устранение неполадок

| Проблема | Решение |
|----------|---------|
| Пустая страница / 404 | Убедитесь, что есть `frontend/dist/index.html` |
| `ModuleNotFoundError: fastapi` | Проверьте `PYTHONPATH`: `vendor/python;backend` |
| 401 при работе с API | Введите `PANEL_TOKEN` в диалоге авторизации |
| Сервер NiFi «Не в сети» | Проверьте host/port, firewall, доступность `/nifi-api/flow/about` |
| Пароли не расшифровываются | Тот же `PANEL_TOKEN`/`CREDENTIALS_KEY`, что при сохранении |
| Порт занят | Смените `--port` или освободите 8000 |

---

## 10. Обновление версии

1. На машине сборки: `git pull` / новый архив → `install-deps` → `build-prod`
2. Остановите сервис на целевом сервере
3. Замените `vendor/python`, `frontend/dist`, `backend/app` (сохраните `backend/data/` и `.env`)
4. Запустите снова

---

**Версия:** 1.0.0  
**Порт по умолчанию:** 8000  
**Документация API:** `http://<host>:8000/docs`

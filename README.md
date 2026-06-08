# NiFi Admin

Веб-панель администрирования Apache NiFi (1.25+, включая 2.7+).

**v1.0.0** — мульти-серверная панель с терминологией Apache NiFi, offline-ready, только HTTP.

## Стек

| Слой | Технологии |
|------|-----------|
| Backend | Python 3.10+, FastAPI, httpx, pydantic |
| Frontend | Vite, Chart.js, vanilla JS (без CDN) |
| API | Apache NiFi REST API (`/nifi-api`) |

## Функционал

- **Серверы** — CRUD, импорт/экспорт, автообновление статусов
- **Обзор** — сводка метрик по всем серверам
- **Summary** — heap, CPU, FlowFiles, кластер, графики, пороги алертов
- **Flow** — Process Groups, Processors, Connections, Start/Stop
- **Bulletin Board** — предупреждения контроллера и компонентов
- **Cluster** — топология нод
- **Controller Services** — Enable/Disable
- **Parameter Contexts** — параметры и привязки
- **NiFi Registry** — buckets, flows, versions, deploy
- **Provenance** — поиск по FlowFile UUID / Component ID
- **Search** — Processors и Process Groups
- Светлая / тёмная тема, автообновление

## Безопасность (production)

1. **Обязательно** задайте `PANEL_TOKEN` в `backend/.env` — иначе API открыт без авторизации.
2. Пароли NiFi в `servers.json` шифруются при сохранении (ключ: `CREDENTIALS_KEY` или `PANEL_TOKEN`).
3. Экспорт серверов содержит учётные данные — храните файл в защищённом месте.
4. Ограничьте `CORS_ORIGINS` и не публикуйте панель в интернет без reverse proxy.
5. NiFi подключается по HTTP (`verify=False`) — используйте в доверенной сети.

## Быстрый старт (разработка)

### Windows

```powershell
.\scripts\install-deps.ps1
.\scripts\install-frontend.ps1
copy backend\.env.example backend\.env

$env:PYTHONPATH = "vendor\python;backend"
python -m uvicorn app.main:app --app-dir backend --host 0.0.0.0 --port 8000 --reload

cd frontend; npm run dev
```

- Dev UI: http://localhost:5173 (proxy `/api` → :8000)
- API: http://localhost:8000/docs

### Ubuntu

```bash
chmod +x scripts/*.sh
./scripts/install-deps.sh && ./scripts/install-frontend.sh
export PYTHONPATH="vendor/python:backend"
python3 -m uvicorn app.main:app --app-dir backend --host 0.0.0.0 --port 8000 --reload
cd frontend && npm run dev
```

## Продакшен (без интернета)

### Сборка

```powershell
.\scripts\install-deps.ps1
.\scripts\install-frontend.ps1
.\scripts\build-prod.ps1
```

### Запуск на целевом сервере

```powershell
$env:PYTHONPATH = "vendor\python;backend"
$env:PANEL_TOKEN = "your-secure-token"
python -m uvicorn app.main:app --app-dir backend --host 0.0.0.0 --port 8000
```

Откройте: http://&lt;server-ip&gt;:8000

### Конфигурация серверов

Через UI → **Серверы** → добавить сервер (host, port, `/nifi-api`, credentials).

Файл: `backend/data/servers.json`:

```json
{
  "servers": [
    {
      "id": "uuid",
      "name": "NiFi Prod",
      "host": "192.168.0.52",
      "port": 8888,
      "base_path": "/nifi-api",
      "username": "admin",
      "password": "enc:..."
    }
  ],
  "active_server_id": "uuid"
}
```

## Переменные окружения

| Переменная | Описание |
|------------|----------|
| `PANEL_TOKEN` | Токен доступа к панели (заголовок `X-Panel-Token`) |
| `CREDENTIALS_KEY` | Ключ шифрования паролей NiFi (по умолчанию = `PANEL_TOKEN`) |
| `CORS_ORIGINS` | Разрешённые origins через запятую |

## API

Основные группы: `/api/servers/*`, `/api/nifi/*`, `/api/alerts/*`.

Полная документация: http://localhost:8000/docs

## Подробная инструкция по развёртыванию

См. **[DEPLOY.md](DEPLOY.md)** — требования, сборка архива, перенос на air-gap сервер, `.env`, systemd, troubleshooting.

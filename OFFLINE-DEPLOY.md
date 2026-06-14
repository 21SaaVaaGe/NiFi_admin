# Перенос NiFi Admin на машину без интернета

Два способа: **офлайн-пакет (рекомендуется)** или **публичный Docker Registry**.

---

## Способ 1. Офлайн-пакет (USB / локальная сеть)

На **этой** машине (с интернетом) уже можно собрать пакет:

```powershell
cd d:\Projects\nifi-admin
.\scripts\export-offline-bundle.ps1
```

Результат: папка **`offline-bundle/`** (~280–350 MB):

| Файл | Назначение |
|------|------------|
| `images/nifi-admin-3.0.0.tar` | Панель (FastAPI + UI) |
| `images/postgres-16-alpine.tar` | База данных |
| `images/minio-minio-latest.tar` | S3-хранилище бэкапов |
| `images/minio-mc-latest.tar` | Инициализация bucket |
| `docker-compose.offline.yml` | Запуск без сборки |
| `.env.docker.example` | Настройки |
| `import.ps1` / `import.sh` | Загрузка образов в Docker |

### Перенос

1. Скопируйте **всю папку** `offline-bundle` на флешку или в общую папку.
2. На целевой машине установите **Docker Desktop** (Windows) или **Docker Engine** (Linux) — установщик тоже без интернета, если скачать заранее с [docker.com](https://www.docker.com/products/docker-desktop/).

### Запуск на целевой машине (Windows)

```powershell
cd D:\offline-bundle          # куда скопировали
.\import.ps1                  # загрузка образов (~1–2 мин)
copy .env.docker.example .env
notepad .env                  # ОБЯЗАТЕЛЬНО: JWT_SECRET, пароли, CORS
mkdir D:\Docker\nifi-admin\postgres, D:\Docker\nifi-admin\minio, D:\Docker\nifi-admin\app
docker compose -f docker-compose.offline.yml up -d
```

Панель: **http://localhost:8000** — логин **admin** / **admin**.

### Запуск на Linux

```bash
cd /opt/offline-bundle
chmod +x import.sh && ./import.sh
cp .env.docker.example .env && nano .env
mkdir -p /var/lib/nifi-admin/{postgres,minio,app}
# В .env: DOCKER_DATA_ROOT=/var/lib/nifi-admin
docker compose -f docker-compose.offline.yml up -d
```

### Перенос данных с текущей машины

Если нужно перенести **уже настроенные серверы NiFi и пользователей**:

```powershell
# Остановить на исходной машине
docker compose down

# Скопировать папки данных (путь из .env DOCKER_DATA_ROOT)
# D:\Docker\nifi-admin\postgres
# D:\Docker\nifi-admin\minio
# D:\Docker\nifi-admin\app

# На целевой машине — те же пути в .env и те же папки
```

---

## Способ 2. Публичный Docker Registry

Образ **приложения** можно выложить в Docker Hub. Образы `postgres` и `minio` на офлайн-машине всё равно нужно один раз скачать или положить в tar (как в способе 1).

### Docker Hub (ваш аккаунт)

```powershell
# 1. Регистрация на https://hub.docker.com

# 2. Вход
docker login

# 3. Публикация (замените YOUR_USER на логин)
.\scripts\push-docker-hub.ps1 -DockerUser YOUR_USER -Version 3.0.0
```

Образ будет: `YOUR_USER/nifi-admin:3.0.0`

На машине **с интернетом** (один раз):

```bash
docker pull YOUR_USER/nifi-admin:3.0.0
docker pull postgres:16-alpine
docker pull minio/minio:latest
docker pull minio/mc:latest
docker save -o nifi-admin-bundle.tar YOUR_USER/nifi-admin:3.0.0 postgres:16-alpine minio/minio:latest minio/mc:latest
```

Файл `nifi-admin-bundle.tar` переносите на офлайн-машину:

```bash
docker load -i nifi-admin-bundle.tar
```

В `docker-compose.offline.yml` замените `image: nifi-admin:3.0.0` на `image: YOUR_USER/nifi-admin:3.0.0`.

### GitHub Container Registry (ghcr.io)

```powershell
docker tag nifi-admin:3.0.0 ghcr.io/YOUR_GITHUB_USER/nifi-admin:3.0.0
docker login ghcr.io -u YOUR_GITHUB_USER
docker push ghcr.io/YOUR_GITHUB_USER/nifi-admin:3.0.0
```

---

## Что нельзя перенести только образом app

| Нужно отдельно | Почему |
|----------------|--------|
| PostgreSQL, MinIO | Отдельные образы (входят в offline-bundle) |
| Папки `postgres/`, `minio/`, `app/` | Данные: серверы, пользователи, бэкапы |
| Файл `.env` | Секреты и пути на целевой машине |
| Доступ к NiFi | IP NiFi в настройках сервера (не в образе) |

---

## Минимальный чеклист на офлайн-машине

- [ ] Docker установлен
- [ ] `import.ps1` выполнен без ошибок
- [ ] `.env` создан, `JWT_SECRET` и пароли изменены
- [ ] `DOCKER_DATA_ROOT` — существующие папки
- [ ] `CORS_ORIGINS` содержит URL, с которого открываете панель
- [ ] `docker compose -f docker-compose.offline.yml ps` — все сервисы healthy
- [ ] В UI добавлен NiFi-сервер по IP (не `localhost`, если NiFi на другой машине)

---

## Обновление версии позже

На машине с интернетом:

```powershell
.\scripts\docker-build.ps1
.\scripts\export-offline-bundle.ps1 -Version 3.0.1
```

На офлайн-машине: снова `import.ps1`, затем `docker compose -f docker-compose.offline.yml up -d` (данные в volumes сохранятся).

https://disk.yandex.ru/d/NfD4it8MXsExTw

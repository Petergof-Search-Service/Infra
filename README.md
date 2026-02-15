# Инфраструктура — nginx + фронтенд в Docker

Всё крутится в Docker: контейнер фронта (образ из registry) и nginx как reverse proxy на порту 80. Файлов фронта на диске сервера нет.

## Содержимое

| Файл | Назначение |
|------|------------|
| `docker-compose.yml` | Сервисы: frontend (образ из env/registry), nginx (прокси на frontend:80). |
| `nginx/frontend.conf` | Конфиг nginx: proxy_pass на контейнер frontend. |

## Как это работает

1. **Frontend-репо**  
   - При **push в `main`**: собирает образ с тегами `latest`, `stable`, sha → пушит в ghcr.io → вызывает `repository_dispatch` с типом `frontend-updated` и payload `image` → деплой на **прод**.  
   - При **push в любую другую ветку** (в т.ч. коммиты в PR): собирает образ с тегом `branch-shortSha` → пушит в ghcr.io → вызывает `repository_dispatch` с типом `frontend-test-updated` и payload `image` → деплой на **тест**.

2. **Этот репо (Infra)**  
   - **Прод**: при `push` в `main` или при событии `frontend-updated` — копирует репо в `$REMOTE_PATH` (при push), на целевом сервере выставляет `FRONTEND_IMAGE` из payload (при dispatch) и выполняет `docker compose pull` и `up -d`.  
   - **Тест**: при событии `frontend-test-updated` — то же самое на тестовом сервере в `$REMOTE_PATH_TEST`.

Образ по умолчанию на проде: `ghcr.io/petergof-search-service/frontend:latest`.

## Секреты репозитория (Infra)

**Прод-сервер:**

| Секрет | Описание |
|--------|----------|
| `SSH_PRIVATE_KEY` | Приватный ключ SSH для прод-сервера. |
| `SSH_HOST` | Хост прод-сервера (IP или домен). |
| `SSH_USER` | Пользователь SSH на прод-сервере. |

**Тестовый сервер (отдельные секреты):**

| Секрет | Описание |
|--------|----------|
| `SSH_PRIVATE_KEY_TEST` | Приватный ключ SSH для тестового сервера. |
| `SSH_HOST_TEST` | Хост тестового сервера. |
| `SSH_USER_TEST` | Пользователь SSH на тестовом сервере. |

**Pull приватного образа с ghcr.io** (если пакет frontend приватный — типично, если в организации отключены публичные пакеты):

| Секрет | Описание |
|--------|----------|
| `GHCR_TOKEN` | PAT с правом `read:packages` (для `docker pull` с ghcr.io на серверах). |
| `GHCR_USERNAME` | Логин GitHub (пользователь или организация, владелец пакета), например `Petergof-Search-Service`. |

## Секрет в репозитории Frontend

| Секрет | Описание |
|--------|----------|
| `INFRA_REPO_TOKEN` | PAT с правом `repo` для вызова `repository_dispatch` в репо Infra. |

## Требования на серверах

- На **обоих** серверах: Docker и Docker Compose (v2), пользователь SSH в группе `docker` (или иначе может выполнять `docker compose`).
- Каталоги `$REMOTE_PATH` (прод) и `$REMOTE_PATH_TEST` (тест) создаются workflow при первом деплое.

## Поведение

- **Порт 80** — nginx слушает на хосте и проксирует запросы на контейнер `frontend:80`.
- **SPA** — роутинг и отдача статики выполняются контейнером фронта (в образе свой nginx).

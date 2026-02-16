# Инфраструктура — nginx + фронтенд в Docker

Всё крутится в Docker: контейнер фронта (образ из registry) и nginx как reverse proxy на порту 80. Файлов фронта на диске сервера нет.

## Содержимое

| Файл | Назначение |
|------|------------|
| `docker-compose.yml` | Сервисы: frontend (образ из env/registry), nginx (80 и 443, прокси на frontend, сертификаты Let's Encrypt). |
| `nginx/frontend-ssl.conf.template` | Шаблон с плейсхолдером `{{DOMAIN}}`; при деплое из него генерируется `frontend.conf` с доменом прод/тест. |

## Как это работает

1. **Frontend-репо**  
   - При **push в `main`**: собирает образ с тегами `latest`, `stable`, sha → пушит в ghcr.io → вызывает `repository_dispatch` с типом `frontend-updated` и payload `image` → деплой на **прод**.  
   - При **push в любую другую ветку** (в т.ч. коммиты в PR): собирает образ с тегом `branch-shortSha` → пушит в ghcr.io → вызывает `repository_dispatch` с типом `frontend-test-updated` и payload `image` → деплой на **тест**.

2. **Этот репо (Infra)**  
   - **Прод**: при `push` в `main` или при событии `frontend-updated` — генерирует `frontend.conf` из шаблона с доменом petergof-sciense-rag.ru, копирует репо в `$REMOTE_PATH`, на сервере выполняет `docker compose pull` и `up -d`.  
   - **Тест**: при событии `frontend-test-updated` — то же с доменом test.petergof-sciense-rag.ru в `$REMOTE_PATH_TEST`.

Образ по умолчанию на проде: `ghcr.io/petergof-search-service/frontend:latest`.

## Секреты организации

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

**Секреты для доступа в Infra из других Actions:**

| Секрет | Описание |
|--------|----------|
| `INFRA_REPO_TOKEN` | PAT с правом `repo` для вызова `repository_dispatch` в репо Infra. |

## Домен и HTTPS

### Стенды

   Тестовый стенд https://test.petergof-sciense-rag.ru

   Продовый стенд https://petergof-sciense-rag.ru

### Первичная установка
   1) Установка certbot `sudo apt update && sudo apt install -y certbot`
   2) Выпуск ssl сертификата для домена (подставить нужны для прода или тестинга) `sudo certbot certonly --standalone -d petergof-sciense-rag.ru`

## Требования на серверах

- На **обоих** серверах: Docker и Docker Compose (v2), пользователь SSH в группе `docker` (или иначе может выполнять `docker compose`).
- Каталоги `$REMOTE_PATH` (прод) и `$REMOTE_PATH_TEST` (тест) создаются workflow при первом деплое.
- Для HTTPS на **проде**: certbot и `/etc/letsencrypt/live/petergof-sciense-rag.ru/`. На **тесте**: certbot и сертификат для test.petergof-sciense-rag.ru.

## Поведение

- **Порт 80** — nginx слушает на хосте и проксирует запросы на контейнер `frontend:80`.
- **SPA** — роутинг и отдача статики выполняются контейнером фронта (в образе свой nginx).

# Инфраструктура — nginx + фронтенд в Docker

Всё крутится в Docker: контейнер фронта (образ из registry) и nginx как reverse proxy на порту 80. Файлов фронта на диске сервера нет.

## Содержимое

| Файл | Назначение |
|------|------------|
| `docker-compose.yml` | Сервисы: frontend (образ из env/registry), nginx (прокси на frontend:80). |
| `docker-compose.test.yml` | Оверрайд для теста: порт 443, HTTPS, сертификаты Let's Encrypt. |
| `nginx/frontend.conf` | Конфиг nginx для прод: proxy_pass на контейнер frontend. |
| `nginx/frontend-test.conf` | Конфиг для теста: домен, HTTPS, редирект 80→443. |

## Как это работает

1. **Frontend-репо**  
   - При **push в `main`**: собирает образ с тегами `latest`, `stable`, sha → пушит в ghcr.io → вызывает `repository_dispatch` с типом `frontend-updated` и payload `image` → деплой на **прод**.  
   - При **push в любую другую ветку** (в т.ч. коммиты в PR): собирает образ с тегом `branch-shortSha` → пушит в ghcr.io → вызывает `repository_dispatch` с типом `frontend-test-updated` и payload `image` → деплой на **тест**.

2. **Этот репо (Infra)**  
   - **Прод**: при `push` в `main` или при событии `frontend-updated` — копирует репо в `$REMOTE_PATH` (при push), на целевом сервере выставляет `FRONTEND_IMAGE` из payload (при dispatch) и выполняет `docker compose pull` и `up -d`.  
   - **Тест**: при событии `frontend-test-updated` — то же самое на тестовом сервере в `$REMOTE_PATH_TEST` (с `docker-compose.test.yml` для HTTPS).

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

## Секрет в репозитории Frontend

| Секрет | Описание |
|--------|----------|
| `INFRA_REPO_TOKEN` | PAT с правом `repo` для вызова `repository_dispatch` в репо Infra. |

## Домен и HTTPS для тестового сервера

Чтобы тестовый сервер работал по домену с HTTPS:

1. **Домен**  
   Заведите домен или поддомен (например `test.example.com`), который будет указывать на тестовый сервер.

2. **DNS**  
   Создайте A-запись (или CNAME) на публичный IP тестового сервера. Дождитесь распространения DNS.

3. **Замените плейсхолдер в конфиге**  
   В файле `nginx/frontend-test.conf` замените **все** вхождения `YOUR_TEST_DOMAIN` на ваш домен (например `test.example.com`). Пути к сертификатам и `server_name` должны совпадать с доменом, для которого вы получите сертификат.

4. **Сертификат Let's Encrypt на тестовом сервере**  
   Один раз на тестовом сервере (по SSH):

   - Установите certbot, если ещё нет:  
     `sudo apt update && sudo apt install -y certbot` (или `snap install --classic certbot`).
   - Получите сертификат (порт 80 должен быть свободен; при первом запуске можно временно остановить контейнеры: `docker compose -f docker-compose.yml -f docker-compose.test.yml down`):  
     `sudo certbot certonly --standalone -d YOUR_TEST_DOMAIN`  
     (подставьте ваш домен). Certbot сохранит сертификаты в `/etc/letsencrypt/live/YOUR_TEST_DOMAIN/`.
   - Включите автообновление:  
     `sudo certbot renew --dry-run` (проверка), обновление по умолчанию через systemd timer или cron.

5. **Деплой**  
   После пуша в Infra и деплоя на тест workflow поднимет контейнеры с `docker-compose.test.yml`: nginx будет слушать 80 и 443 и отдавать HTTPS по вашему домену.

**Итог:** домен → DNS на IP теста → заменить `YOUR_TEST_DOMAIN` в `frontend-test.conf` → один раз получить сертификат certbot на сервере → деплой подхватит HTTPS-конфиг.

## Требования на серверах

- На **обоих** серверах: Docker и Docker Compose (v2), пользователь SSH в группе `docker` (или иначе может выполнять `docker compose`).
- Каталоги `$REMOTE_PATH` (прод) и `$REMOTE_PATH_TEST` (тест) создаются workflow при первом деплое.
- На **тестовом** сервере для HTTPS: certbot и каталог `/etc/letsencrypt` с сертификатом для выбранного домена.

## Поведение

- **Порт 80** — nginx слушает на хосте и проксирует запросы на контейнер `frontend:80`.
- **SPA** — роутинг и отдача статики выполняются контейнером фронта (в образе свой nginx).

# Продакшен-развёртывание на одном сервере

Status: In Progress
Bounded context: нет (инфраструктура, доменная модель не меняется)
Service: aeroflow-core, aeroflow-playback, aeroflow-tts, aeroflow-web, конфигурация развёртывания
Change type: Infrastructure
Depends on: 035 (transactional outbox)

## Goal

Сейчас стек запускается только локально: `dev.sh`, dev-образы с кодом через bind
mount, web на dev-сервере Vite. Нужен воспроизводимый prod-запуск core, playback,
tts и web на одном сервере (Hetzner Cloud, для начала 1 ГБ RAM, домен
`aeroflow.duckdns.org`) с HTTPS, сборкой образов и деплоем из GitHub Actions.

Агент в стек не входит: он работает на машине у динамиков. Его подключение к
серверу через интернет — задача 038.

## Decisions

* **Один сервер, docker compose.** Оркестратор (Kubernetes, ECS) для одной машины
  избыточен.
* **Образы собирает GitHub Actions** в каждом репозитории сервиса и публикует в
  GHCR: `ghcr.io/vasilii-kostiuc/aeroflow-{core,playback,tts,web}`, теги `latest`
  и sha коммита. На сервере ничего не собирается (на 1 ГБ сборка не помещается),
  откат сервиса — деплой с тегом предыдущего sha.
* **Деплой тоже из GitHub Actions:** по SSH на сервер — скачать образы,
  применить миграции, перезапустить сервисы.
* **Входная точка — Caddy** (образ web: собранная статика внутри Caddy). Он
  получает и продлевает сертификат Let's Encrypt для домена сам, отдаёт статику
  web, проксирует `/api` в php-fpm core и отвечает `404` на `/internal` до задачи
  038 (эндпоинт скачивания аудио для агента без авторизации). nginx + certbot
  рассматривался; вариант для него: certbot на хосте, первый сертификат через
  `standalone`, продление через `webroot` и reload nginx в hook. Отложен —
  возможен позже, если понадобится.
* **Наружу публикуются только 80 и 443.** Postgres, RabbitMQ, core, playback и tts
  доступны только внутри compose-сети. Hetzner Cloud Firewall пропускает 22, 80,
  443.
* **Секреты — только в `.env` на сервере.** Образы публичные, поэтому
  `.dockerignore` исключает `config/jwt/*.pem`, `.env.local`, `var`, `vendor`.
  JWT-ключи core генерируются на сервере (`lexik:jwt:generate-keypair
  --skip-if-exists`) в volume; ключ и passphrase из `aeroflow-core/.env` в prod
  не используются.
* **Миграции core и playback — шаг деплоя**, а не старта контейнера.
* **Outbox relay — ровно один контейнер.** Compose при обновлении останавливает
  старый контейнер до запуска нового, два relay одновременно не работают.

### Prod-образы

Черновик (ветки `chore/prod-deploy`) проверил:

* **core** — `php:8.5-fpm`, код в образе, `composer install --no-dev`,
  `cache:warmup` при сборке, тот же образ запускает воркер и outbox relay
  (команда задаётся в compose):
  * OPcache в PHP 8.5 встроен: `docker-php-ext-install opcache` падает, нужны
    только настройки в `php.ini` (`validate_timestamps=0`);
  * `pcntl` — чтобы `messenger:consume` по SIGTERM дорабатывал сообщение, а не
    обрывался при замене контейнера;
  * `clear_env = no` в пуле php-fpm: по умолчанию php-fpm не передаёт PHP
    переменные окружения контейнера, и Symfony молча взял бы dev-значения из
    `.env`;
  * `pm = ondemand`, `pm.max_children = 4` — под маленький сервер;
  * `doctrine/doctrine-fixtures-bundle` переносится из `require-dev` в `require`.
    Фикстуры (`src/*/Infrastructure/DataFixtures`) наследуют его `Fixture`, и без
    dev-зависимостей сборка падала дважды: `services.yaml` регистрирует как
    сервисы весь `src/`, а attribute-драйвер Doctrine загружает каждый файл `src/`
    в поисках сущностей. Бандл по-прежнему включён только в dev и test
    (`bundles.php`), поэтому `doctrine:fixtures:load`, который по умолчанию
    очищает базу, в prod недоступен. Рассматривались и отклонены: вынос фикстур
    из `src/` в каталог с `autoload-dev` (фикстуры уходят из своих bounded
    contexts) и исключение их каталогов в `services.yaml` и attribute-драйвере
    Doctrine через compiler pass (нетривиальная настройка ради расположения
    файлов).
* **playback** — `php:8.5-cli`, HTTP нет, только воркеры и миграции, `USER
  www-data`.
* **tts** — существующий `Dockerfile.piper` без изменений, добавить
  `.dockerignore`.
* **web** — сборка в `node:22-alpine`, статика в образе `caddy:2-alpine`,
  `VITE_CORE_API_URL=/api` (тот же origin).
* **Caddy → core:** явная FastCGI-настройка (`rewrite * /index.php` и `transport
  fastcgi { root /var/www/public }`). Сокращённый `php_fastcgi` ищет файлы на
  диске Caddy, а файлов core в его контейнере нет.

### Конфигурация развёртывания

`docker-compose.yml`, конфиг Caddy, `.env.example`, `deploy.sh`, `README.md` —
в каталоге `deploy/` корневого репозитория AeroFlow, workflow деплоя — в его
`.github/workflows/`. Корневой репозиторий уже хранит общие файлы (`dev.sh`,
`RUNNING.md`); отдельный репозиторий `aeroflow-deploy` рассматривался и отклонён.

Кнопка ручного запуска workflow появляется только на ветке по умолчанию, поэтому
в корневом репозитории наводится порядок в ветках: `master` перематывается на
актуальную ветку (сейчас `chore/dev-stack-tts-and-host-agent`, `master` — её
предок), публикуется и становится веткой по умолчанию на GitHub вместо
`feature/012-dispatcher-panel`.

### Запуск деплоя

* Кнопкой «Run workflow» в корневом репозитории и автоматически при изменении
  `deploy/` в `master`. SSH-ключ и адрес сервера хранятся только там. После
  сборки нового образа сервиса деплой — нажатие кнопки.
* workflow копирует `deploy/` на сервер (кроме `.env`) и запускает `deploy.sh`:
  скачать образы, сгенерировать JWT-ключи при отсутствии, миграции core и
  playback, перезапуск.
* Автоматический деплой после сборки образа (`repository_dispatch` с токеном в
  каждом репозитории сервиса) и отдельный job деплоя в каждом сервисе
  рассматривались; первый — следующий шаг, когда деплоев станет много.
* GitHub подключается к серверу по SSH с меняющихся адресов, поэтому порт 22 в
  Cloud Firewall открыт для всех (вход только по ключу). Для GitHub — отдельный
  ключ, не личный.

### Базы данных и брокер

* Один Postgres, две базы (`aeroflow` и `aeroflow_playback`) с разными
  пользователями; базу playback создаёт init-скрипт при первом запуске пустого
  volume. Сервисы по-прежнему не делят базу — общий у них только сервер.
* Один пользователь RabbitMQ для core и playback (брокер наружу не опубликован);
  отдельный пользователь с правами только на очереди агента — в задаче 038.
* Фиксированный `hostname` у RabbitMQ: он хранит данные по имени узла, без этого
  очереди пропадут при пересоздании контейнера.

### Память (1 ГБ RAM)

* Лимиты памяти: core-app 256 МБ, воркеры по 192 МБ, tts 512 МБ. Postgres и
  RabbitMQ без лимита, Postgres — `shared_buffers=64MB`, `max_connections=30`.
* Воркеры — `--memory-limit=128M --time-limit=3600`, перезапуск через `restart:
  unless-stopped`.
* tts работает постоянно: Piper запускается отдельным процессом на каждый запрос,
  между запросами сервис почти не занимает памяти.
* На сервере swap 2 ГБ. Сервер увеличивать, если swap постоянно занят, web
  тормозит или в `dmesg` есть `Out of memory`.

## Out of Scope

* подключение агента через интернет (amqps, токен для `/internal`) — задача 038;
* бэкапы Postgres и аудиофайлов, внешняя проверка доступности;
* тесты в CI: workflow только собирает и публикует образ;
* nginx + certbot вместо Caddy;
* несколько серверов, высокая доступность, outbox для playback.

## Acceptance Criteria

* push в `master` репозитория сервиса публикует образ в GHCR с тегами `latest` и
  sha коммита;
* деплой из GitHub Actions поднимает стек: `https://aeroflow.duckdns.org`
  открывает web с валидным сертификатом, вход работает, `/api` отвечает;
* снаружи открыты только 22, 80 и 443;
* миграции core и playback применяются при деплое, повторный деплой безопасен;
* JWT-ключей и секретов нет в образах;
* экран «Состояние системы» показывает воркеры core и playback и tts
  работающими (агент — «Не подключён» до задачи 038);
* после перезагрузки сервера стек поднимается сам.

## Verification

```bash
# локально: prod-стек из конфигурации развёртывания с DOMAIN=localhost
./deploy.sh
curl -sk -o /dev/null -w '%{http_code}\n' https://localhost/api/v1/me   # 401
curl -sk -o /dev/null -w '%{http_code}\n' https://localhost/internal/x  # 404

# с внешней машины
nmap -Pn aeroflow.duckdns.org   # открыты только 22, 80, 443
```

## Documentation Updates

* `architecture.md`: раздел «Развёртывание» — один сервер, compose, образы из
  GHCR, наружу только Caddy (80/443), агент вне стека.
* `deploy/README.md` в корневом репозитории: первый запуск сервера, деплой, откат
  по тегу, первый администратор.

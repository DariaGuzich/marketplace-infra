# marketplace-infra

Запуск всей системы одной командой (docker-compose), Toxiproxy для сбоев между сервисами и
[EXPERIMENTS.md](EXPERIMENTS.md) — эксперименты по всей системе.

## Репозитории

| Репозиторий | Что это | Порт |
|---|---|---|
| [marketplace-ui](https://github.com/DariaGuzich/marketplace-ui) | страница настроек (React) | 5173 |
| [marketplace-bff](https://github.com/DariaGuzich/marketplace-bff) | GraphQL BFF (Node.js, TypeScript) | 4000 |
| [marketplace-api](https://github.com/DariaGuzich/marketplace-api) | REST API настроек, outbox (Java, Spring Boot) | 8080 |
| [marketplace-publisher](https://github.com/DariaGuzich/marketplace-publisher) | Config publisher: outbox → Serving | — |
| [marketplace-serving](https://github.com/DariaGuzich/marketplace-serving) | фейковый Serving: принимает конфиги, решает о показе, шлёт события | 8081 |
| [marketplace-reporting](https://github.com/DariaGuzich/marketplace-reporting) | события, почасовые агрегаты | 8082 |
| marketplace-infra (этот) | docker-compose, Toxiproxy, эксперименты | — |

```
ui ──► bff ──[toxiproxy: bff_to_api]──────► api ──► PostgreSQL: public (settings, outbox, idempotency_keys)
        └───[toxiproxy: bff_to_reporting]─► reporting ──► PostgreSQL: reporting (events, hourly_stats)
                                                ▲
publisher ──► PostgreSQL (outbox)               │ события (напрямую)
    └──[toxiproxy: publisher_to_serving]──► serving
```

## Запуск всей системы

Нужен Docker Desktop. Все репозитории должны лежать **рядом** в одной папке: compose собирает образы из
`../marketplace-*`.

```bash
docker compose up --build -d     # собрать и запустить всё (первая сборка — несколько минут)
docker compose ps                # статус
docker compose logs -f bff       # логи одного сервиса
docker compose down              # остановить (данные PostgreSQL сохраняются)
docker compose down -v           # остановить и удалить данные
```

Перед запуском останови локально запущенные сервисы (`mvn spring-boot:run`, `npm start`, `npm run dev`):
они занимают те же порты. Особенно коварен локальный `npm run dev` в marketplace-ui. Docker публикует порт
на `0.0.0.0:5173`, а Vite слушает `[::1]:5173`, оба запускаются без ошибки, и `localhost:5173` в браузере
может открыть локальный Vite, а не контейнер.

Проверка:
1. http://localhost:5173 — страница настроек (UI → BFF → API);
2. через 3 секунды `curl localhost:8081/debug/config/acc-1` — конфиг доставлен в Serving (API → outbox → publisher);
3. `curl -X POST localhost:8081/ad-request -H "Content-Type: application/json" -d '{"account_id": "acc-1", "domain": "news.com", "bid_price": 2}'`
   — событие в `reporting.events`.

Переменные, которые можно передать при запуске (`USE_DATALOADER=false docker compose up -d bff`):
`OPTIMISTIC_LOCKING_ENABLED` (api), `CRASH_AFTER_SEND` (publisher), `USE_DATALOADER` (bff), `AGGREGATION_CRON` (reporting).

Консоль базы:

```bash
docker compose exec postgres psql -U marketplace
```

```sql
SELECT * FROM settings;
SELECT id, account_id, version, status, payload FROM outbox ORDER BY id;
SELECT idempotency_key, response FROM idempotency_keys;
SELECT * FROM reporting.events ORDER BY event_time;
SELECT * FROM reporting.hourly_stats ORDER BY hour, account_id;
```

## Toxiproxy: сбои между сервисами

Toxiproxy — TCP-прокси, в который можно на лету добавлять «токсины»: задержку, обрыв, сброс соединения.
Сервисы ходят к соседям не напрямую, а через него (адреса в `docker-compose.yml`, прокси в `toxiproxy.json`):

| Прокси | Связь | Таймаут на стороне вызывающего |
|---|---|---|
| `bff_to_api` | BFF → API | 2 с (`API_TIMEOUT_MS`) |
| `bff_to_reporting` | BFF → Reporting | 1 с (`REPORTING_TIMEOUT_MS`) |
| `publisher_to_serving` | Publisher → Serving | 2 с (connect и read) |

Связь Serving → Reporting идёт напрямую. Её сбой воспроизводится остановкой контейнера:
`docker compose stop reporting`.

### Команды

Это команды `toxiproxy-cli` внутри контейнера. **В Git Bash** перед ними нужен `MSYS_NO_PATHCONV=1`, иначе Git Bash
превратит путь `/toxiproxy-cli` в `C:/Program Files/Git/toxiproxy-cli`. Проще один раз выполнить
`export MSYS_NO_PATHCONV=1`. В PowerShell это не нужно.

```bash
# Список прокси
docker compose exec toxiproxy /toxiproxy-cli list
docker compose exec toxiproxy /toxiproxy-cli inspect bff_to_api

# Задержка ответов на 3 секунды (больше таймаута вызывающего → таймаут)
docker compose exec toxiproxy /toxiproxy-cli toxic add -t latency -a latency=3000 bff_to_api
docker compose exec toxiproxy /toxiproxy-cli toxic add -t latency -a latency=3000 bff_to_reporting
docker compose exec toxiproxy /toxiproxy-cli toxic add -t latency -a latency=3000 publisher_to_serving

# Убрать задержку (имя токсина по умолчанию — latency_downstream)
docker compose exec toxiproxy /toxiproxy-cli toxic remove -n latency_downstream bff_to_api

# Оборвать связь: прокси выключен, соединение отклоняется (как будто сосед лежит)
docker compose exec toxiproxy /toxiproxy-cli toggle bff_to_api      # повторный toggle включает обратно

# Сбросить соединение (TCP reset), как при падении соседа посреди ответа
docker compose exec toxiproxy /toxiproxy-cli toxic add -t reset_peer -a timeout=0 bff_to_reporting
docker compose exec toxiproxy /toxiproxy-cli toxic remove -n reset_peer_downstream bff_to_reporting
```

`latency` по умолчанию задерживает **ответ** (downstream), а запрос доходит до соседа сразу. Так выглядит
самый коварный сбой: сосед **выполнил** операцию, а вызывающий не дождался ответа и повторяет её.

То же доступно по HTTP API Toxiproxy на `localhost:8474`, например:
`curl -X POST localhost:8474/proxies/bff_to_api/toxics -d '{"type": "latency", "attributes": {"latency": 3000}}'`.

## Инструменты

- **Docker Compose** — вся система с одинаковыми настройками у всех одной командой. Образы собираются из
  Dockerfile каждого репозитория (multi-stage: Maven/npm внутри контейнера, локальные Java и Node не нужны).
- **Toxiproxy** — управляемые сбои сети между сервисами без изменения их кода: задержки, обрывы, сброс
  соединения. Позволяет вручную проверить таймауты, повторы и защиту от дублей.

## Сознательные упрощения

- UI в контейнере запускается dev-сервером Vite, а не собранной статикой за nginx.
- У Java-сервисов нет healthcheck: `depends_on` ждёт только запуска контейнера. На пустой базе publisher может
  начать опрос раньше, чем API применит миграции. Тогда первые циклы, ожидаемо, упадут (нет таблицы outbox),
  а следующие пройдут. Не проверялось: при проверке база уже была создана.
- Все сервисы ходят в одну базу под одним пользователем. В реальной системе у каждого сервиса свои учётные
  данные и права только на свою схему.
- Serving → Reporting не идёт через Toxiproxy (так в задании). Добавить его — ещё одна запись в `toxiproxy.json`
  и `REPORTING_URL: http://toxiproxy:<порт>` у serving.

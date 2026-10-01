# marketplace-infra

Общая инфраструктура учебного проекта: docker-compose, инструкции по запуску всей системы и
[EXPERIMENTS.md](EXPERIMENTS.md) — эксперименты по всей системе.

## Репозитории

| Репозиторий | Что это | Порт |
|---|---|---|
| [marketplace-api](https://github.com/DariaGuzich/marketplace-api) | REST API настроек (Java, Spring Boot), PostgreSQL | 8080 |
| [marketplace-bff](https://github.com/DariaGuzich/marketplace-bff) | GraphQL BFF (Node.js, TypeScript) | 4000 |
| [marketplace-ui](https://github.com/DariaGuzich/marketplace-ui) | страница настроек (React) | 5173 |
| marketplace-infra (этот) | docker-compose, общие инструкции | — |

```
marketplace-ui → marketplace-bff → marketplace-api → PostgreSQL
```

## Что поднимает docker-compose

Пока только **PostgreSQL** (база `marketplace`, пользователь и пароль `marketplace`, порт 5432).
Сервисы запускаются из своих репозиториев (см. ниже). Позже сюда добавятся все сервисы и Toxiproxy.

Нужен запущенный Docker Desktop.

```bash
docker compose up -d        # запустить PostgreSQL в фоне
docker compose ps           # статус (должно быть healthy)
docker compose logs -f      # логи
docker compose down         # остановить (данные сохраняются)
docker compose down -v      # остановить и удалить данные: следующий запуск начнётся с пустой базы
```

Открыть консоль базы:

```bash
docker compose exec postgres psql -U marketplace
```

Полезные запросы в psql:

```sql
\dt                                                        -- таблицы
SELECT * FROM settings;
SELECT id, account_id, version, status, payload FROM outbox ORDER BY id;
SELECT version, description, success FROM flyway_schema_history;   -- применённые миграции
```

## Запуск всей системы

По порядку, каждый пункт — в своём окне терминала:

1. `docker compose up -d` — здесь, в marketplace-infra;
2. `mvn spring-boot:run` — в marketplace-api (при старте Flyway применит миграции);
3. `npm start` — в marketplace-bff;
4. `npm run dev` — в marketplace-ui, затем открыть http://localhost:5173.

## Инструменты

- **Docker Compose** — поднимает инфраструктуру одной командой с одинаковыми настройками у всех.
  Без него каждому пришлось бы устанавливать PostgreSQL и вручную создавать базу и пользователя.

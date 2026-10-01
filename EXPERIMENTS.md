# Эксперименты

Эксперименты, которые проводятся вручную, чтобы увидеть, какие механизмы ловят изменения и сбои.
Если не сказано иное, эксперимент делается в отдельной ветке, PR не мёржится, а после эксперимента ветка удаляется.

Обозначения механизмов:

| Механизм | Где | Команда |
|---|---|---|
| **API-тесты** | marketplace-api | `mvn verify` (CI: job `test`) |
| **oasdiff** | CI marketplace-api, только в PR | job `breaking-changes` |
| **tsc BFF** | marketplace-bff | `npm run gen:api` → `npm run typecheck` |
| **мок на типах / мок без типов** | marketplace-bff/test | ошибки `tsc` в `resolvers.typed-mock.test.ts` / `resolvers.untyped-mock.test.ts` |
| **тесты BFF** | marketplace-bff | `npm test` (vitest типы не проверяет) |
| **GraphQL Inspector** | CI marketplace-bff, только в PR | job `schema-breaking-changes` (локально `npm run schema:diff`) |
| **codegen UI** | marketplace-ui | `npm run gen`: запросы UI проверяются по схеме BFF |
| **tsc UI** | marketplace-ui | `npm run gen` → `npm run typecheck` |

Важно: CI BFF и UI не запускаются от изменений в соседнем репозитории. Чтобы увидеть, как изменение
в API ломает BFF, изменение нужно **вмёржить** в main API, а затем запустить CI BFF вручную (Actions → CI →
Run workflow) или локально выполнить `npm run gen:api && npm run typecheck && npm test`. То же для BFF → UI.
После такого эксперимента верни main обратно revert-PR (`git revert -m 1 <merge-коммит>`).

---

## Сводка: что ловит каждый эксперимент

✅ — ловит, ❌ — не ловит, — — не относится. Эксперимент 1 проведён, остальные строки — прогноз по коду, проверь его.

| # | Эксперимент | oasdiff | tsc BFF (код) | мок на типах | мок без типов | GraphQL Inspector | codegen / tsc UI |
|---|---|---|---|---|---|---|---|
| 1 | переименовать поле ответа API | ✅ | ✅ | ✅ | ❌ | — | — |
| 2 | удалить поле ответа API | ✅ | ✅ | ✅ | ❌ | — | — |
| 3 | добавить необязательное поле | ❌ (и не должен) | ❌ | ❌ | ❌ | — | — |
| 4 | добавить обязательное поле в запрос | ✅ | ✅ | ✅ | ❌ | — | — |
| 5 | число → строка | ✅ | ✅ | ✅ | ❌ | — | — |
| 6 | цена в центах вместо долларов | ❌ | ❌ | ❌ | ❌ | — | — |
| 7 | удалить поле GraphQL | — | ❌ | — | — | ✅ | ✅ codegen |
| 8 | переименовать поле GraphQL | — | ❌ | — | — | ✅ | ✅ codegen |
| 9 | `@deprecated`, затем удалить | — | ❌ | — | — | ✅ только удаление | ✅ codegen только удаление |
| 10a | поле ответа `Float!` → `Float` | — | ❌ | — | — | ✅ | ❌ |
| 10b | `Settings` → `Settings!` | — | ❌ | — | — | ❌ (и не должен) | ✅ tsc на моке |
| 11 | обязательный аргумент | — | ❌ | — | — | ✅ | ✅ codegen |
| 12 | переименование через обе границы | ✅ | ✅ | ✅ | ✅ во время выполнения | ❌ (и не должен) | ❌ (и не должен) |

---

## REST API (marketplace-api)

В API две модели: `SettingsValues` — тело запроса PUT, `Settings` — ответ (значения + `version`).
В BFF им соответствуют типы `ApiSettingsValues` и `ApiSettings` (`src/api/client.ts`).

### 1. Переименовать поле в ответе ✔ проведён

**Что изменить:** в `Settings.java` `@JsonProperty("floor_price")` → `@JsonProperty("min_price")`.
(Когда эксперимент проводился, запрос и ответ были одной моделью, поэтому поле переименовалось и в запросе.)

**Что произошло:**
- `mvn verify` упал на `SettingsApiTest`: тест отправляет и проверяет `floor_price`. Первыми замечают
  собственные API-тесты. Их пришлось поправить вместе с кодом.
- oasdiff в PR: `removed the required property 'floor_price' from the response` (error),
  `added the new required request property 'min_price'` (error), `removed the request property 'floor_price'` (warning).
- После merge, CI BFF: `gen:api` → предупреждение «contract changed»; `tsc` упал в `src/resolvers.ts`
  (`fromApi` и `toApi`) и в `resolvers.typed-mock.test.ts` (2 места). Мок без типов молчал.
  Шаг тестов не запустился, а если запустить локально, оба варианта зелёные: vitest не проверяет типы.
- GraphQL Inspector и UI ничего не заметили: `schema.graphql` не менялся.

### 2. Удалить поле из ответа

**Что изменить:** убрать `currency` из `Settings.java` (и из `SettingsEntity.toSettings()`).

**Ожидаемо упадёт:**
- API-тесты: `SettingsApiTest` проверяет `currency` в ответе.
- oasdiff: `removed the required property 'currency' from the response` — error.
- tsc BFF: `settings.currency` в `fromApi` — свойства больше нет.
- мок на типах: `currency` в объекте `ApiSettings` — лишнее свойство.
- мок без типов и vitest: ничего, мок по-прежнему отдаёт `currency`.
- Inspector, UI: ничего. Вживую старый BFF отдаст `null` в `currency: String!` → ошибка в `errors`.

### 3. Добавить необязательное поле (не breaking change)

**Что изменить:** в `Settings.java` добавить `@JsonProperty("comment") String comment` без `@NotNull`.

**Ожидаемо:**
- oasdiff: зелёный. Новое необязательное поле в ответе не ломает клиентов.
- CI BFF: `gen:api` → предупреждение «contract changed» (в типах появилось `comment?: string`), `tsc` и тесты зелёные.
- Ничего не падает — так и должно быть. Предупреждение подсказывает: перегенерируй и закоммить типы,
  чтобы новое поле можно было использовать.

### 4. Добавить обязательное поле в запрос

**Что изменить:** в `SettingsValues.java` добавить `@JsonProperty("timezone") @NotNull String timezone`
(и в `SettingsEntity` + миграцию, если хочешь его хранить).

**Ожидаемо упадёт:**
- API-тесты: PUT без `timezone` → 400.
- oasdiff: `added the new required request property 'timezone'` — error.
- tsc BFF: `toApi` возвращает объект без `timezone`, а тип `ApiSettingsValues` его требует.
- мок на типах: ожидаемое тело PUT (`ApiSettingsValues`) без `timezone`.
- мок без типов и vitest: ничего. Вживую BFF получит 400 → `Marketplace API responded with 400` в `errors`.

### 5. Поменять тип поля (число → строка)

**Что изменить:** в `Settings.java` `BigDecimal floorPrice` → `String floorPrice`
(в `toSettings()` — `floorPrice.toPlainString()`).

**Ожидаемо упадёт:**
- oasdiff: тип свойства ответа изменился с `number` на `string` — error.
- tsc BFF: в `fromApi` `string` нельзя присвоить `number` (поле `floorPrice` типа `Settings`).
- мок на типах: `floor_price: 1.5` — `number` нельзя присвоить `string` (проверено подменой типа: ошибки в
  `resolvers.ts` и в моке на типах, мок без типов молчит, vitest зелёный).
- мок без типов и vitest: ничего.
- Интересно вживую: старый BFF вернёт строку `"1.5"` в поле `Float!`. GraphQL, скорее всего, молча превратит
  её в число, и UI ничего не заметит. Проверь в GraphiQL.

### 6. Поменять смысл без изменения формата (цена в центах)

**Что изменить:** API начинает отдавать `floor_price` в центах: `150` вместо `1.5`.
Например, в `toSettings()` умножать на 100.

**Ожидаемо:**
- Падают только API-тесты, где явно проверяется значение (`equalTo(1.5f)`), и только пока их не «поправили»
  вместе с кодом.
- oasdiff, tsc, моки, Inspector, UI — **ничего**. Формат прежний: `number`.
- UI показывает 150 там, где пользователь вводил 1.5.

**Чем защищаются:** не менять смысл существующего поля, а добавить новое (`floor_price_cents`) и пометить старое
устаревшим; описание единиц в спецификации (`@Schema(description = ...)`); контрактные тесты с конкретными
значениями. Pact по умолчанию сравнивает типы, а не значения, поэтому, скорее всего, тоже не поймает.

---

## GraphQL-схема BFF (marketplace-bff)

У BFF нет сгенерированных типов для **своей** схемы: резолверы не связаны с `schema.graphql` на уровне TypeScript.
Поэтому изменения схемы ловят Inspector, тесты BFF (они отправляют настоящие запросы) и UI.

### 7. Удалить поле

**Что изменить:** в `schema.graphql` убрать `blockedDomains` из `type Settings`.

**Ожидаемо:**
- tsc BFF: ничего.
- тесты BFF: ❌ оба варианта: запрос в тесте просит `blockedDomains` → ошибка валидации запроса.
- Inspector: `Field blockedDomains was removed from object type Settings` — breaking.
- UI (после merge в BFF): `npm run gen` падает — `Cannot query field "blockedDomains" on type "Settings"`.
- Поле, которое UI не запрашивает, удалилось бы без последствий для UI (Inspector всё равно покажет breaking:
  он не знает, кто что использует).

### 8. Переименовать поле

**Что изменить:** `floorPrice` → `minPrice` в `type Settings` (резолвер не трогать).

**Ожидаемо:**
- tsc BFF: ничего, хотя резолвер возвращает `floorPrice`, а схема ждёт `minPrice`.
- тесты BFF: ❌ запросы в тестах просят `floorPrice`. А если поправить запросы в тестах, но не резолвер:
  `Cannot return null for non-nullable field Settings.minPrice`.
- Inspector: `floorPrice` удалено (breaking), `minPrice` добавлено (не breaking).
- UI: `npm run gen` падает на `floorPrice`.
- Вывод: генерация типов резолверов из схемы (GraphQL Code Generator, плагин `typescript-resolvers`) поймала бы
  несоответствие резолвера и схемы на этапе `tsc`. Здесь её нет, чтобы не усложнять.

### 9. Пометить поле @deprecated, затем удалить

**Шаг 1:** добавить `minPrice: Float!` (резолвер возвращает то же значение) и пометить
`floorPrice: Float! @deprecated(reason: "Use minPrice")`.
- Inspector: изменения не breaking, CI зелёный.
- UI: `npm run gen` проходит; в `src/gql/graphql.ts` у поля появится `@deprecated`, WebStorm покажет
  `floorPrice` в запросе зачёркнутым. Ничего не падает — у клиентов есть время перейти.

**Шаг 2:** перевести UI на `minPrice`, вмёржить, затем удалить `floorPrice` из схемы BFF.
- Inspector: удаление всё равно breaking (по умолчанию он не делает исключения для deprecated-полей;
  есть правило `suppressRemovalOfDeprecatedField`, которое понижает это до «опасного»).
- UI: `npm run gen` проходит — UI уже не использует поле.

**Вывод:** `@deprecated` — механизм договорённости, а не проверки. Он даёт время, но решение «все перешли,
можно удалять» принимает человек (в реальных системах — по метрикам использования полей).

### 10. Поменять обязательность поля в ответе

**10a. Обязательное → необязательное:** `floorPrice: Float!` → `floorPrice: Float`.
- Inspector: breaking — клиенты не готовы к `null`.
- UI: тип станет `number | null`. `String(settings.floorPrice)` компилируется и с `null` (будет строка `"null"`),
  моки с числом тоже подходят, поэтому tsc UI ничего не заметит. Поймал только Inspector.

**10b. Необязательное → обязательное:** `settings(accountId: ID!): Settings` → `Settings!`.
- Inspector: не breaking — клиентам стало только проще.
- тесты BFF: ❌ тест «returns null when API responds 404»: резолвер возвращает `null` в non-null поле →
  `Cannot return null for non-nullable field Query.settings`.
- UI: tsc падает на моке `{ settings: null }` в `SettingsPage.test.tsx` — `null` больше не подходит по типу.
- Вывод: «не breaking по схеме» не значит «работает» — сам BFF нарушает своё новое обещание.

### 11. Добавить обязательный аргумент

**Что изменить:** `settings(accountId: ID!, currency: String!): Settings`.

**Ожидаемо:**
- tsc BFF: ничего.
- тесты BFF: ❌ запросы без `currency` → ошибка валидации.
- Inspector: `Argument currency: String! added to field Query.settings` — breaking.
- UI: `npm run gen` падает: аргумент `currency` обязателен, но не передан.
- Для сравнения: необязательный аргумент (`currency: String`) — не breaking, ничего не падает.

---

## Через обе границы

### 12. Переименовать поле в API, починить BFF, убедиться, что UI не изменился

1. В API: как в эксперименте 1 (`floor_price` → `min_price`), PR, merge.
2. В BFF: `npm run gen:api` → `npm run typecheck` (ошибки в `resolvers.ts` и в моке на типах).
3. Починить `fromApi`/`toApi` и мок на типах, подставив `min_price`.
4. `npm test`: ❌ **мок без типов** теперь падает во время выполнения. Он всё ещё отдаёт `floor_price`, а резолвер
   читает `min_price` → `Cannot return null for non-nullable field Settings.floorPrice`. Устаревший мок
   обнаружился только сейчас. Почини и его.
5. Закоммить в BFF через PR: `git diff schema.graphql` пустой, Inspector зелёный.
6. В UI: запустить CI вручную или `npm run gen` → `git diff src/gql` пустой, тесты зелёные.

**Вывод:** BFF поглотил изменение API, до UI оно не дошло.

---

## Шаг 1 второго этапа: PostgreSQL, Flyway, транзакции

Подготовка: `docker compose up -d` в marketplace-infra, `mvn spring-boot:run` в marketplace-api.
Консоль базы: `docker compose exec postgres psql -U marketplace` (в marketplace-infra).

Автотесты (marketplace-api, нужен Docker):
- `SettingsTransactionTest` — запись в outbox падает → settings не сохранились;
- `MigrationsTest` — все миграции на пустой базе; данные до V2 → после V2 на месте, `version = 0`;
- `SettingsServiceTest` — версии и записи в outbox.

### 13. Версии и outbox

1. Дважды отправь PUT с разными значениями, затем ещё раз с теми же:
   ```bash
   curl -X PUT localhost:8080/accounts/acc-1/settings -H "Content-Type: application/json" -d '{"floor_price": 1.5, "currency": "USD", "blocked_domains": []}'
   curl -X PUT localhost:8080/accounts/acc-1/settings -H "Content-Type: application/json" -d '{"floor_price": 2.0, "currency": "USD", "blocked_domains": []}'
   curl -X PUT localhost:8080/accounts/acc-1/settings -H "Content-Type: application/json" -d '{"floor_price": 2.0, "currency": "USD", "blocked_domains": []}'
   ```
2. **Ожидаемо:** `version` в ответах — 0, 1, 1. В psql `SELECT account_id, version, status, payload FROM outbox;` —
   две записи (версии 0 и 1), обе `NEW`. Третий PUT ничего не изменил, поэтому новой версии и записи нет.

### 14. Транзакция вручную

1. В psql «сломай» outbox: `ALTER TABLE outbox RENAME TO outbox_off;`
2. Отправь PUT с новыми значениями. **Ожидаемо:** 500, в логах API ошибка `relation "outbox" does not exist`.
3. GET настроек: значения и `version` **старые**. INSERT/UPDATE в settings был выполнен, но откатился вместе
   с упавшей записью в outbox.
4. Верни таблицу: `ALTER TABLE outbox_off RENAME TO outbox;`

### 15. Миграции на пустой базе

1. Останови API, затем `docker compose down -v && docker compose up -d` (база удалена и создана заново).
2. Запусти API. **Ожидаемо в логах:** `Migrating schema "public" to version "1 - create settings"`, затем 2 и 3.
3. В psql: `SELECT version, description, success FROM flyway_schema_history;` — три строки.
4. Перезапусти API: миграций нет (`Schema "public" is up to date`). Flyway применяет каждую миграцию один раз.

### 16. Миграция, меняющая данные

1. Пустая база (`docker compose down -v && docker compose up -d`).
2. Запусти API только с первой миграцией:
   `SPRING_FLYWAY_TARGET=1 mvn spring-boot:run` (PowerShell: `$env:SPRING_FLYWAY_TARGET=1; mvn spring-boot:run`).
   **Ожидаемо:** Flyway применит V1, после чего приложение **не стартует**: Hibernate (`ddl-auto=validate`) не найдёт
   колонку `version`. Так защищаются от запуска кода на несовместимой схеме.
3. В psql вставь «старые» данные:
   `INSERT INTO settings VALUES ('old-1', 1.5, 'USD', ARRAY['bad.com']);`
4. Запусти API обычным способом (в PowerShell сначала `Remove-Item Env:SPRING_FLYWAY_TARGET`).
   **Ожидаемо:** применятся V2 и V3; `SELECT * FROM settings;` — у `old-1` данные на месте и `version = 0`;
   `curl localhost:8080/accounts/old-1/settings` возвращает их с `"version": 0`.

### 17. Изменить уже применённую миграцию

1. Добавь пробел или комментарий в `V1__create_settings.sql` и перезапусти API.
2. **Ожидаемо:** приложение не стартует — `Migration checksum mismatch for migration version 1`.
   Flyway хранит контрольную сумму каждого применённого файла. Применённые миграции не правят — пишут новую.
3. Верни файл (`git checkout src/main/resources/db/migration`).

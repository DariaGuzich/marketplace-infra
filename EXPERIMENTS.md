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

**Наблюдение при внедрении (PR #3 в marketplace-api):** в ответе API появилось обязательное поле `version`,
а тело PUT стало отдельной схемой `SettingsValues`. oasdiff — **зелёный**: новое поле в ответе клиентов не ломает.
Но после `npm run gen:api` `tsc` BFF упал в трёх местах: `toApi` возвращал тип ответа `ApiSettings` (теперь в нём
обязательный `version`), а мок на типах собирал ответ без `version`. Во время выполнения BFF продолжал бы
работать. «Не breaking для протокола» не значит «код потребителя не придётся трогать»: это видно только по
сгенерированным типам у потребителя.

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

---

## Шаг 2 второго этапа: идемпотентность и race condition

Подготовка: `docker compose up -d` (здесь), `mvn spring-boot:run` в marketplace-api. Команды ниже — для **Git Bash**
(в PowerShell вместо `curl` пиши `curl.exe` и экранируй кавычки в JSON: `'{\"domain\": \"bad.com\"}'`).

```bash
J='Content-Type: application/json'
curl -X PUT localhost:8080/accounts/exp-2/settings -H "$J" -d '{"floor_price": 1.5, "currency": "USD", "blocked_domains": []}'
```

Автотесты (marketplace-api): `IdempotencyApiTest`, `OptimisticLockingApiTest`, `LostUpdateApiTest`.

### 18. Неидемпотентный POST и Idempotency-Key

1. Дважды без ключа (так выглядит retry клиента после таймаута):
   ```bash
   curl -X POST localhost:8080/accounts/exp-2/blocked-domains -H "$J" -d '{"domain": "spam.net"}'
   curl -X POST localhost:8080/accounts/exp-2/blocked-domains -H "$J" -d '{"domain": "spam.net"}'
   ```
   **Ожидаемо:** `"blocked_domains": ["spam.net", "spam.net"]`, версия выросла дважды, в outbox две новые записи.
2. Дважды с одним ключом:
   ```bash
   curl -X POST localhost:8080/accounts/exp-2/blocked-domains -H "$J" -H "Idempotency-Key: key-1" -d '{"domain": "bad.com"}'
   curl -X POST localhost:8080/accounts/exp-2/blocked-domains -H "$J" -H "Idempotency-Key: key-1" -d '{"domain": "bad.com"}'
   ```
   **Ожидаемо:** оба ответа одинаковые, включая `version`. `bad.com` в списке один раз, в outbox одна новая запись.
3. Тот же ключ, другой домен (`-d '{"domain": "other.com"}'`): **ожидаемо** вернётся сохранённый ответ, `other.com`
   не добавится. Это упрощение: настоящие API в таком случае отвечают ошибкой (см. README API).
4. Где смотреть: `SELECT idempotency_key, response FROM idempotency_keys;` в psql.

### 19. Lost update: последовательный сценарий

Двое редактируют одни настройки: оба прочитали `version`, первый сохранил, второй сохраняет поверх.

1. `curl localhost:8080/accounts/exp-2/settings` — запомни `version` (допустим, `N`).
2. «Пользователь B» меняет валюту:
   `curl -X PUT localhost:8080/accounts/exp-2/settings -H "$J" -d '{"floor_price": 1.5, "currency": "EUR", "blocked_domains": [], "version": N}'`
   → 200, версия `N+1`.
3. «Пользователь A» меняет цену, но шлёт то, что видел он, включая `version: N`:
   `curl -i -X PUT localhost:8080/accounts/exp-2/settings -H "$J" -d '{"floor_price": 9, "currency": "USD", "blocked_domains": [], "version": N}'`
   **Ожидаемо:** `409 Conflict`. A должен перечитать настройки и повторить.
4. Перезапусти API с выключенной блокировкой: `OPTIMISTIC_LOCKING_ENABLED=false mvn spring-boot:run`
   (PowerShell: `$env:OPTIMISTIC_LOCKING_ENABLED="false"; mvn spring-boot:run`) и повтори шаги 1–3.
   **Ожидаемо:** на шаге 3 — **200**, валюта снова `USD`: изменение B молча потеряно. В outbox обе версии,
   и Serving получит обе, но последней будет версия без изменения B.

### 20. Гонка: два PUT одновременно

```bash
V=$(curl -s localhost:8080/accounts/exp-2/settings | grep -o '"version":[0-9]*' | cut -d: -f2)
curl -s -w ' %{http_code}\n' -X PUT localhost:8080/accounts/exp-2/settings -H "$J" -d "{\"floor_price\": 2, \"currency\": \"USD\", \"blocked_domains\": [], \"version\": $V}" &
curl -s -w ' %{http_code}\n' -X PUT localhost:8080/accounts/exp-2/settings -H "$J" -d "{\"floor_price\": 3, \"currency\": \"USD\", \"blocked_domains\": [], \"version\": $V}" &
wait
```

**Ожидаемо (блокировка включена):** один ответ 200, второй 409. Какой из них победит, зависит от случая.

Чего ожидать на практике: запрос выполняется за миллисекунды, поэтому два `curl` почти никогда не
пересекаются во времени. Обычно первый успевает закоммитить, а второй получает 409 от проверки `version`
в `SettingsService` — как в последовательном сценарии. Если запросы всё-таки пересеклись (оба прочитали
версию до коммита первого), 409 даст `@Version` в Hibernate. Какой механизм сработал, видно в логах API:
`409 by client version check` или `409 by Hibernate @Version (concurrent update)`. Автотест
`twoParallelPutsWithSameVersion_oneSucceedsOneGets409` запускает запросы одновременно через `CountDownLatch`,
но и там порядок не гарантирован. Он проверяет только итог: один 200 и один 409. (При проверке 5 запусков
подряд запросы пересекались каждый раз, и 409 давал `@Version` в Hibernate.)

**С выключенной блокировкой** (`OPTIMISTIC_LOCKING_ENABLED=false`): если запросы не пересеклись, оба получат 200
(lost update, как в эксперименте 19). Если пересеклись, второй всё равно получит 409 от `@Version`:
одновременные записи Hibernate защищает всегда (см. README API, «Идемпотентность и optimistic locking»).

---

## Шаг 3 второго этапа: marketplace-serving

Подготовка: `mvn spring-boot:run` в marketplace-serving (порт 8081). Команды — для Git Bash.

```bash
J='Content-Type: application/json'
config() { curl -s -X POST localhost:8081/config -H "$J" -d "{\"account_id\": \"acc-1\", \"version\": $1, \"floor_price\": $2, \"currency\": \"USD\", \"blocked_domains\": [\"bad.com\"]}"; echo; }
ad()     { curl -s -X POST localhost:8081/ad-request -H "$J" -d "{\"account_id\": \"acc-1\", \"domain\": \"$1\", \"bid_price\": $2}"; echo; }
```

Автотесты (marketplace-serving): `ConfigStoreTest`, `AdDecisionTest`, `ServingApiTest`.

### 21. Версии: новая, старая, повторная

```bash
ad news.com 2      # {"shown":false,"reason":"no config for account",...} — конфиг ещё не доставлен
config 2 1.5       # {"applied":true,"current_version":2}
config 1 0.5       # {"applied":false,"current_version":2} — старая версия (доставка не по порядку)
config 2 9         # {"applied":false,"current_version":2} — та же версия ещё раз (повторная доставка)
config 3 3         # {"applied":true,"current_version":3}
curl localhost:8081/debug/config/acc-1     # version 3, floor_price 3
```

**Ожидаемо в логах Serving:** `Applied ... version 2`, `Ignored ... version 1 is not newer than current version 2`,
`Ignored ... version 2 ...`, `Applied ... version 3`.

Обрати внимание: `config 2 9` — та же версия, но **с другими данными** — тоже проигнорирован. Serving доверяет
номеру версии, а не содержимому. Если отправитель по ошибке присвоит одну версию разным данным, Serving этого
не заметит. Поэтому версии назначает только один источник — API в транзакции с изменением.

### 22. Решение о показе

После эксперимента 21 (`floor_price` 3, `bad.com` заблокирован):

```bash
ad news.com 2      # не показано: bid is below floor price
ad bad.com 100     # не показано: domain is blocked
ad news.com 5      # показано: ok
```

`config_version` в каждом ответе — по какой версии конфига принято решение. Это пригодится в шагах 4–5:
видно, «доехало» ли изменение настроек до Serving.

### 23. Перезапуск Serving

1. Останови и снова запусти Serving.
2. `curl -i localhost:8081/debug/config/acc-1` → **404**, `ad news.com 5` → `no config for account`.
3. Конфиг хранится в памяти и потерян. Publisher (шаг 4) уже доставил все версии и повторно их не пришлёт.
   Это сознательное упрощение (см. README marketplace-serving): настоящий Serving при старте загружает снапшот.

---

## Шаг 4 второго этапа: marketplace-publisher

Подготовка, каждое в своём окне:
1. `docker compose up -d` (здесь);
2. `mvn spring-boot:run` в marketplace-api (8080);
3. `mvn spring-boot:run` в marketplace-serving (8081);
4. publisher запускается в каждом эксперименте по-своему: `mvn spring-boot:run` в marketplace-publisher.

Команды — для Git Bash:

```bash
J='Content-Type: application/json'
put() { curl -s -X PUT localhost:8080/accounts/pub-1/settings -H "$J" -d "{\"floor_price\": $1, \"currency\": \"${2:-USD}\", \"blocked_domains\": []}"; echo; }
serving() { curl -s localhost:8081/debug/config/pub-1; echo; }
```

Где смотреть:
- **база:** `docker compose exec postgres psql -U marketplace -c "SELECT version, status FROM outbox WHERE account_id = 'pub-1' ORDER BY version;"`;
- **Serving:** `serving` и строки `Applied` / `Ignored` в логе Serving;
- **publisher:** строки `Delivered`, `Invalid config`, `Attempt`, `Serving unavailable`, `CRASH_AFTER_SEND` в его логе.

Автотесты (marketplace-publisher): `PublisherTest`, `CrashAfterSendTest`, `ConfigValidatorTest`;
(marketplace-api): `RollbackApiTest`.

Вручную проверены: 24, 25, 26 (шаги 1–2), 27, 28, 30 (шаги 1–3). Остальное — прогноз: валюта `dollars` и порядок
для нескольких аккаунтов подтверждены автотестами (`PublisherTest`), откат на невалидную версию (30.5) не проверялся.
Номера версий будут другими, если у `pub-1` уже есть версии.

### 24. Доставка и сравнение версии в БД и в Serving

1. Publisher запущен. `put 1.5`, через 3–4 секунды `serving`.
2. **Ожидаемо:** в outbox версия 0 со статусом `SENT`, в Serving `"version": 0`. Лог publisher:
   `Delivered account pub-1 version 0`.
3. Версия в API (`curl localhost:8080/accounts/pub-1/settings`) совпадает с версией в Serving. Если они
   различаются, значит, есть недоставленные записи (`NEW`) или publisher не работает. Это главная проверка
   «доехало ли изменение».

### 25. Отказ publisher

1. Останови publisher (Ctrl+C).
2. `put 2.5`, `put 3.5` → в outbox две записи `NEW`, `serving` показывает старую версию: изменения в API есть,
   до Serving они не дошли.
3. Запусти publisher. **Ожидаемо:** через 3 секунды обе записи `SENT`, в логе `Delivered ... version 1`,
   затем `version 2` (по порядку), Serving на последней версии. Ничего не потеряно: outbox сохранил всё,
   пока publisher лежал.

### 26. Невалидный конфиг

1. `put -1` (отрицательная цена), затем `put 4.5`.
2. **Ожидаемо:** в логе publisher `Invalid config, account pub-1 version N marked as FAILED: [floor_price must be >= 0, got -1]`,
   затем `Delivered ... version N+1`. В outbox: `FAILED`, потом `SENT`. Serving ни разу не видел невалидную
   версию.
3. То же с `put 5 dollars` (валюта не из 3 заглавных букв).
4. Обрати внимание: API сохранил невалидные настройки (`curl` к API их показывает), а в Serving их нет.
   Источник правды и Serving разошлись. Это следствие того, что валидация есть только в publisher.
   Правильнее отвергать такие значения ещё в API (400).

### 27. Повторная доставка (падение после отправки)

1. Останови publisher. `put 6.5`.
2. Запусти publisher с падением: `CRASH_AFTER_SEND=true mvn spring-boot:run`
   (PowerShell: `$env:CRASH_AFTER_SEND="true"; mvn spring-boot:run`).
3. **Ожидаемо:** в логе `Delivered account pub-1 version N`, затем
   `CRASH_AFTER_SEND: ... sent, but NOT marked as SENT. Halting the process.`, процесс завершился.
   Serving уже на версии N, а в outbox она `NEW`.
4. Запусти publisher без переменной (в PowerShell сначала `Remove-Item Env:CRASH_AFTER_SEND`).
   **Ожидаемо:** `Delivered account pub-1 version N, but Serving ignored it: current version is N`,
   в логе Serving `Ignored config ... version N is not newer than current version N`, в outbox `SENT`.
5. Вывод: дубль возник и был безопасно отброшен получателем. Так работает at-least-once + идемпотентный получатель.

### 28. Отказ Serving и retry

1. Publisher запущен. Останови Serving. `put 7.5`.
2. **Ожидаемо** в логе publisher, по кругу раз в ~3 секунды:
   `Attempt 1/3 ... failed: I/O error ... Connection refused`, `Attempt 2/3`, `Attempt 3/3`,
   `Serving unavailable, stopping this cycle. Account pub-1 version N stays NEW`.
3. Запусти Serving. **Ожидаемо:** в следующем цикле `Delivered ... version N`, запись `SENT`.
4. Обрати внимание: после перезапуска Serving знает **только** версию N. Все предыдущие версии потеряны вместе
   с памятью процесса, а publisher досылает только новые записи. Для этого аккаунта это не страшно (у него есть
   последняя версия), но аккаунт, который в это время не менялся, останется в Serving без конфига
   (см. эксперимент 23).

### 29. Порядок изменений

1. Останови publisher. Сделай несколько изменений подряд для двух аккаунтов:
   `put 1`, `put 2`, `put 3` для `pub-1` и то же для `pub-2` (поменяй аккаунт в функции `put`).
2. Запусти publisher. **Ожидаемо:** в логе версии каждого аккаунта идут строго по возрастанию
   (`ORDER BY account_id, version`), Serving на последних версиях.
3. Что было бы без порядка: если бы `put 3` дошёл раньше `put 2`, Serving применил бы 3, а 2 проигнорировал
   бы как устаревшую. Итог тот же. Защита версиями работает, даже когда порядок нарушен: порядок здесь
   экономит лишние отправки, а корректность обеспечивают версии.

### 30. Откат

1. Publisher и Serving запущены. Посмотри версии: `SELECT version, payload FROM outbox WHERE account_id = 'pub-1' ORDER BY version;`.
2. Откати на одну из старых версий:
   `curl -X POST "localhost:8080/accounts/pub-1/settings/rollback?to_version=0"`.
3. **Ожидаемо:** ответ — **новая** версия (последняя + 1) со значениями версии 0, в outbox новая запись,
   через 3 секунды `serving` показывает её. Номера версий только растут: если бы откат возвращал номер 0,
   Serving отбросил бы его как устаревший.
4. Откат на несуществующую версию (`to_version=999`) → 404.
5. Откат на невалидную версию из эксперимента 26: API создаст новую версию, publisher пометит её `FAILED`,
   Serving останется на прежней. Откат не обходит валидацию.

---

## Шаг 5 второго этапа: marketplace-reporting

Подготовка: `docker compose up -d` (здесь), `mvn spring-boot:run` в marketplace-reporting (8082) и
в marketplace-serving (8081). Команды — для Git Bash.

```bash
J='Content-Type: application/json'
event()     { curl -s -X POST localhost:8082/events -H "$J" -d "{\"event_id\": \"$1\", \"account_id\": \"rep-1\", \"timestamp\": \"$2\", \"shown\": ${3:-true}}"; echo; }
aggregate() { curl -s -X POST "localhost:8082/admin/aggregate?hour=$1"; echo; }
report()    { curl -s localhost:8082/reports/rep-1; echo; }
```

Где смотреть: `report`, в psql `SELECT * FROM reporting.events WHERE account_id = 'rep-1';` и
`SELECT * FROM reporting.hourly_stats WHERE account_id = 'rep-1';`, в логе Reporting `Aggregated hour ...`.

Автотесты (marketplace-reporting): `ReportingApiTest`, `AggregationJobTest`; (marketplace-serving): `EventSendingTest`.
Эксперименты 31–35 и 37 проверены вручную (аналогичными запросами), 36 — нет.

### 31. События от Serving

1. `curl -X POST localhost:8081/ad-request -H "$J" -d '{"account_id": "rep-1", "domain": "news.com", "bid_price": 2}'` — три раза.
2. **Ожидаемо:** в `reporting.events` три строки с разными `event_id` и временем «сейчас» (UTC).

### 32. Агрегация за конкретный час и границы часа

```bash
event e1 2026-10-01T07:59:59Z          # предыдущий час
event e2 2026-10-01T08:00:00Z          # начало часа — входит
event e3 2026-10-01T08:30:00Z false
event e4 2026-10-01T09:00:00Z          # следующий час
aggregate 2026-10-01T08:00:00Z         # {"hour":"2026-10-01T08:00:00Z","accounts":1}
report                                 # [{"hour":"2026-10-01T08:00:00Z","requests":2,"shown":1}]
```

### 33. Дубли событий

1. `event e5 2026-10-01T08:40:00Z` → `{"stored":true}`, ещё раз то же → `{"stored":false}`.
2. `aggregate 2026-10-01T08:00:00Z`, `report` → `requests` выросло на 1, а не на 2.
3. Так выглядит повтор Serving после таймаута: Reporting событие уже сохранил, а Serving об этом не узнал.

### 34. Повторный запуск агрегации

`aggregate 2026-10-01T08:00:00Z` ещё 2–3 раза → `report` не меняется. Агрегаты пересчитываются из сырых событий
и заменяют прежние (см. README marketplace-reporting).

### 35. Позднее событие

1. `event late-1 2026-10-01T08:59:00Z` — событие за уже посчитанный час.
2. `report` — **не изменился**: событие сохранено в `events`, но агрегат за 08:00 старый.
3. `aggregate 2026-10-01T08:00:00Z` → `report` учёл событие.
4. Расписание считает только предыдущий час и к 08:00 само не вернётся. Как это решают — в README marketplace-reporting.

### 36. Запуск по расписанию (не проверялось)

1. Перезапусти Reporting с `AGGREGATION_CRON="0 * * * * *"` (каждую минуту;
   PowerShell: `$env:AGGREGATION_CRON="0 * * * * *"; mvn spring-boot:run`).
2. **Ожидаемо:** в начале каждой минуты в логе `Aggregated hour <предыдущий час>: N account(s)`. События из
   эксперимента 31 попадут в отчёт, когда закончится их час и пройдёт следующая минута.

### 37. Reporting недоступен

1. Останови Reporting, отправь `ad-request` (как в 31).
2. **Ожидаемо:** ответ приходит, как обычно, но заметно позже. В логе Serving: `Attempt 1/3 ... failed: ... Connection refused`,
   `Attempt 2/3`, `Attempt 3/3`, затем `Event ... lost: Reporting unavailable after 3 attempts`.
3. Вывод: для конфигов (outbox) недоступность получателя не приводит к потерям, для событий — приводит.
   Это сознательное упрощение (см. README marketplace-serving), вернёмся к нему в шаге 7.

---

## Шаг 6 второго этапа: BFF — N+1, лимиты, отказ сервиса, retry

Подготовка: `docker compose up -d` (здесь), запущены marketplace-api (8080) и marketplace-reporting (8082),
у нескольких аккаунтов есть настройки (`curl localhost:8080/accounts`). BFF запускается в каждом эксперименте
по-своему: `npm start` в marketplace-bff. Запросы удобно отправлять в GraphiQL: http://localhost:4000/graphql.

Где смотреть: ответ GraphQL и строки `[api] ...` / `[reporting] ...` в логе BFF — каждая строка это один
HTTP-запрос к соседу.

Автотесты (marketplace-bff): `n-plus-one.test.ts`, `limits.test.ts`, `reporting-failure.test.ts`, `retry.test.ts`.
Вручную проверены: 38, 39.1, 40 и 42.1. Остальное — прогноз: 39.3 — расчёт по измеренной стоимости одной копии
(6.5), 41 и 42.3 покрыты автотестами (`reporting-failure.test.ts`, `retry.test.ts`), но вживую не запускались.

### 38. N+1 и DataLoader

```graphql
query { accounts { id settings { floorPrice } } }
```

1. `USE_DATALOADER=false npm start` (PowerShell: `$env:USE_DATALOADER="false"; npm start`), выполни запрос.
   **Ожидаемо в логе:** `[api] GET /accounts`, затем по строке `[api] GET /accounts/<id>/settings` **на каждый
   аккаунт** (при проверке — 1 + 5).
2. `npm start` (по умолчанию DataLoader включён), тот же запрос.
   **Ожидаемо:** две строки — `GET /accounts` и `GET /settings?account_ids=...&account_ids=...`.
3. Создай ещё несколько аккаунтов (PUT в API) и повтори: без DataLoader строк становится больше, с ним — всегда 2.

### 39. Лимиты глубины и сложности

1. Слишком глубокий запрос (цикл `Settings.account` ↔ `Account.settings`):
   ```graphql
   query { accounts { settings { account { settings { account { settings { account { id } } } } } } } }
   ```
   **Ожидаемо:** `Syntax Error: Query depth limit of 6 exceeded, found 8.`, в логе BFF **ни одного** `[api]`:
   запрос отклонён до выполнения.
2. Убери один уровень `account { settings { ... } }` — глубина 6, запрос выполняется.
3. Широкий запрос: 200 копий `sN: settings(accountId: "x") { floorPrice currency blockedDomains }` (каждая стоит
   6.5). **Ожидаемо:** `Query Cost limit of 1000 exceeded, found 1300.`. 150 копий (975) пройдут.
4. Перезапусти с `MAX_DEPTH=10 MAX_COST=100000` — те же запросы выполняются, и широкий запрос делает 200
   запросов в API.

### 40. Отказ Reporting

```graphql
query { accounts { id settings { floorPrice } report { hour requests shown } } }
```

1. Reporting запущен → `report` — список (пустой, если агрегаций для аккаунта нет).
2. Останови Reporting, повтори запрос. **Ожидаемо:** `settings` на месте, `report: null`, в `errors` по одной
   ошибке `Reporting unavailable: fetch failed` на аккаунт с `path: ["accounts", N, "report"]`.
3. Обрати внимание на лог: `[reporting] GET /reports/<id>` для каждого аккаунта. У `report` свой N+1.

### 41. Таймаут

1. Запусти BFF с очень маленьким таймаутом Reporting: `REPORTING_TIMEOUT_MS=1 npm start`.
2. Запрос из 40 при работающем Reporting. **Ожидаемо:** `report: null`, ошибка про прерванный запрос
   (timeout / aborted), настройки на месте. Ответ приходит сразу, BFF не ждёт Reporting.
   Задержку в сети можно будет устроить и честнее, через Toxiproxy в шаге 7.

### 42. Retry мутации с Idempotency-Key

```graphql
mutation { addBlockedDomain(accountId: "<id>", domain: "via-bff.com") { blockedDomains } }
```

1. Обычный вызов: домен добавлен, в логе одна строка `[api] POST /accounts/<id>/blocked-domains`.
2. Повтор после сбоя вручную воспроизвести трудно: нужно, чтобы API обработал запрос, а ответ потерялся.
   Это покрыто автотестом `retry.test.ts` (первый ответ 503 после обработки → повтор с тем же ключом →
   домен один раз). В шаге 7 то же самое можно будет устроить через Toxiproxy между BFF и API.
3. Если API остановлен: в логе BFF три строки `addBlockedDomain attempt N/3 failed (fetch failed), key ...`
   с **одинаковым** ключом, затем ошибка `Marketplace API unavailable after 3 attempts`.

---

## Шаг 7 второго этапа: вся система в docker-compose и Toxiproxy

Подготовка: `docker compose up --build -d` (здесь), см. README. Команды — для Git Bash, сначала
`export MSYS_NO_PATHCONV=1` (иначе `/toxiproxy-cli` превратится в путь Windows).

```bash
export MSYS_NO_PATHCONV=1
J='Content-Type: application/json'
tox() { docker compose exec toxiproxy /toxiproxy-cli "$@"; }
gql() { curl -s localhost:4000/graphql -H "$J" -d "{\"query\": \"$1\"}"; echo; }
curl -X PUT localhost:8080/accounts/tox-1/settings -H "$J" -d '{"floor_price": 1.5, "currency": "USD", "blocked_domains": []}'
```

Все эксперименты 43–47 проверены вручную на compose.

### 43. Медленный Reporting: таймаут и частичный ответ

```bash
tox toxic add -t latency -a latency=3000 bff_to_reporting
time gql 'query { accounts { id settings { floorPrice } report { hour } } }'
tox toxic remove -n latency_downstream bff_to_reporting
```

**Ожидаемо:** ответ примерно через 1,3 с (таймаут BFF 1 с), а не через 3. `settings` на месте, `report: null`, в `errors` —
`Reporting unavailable: The operation was aborted due to timeout`.

### 44. Обрыв BFF → API

```bash
tox toggle bff_to_api
gql 'query { settings(accountId: \"tox-1\") { floorPrice } }'
tox toggle bff_to_api
```

**Ожидаемо:** `"settings": null` и ошибка `Unexpected error.` с `"code": "INTERNAL_SERVER_ERROR"`. Yoga
намеренно скрывает текст непредвиденных ошибок (здесь `fetch failed`), чтобы не раскрывать клиенту внутренности.
Явные ошибки резолверов (`Reporting unavailable`) видны, потому что созданы через `createGraphQLError`.
После второго `toggle` всё снова работает.

### 45. Повтор после потерянного ответа: BFF → API

```bash
tox toxic add -t latency -a latency=3000 bff_to_api
time gql 'mutation { addBlockedDomain(accountId: \"tox-1\", domain: \"slow.com\") { blockedDomains } }'
tox toxic remove -n latency_downstream bff_to_api
curl localhost:8080/accounts/tox-1/settings
docker compose logs bff | grep -E "blocked-domains|attempt"
```

**Ожидаемо:**
- через ~6,5 с ошибка `Marketplace API unavailable after 3 attempts: ... timeout`;
- в логе BFF три `POST /accounts/tox-1/blocked-domains` и три `attempt N/3 failed` с **одним и тем же ключом**;
- **но** в API `slow.com` есть, и ровно **один раз**. Все три запроса дошли до API (задерживался только ответ):
  первый добавил домен, второй и третий получили сохранённый ответ по `Idempotency-Key`.
- Выводов два: «ошибка» у клиента не значит «не выполнено», а без ключа домен добавился бы трижды.

### 46. Повторная доставка: Publisher → Serving

```bash
tox toxic add -t latency -a latency=3000 publisher_to_serving
curl -X PUT localhost:8080/accounts/tox-1/settings -H "$J" -d '{"floor_price": 2.5, "currency": "USD", "blocked_domains": []}'
# подожди ~20 секунд
docker compose exec postgres psql -U marketplace -c "SELECT version, status FROM outbox WHERE account_id = 'tox-1' ORDER BY version;"
tox toxic remove -n latency_downstream publisher_to_serving
docker compose logs serving | grep "tox-1" | sort | uniq -c
```

**Ожидаемо:**
- пока висит задержка, новая версия в outbox **`NEW`**, в логе publisher по кругу `Attempt 1/3, 2/3, 3/3`
  и `Serving unavailable, stopping this cycle`;
- после снятия задержки — `SENT`;
- в логе Serving: `Applied ... version N` **один раз** и много `Ignored ... version N is not newer` (при проверке — 7).
  Каждая попытка publisher дошла до Serving, применилась только первая.
- **Наблюдение:** вместо явного таймаута publisher пишет `Error while extracting response ... content type [application/octet-stream]`.
  Ошибка появляется только под задержкой, то есть это и есть недождавшийся ответ, но текст вводит в заблуждение.
  Хорошо бы, чтобы publisher различал «таймаут» и «непонятный ответ».

### 47. Serving → Reporting: потеря событий

```bash
docker compose stop reporting
curl -X POST localhost:8081/ad-request -H "$J" -d '{"account_id": "tox-1", "domain": "news.com", "bid_price": 9}'
docker compose logs serving | grep -E "Attempt|lost" | tail -4
docker compose start reporting
```

**Ожидаемо:** ответ на `ad-request` приходит, в логе Serving три `Attempt N/3 to send event ... failed`
и `Event ... lost`. В `reporting.events` этого события нет и не будет.

---

## Retry и дубли в системе

Где в системе возникают повторы, к чему они приводят и чем защищено каждое место.

| Место | Кто повторяет и когда | Что было бы без защиты | Защита | Где проверить |
|---|---|---|---|---|
| **BFF → API**, `addBlockedDomain` | BFF: таймаут, сетевая ошибка, 5xx — до 3 попыток | домен добавлен 2–3 раза | **Idempotency-Key**: один ключ на мутацию, API хранит ответ по ключу (`idempotency_keys`) в той же транзакции, что и изменение | 18, 42, 45; `retry.test.ts`, `IdempotencyApiTest` |
| **BFF → API**, `updateSettings` (PUT) | никто | — | PUT идемпотентен по смыслу: повтор с теми же данными даёт тот же результат, новой версии не создаёт. Но повтор после чужой записи затрёт её — для этого `version` (optimistic locking) | 19, 20; `OptimisticLockingApiTest` |
| **API → outbox → Publisher** | publisher: падение между отправкой и пометкой `SENT`, следующий цикл отправляет снова | — | **at-least-once**: outbox ничего не теряет, а дубли гасит получатель (строка ниже) | 25, 27; `CrashAfterSendTest` |
| **Publisher → Serving** | publisher: таймаут, ошибка — 3 попытки, затем весь цикл повторяется | старая или та же версия применилась бы поверх новой; при перестановке — откат настроек | **версия конфига**: Serving применяет только версию больше текущей (сравнение и запись атомарно), повтор → 200 `applied: false` | 21, 27, 28, 46; `ConfigStoreTest` |
| **Serving → Reporting** | Serving: таймаут, ошибка — 3 попытки с тем же `event_id` | событие посчитано дважды | **event_id**: `INSERT ... ON CONFLICT (event_id) DO NOTHING`. Но **потеря** возможна: после 3 неудач событие выброшено | 33, 37, 47; `EventSendingTest`, `ReportingApiTest` |
| **Агрегация в Reporting** | расписание, ручной запуск, несколько экземпляров | цифры за час удваиваются | **пересчёт с нуля и замена** (`ON CONFLICT DO UPDATE SET requests = EXCLUDED.requests`) | 34; `rerunOfAggregationDoesNotDoubleCounts` |
| **UI → BFF** | пользователь: двойной клик, повторная отправка формы | `addBlockedDomain` из UI добавил бы домен дважды | **нет защиты**: BFF создаёт новый ключ на каждую мутацию, два клика — две разные операции. Нужен ключ от клиента (UI создаёт его при открытии формы и передаёт в мутацию) | — |

Общие правила, которые видны по этой таблице:

1. **Повтор безопасен, только если получатель умеет узнавать повтор.** Узнаёт по идентификатору операции
   (Idempotency-Key, event_id) или по версии (Serving). Ключ создаёт отправитель **один раз на операцию** и не
   меняет его между попытками.
2. **Ошибка у вызывающего не значит, что операция не выполнена** (45, 46): при таймауте ответа сосед уже мог
   всё сделать. Поэтому «повторить» и «защититься от дубля» — всегда пара.
3. **Без надёжного хранилища между сервисами повторы ограничены**, и данные могут теряться (Serving → Reporting).
   С outbox повторять можно сколько угодно: запись лежит в базе, пока не доставлена.
4. **«Ровно один раз»** на практике — это at-least-once + идемпотентный получатель.
5. **Идемпотентность пересчёта** — заменять результат, а не прибавлять к нему: агрегат за час всегда считается
   заново из сырых данных.

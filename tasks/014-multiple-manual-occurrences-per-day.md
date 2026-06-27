# Multiple Manual Flight Occurrences Per Day

Status: Ready  
Bounded context: Flight Operations  
Service: aeroflow-core, aeroflow-web  
Change type: Domain invariant + Application/API addition + small Frontend

## Goal

Дать диспетчеру возможность запустить **новый операционный рейс по той же
карточке `FlightDefinition` в тот же день**, после того как предыдущий
`FlightOccurrence` дошёл до терминального состояния (`completed` или
`cancelled`).

Сейчас панель диспетчера definition-driven и обеспечивает occurrence через
идемпотентный `EnsureManualFlightOccurrence` с фиксированным
`sequenceNumber=1` (см. `012`). Это означает фактически **один запуск рейса в
день**: после завершения карточка «застревает», и второй прогон того же рейса
(вторая ротация, повторный рейс) сделать нельзя.

Доменная модель это уже допускает: бизнес-ключ occurrence —
`flightDefinitionId + operationalDate + source + sequenceNumber`, то есть
несколько occurrence за дату различаются `sequenceNumber`. Не хватает только
use case, который выделяет следующий `sequenceNumber`, и UX-входа в него.

## Не делаем

Не «оживляем» завершённый occurrence. Инвариант `завершённый или отменённый
occurrence нельзя менять` остаётся в силе, lifecycle линейный
(`scheduled → ... → completed`). Повторный прогон — это **новый** occurrence с
бóльшим `sequenceNumber`, а не возврат старого из `completed` в `boarding`.

## Ключевое решение: серверный инкремент + state-guard

`sequenceNumber` назначает **сервер**, а не клиент. Клиент шлёт только
`flightDefinitionId` и `operationalDate`.

Защита от двойного нажатия строится не на синтетическом ключе, а на **доменном
состоянии**:

```text
Новый запуск разрешён, только если последний occurrence карточки за дату
терминален (completed/cancelled). Иначе — 409.
```

Это автоматически даёт защиту от двойного клика, без idempotency key:

* клик 1 → создаётся `sequenceNumber = max + 1`, статус `scheduled` (не терминальный);
* клик 2 → последний occurrence не терминален → `409 Conflict`.

Так дедупликацию обеспечивает само состояние, а правило совпадает с интуицией
диспетчера «новый прогон только после завершения предыдущего».

Рассмотренная альтернатива — клиентский `idempotencyKey` — отклонена для v1: она
перекладывает дисциплину на фронт и нужна только для сценариев параллельных
легитимных запусков, которых в ручном потоке пока нет. Оставлена как возможный
follow-up.

## Architectural Classification

Затрагивает только `Flight Operations`.

Затрагиваемые агрегаты:

* `FlightOccurrence` — новый инвариант на создание следующего manual-прогона;
* `FlightDefinition` — только как источник карточки (без изменений).

`Announcements`, `Audio Catalog`, `Localization`, `aeroflow-playback`,
`aeroflow-agent` не затрагиваются. Запуск объявлений по новому occurrence идёт
через уже существующие action-эндпоинты (`011`/`012`) без изменений.

## Backend Scope

### Новый use case: StartNextManualFlightOccurrence

```http
POST /api/v1/flight-occurrences:start-next-manual
```

```json
{ "flightDefinitionId": "uuid", "operationalDate": "2026-06-27" }
```

Поведение (одна локальная транзакция, последний occurrence грузится на запись):

1. Найти последний manual-occurrence по `flightDefinitionId + operationalDate +
   source=manual`, упорядоченный по `sequenceNumber` убыванию.
2. Если его нет — это путь «первого запуска», обслуживаемый
   `EnsureManualFlightOccurrence`; `start-next-manual` возвращает `409` с
   причиной «нет предыдущего запуска» (или `404`, уточнить при реализации).
   Сознательно **не** дублируем здесь создание seq=1, чтобы у каждого входа была
   одна ответственность.
3. Если последний occurrence **не терминален** (`scheduled`, `check_in_open`,
   `check_in_closed`, `boarding`, `arrival_announced`) → `409 Conflict`,
   ничего не создаётся.
4. Иначе создать occurrence с `sequenceNumber = max + 1`, статус `scheduled`,
   `source=manual`, direction наследуется от `FlightDefinition`.

Response: `201` с `FlightOccurrenceResult` (новый occurrence).

### Разграничение с EnsureManualFlightOccurrence

* `EnsureManualFlightOccurrence` (seq=1, идемпотентный) — **первый** прогон
  карточки за дату; остаётся как есть, панель использует его без изменений;
* `StartNextManualFlightOccurrence` — **последующие** прогоны, серверный
  инкремент, guard по терминальности.

### Конкурентность

`FlightOccurrence` получает optimistic/pessimistic lock на чтение последнего
прогона при создании следующего, чтобы две параллельные команды не выделили
один и тот же `sequenceNumber`. Уникальный индекс по бизнес-ключу остаётся
последней защитой: проигравшая гонку команда получает конфликт, а не дубликат.

### Backend Requirements

* `start-next-manual` создаёт occurrence только когда предыдущий терминален;
* при не-терминальном предыдущем возвращает `409` и ничего не персистит;
* при отсутствии предыдущего возвращает `409/404` (не создаёт seq=1);
* следующий `sequenceNumber` назначает сервер, клиентское значение не принимается;
* параллельные вызовы не создают дубликат бизнес-ключа;
* покрыть application/API-тестами: happy path (seq 2 после completed),
  отказ при не-терминальном, отказ при отсутствии предыдущего, идемпотентность
  guard при двойном вызове.

## Frontend Scope

Расширить feature `dispatcher`.

* На карточке, чей сегодняшний occurrence в статусе `completed`/`cancelled`,
  показать действие **«Новый запуск»**;
* «Новый запуск» вызывает `POST /flight-occurrences:start-next-manual`
  (`flightDefinitionId`, `operationalDate`), получает новый occurrence в статусе
  `scheduled`;
* после успеха инвалидировать query-ключи доски и dispatcher-occurrences за дату,
  чтобы карточка снова появилась под фильтрами первичных действий («Начало
  регистрации» / «Прибытие»);
* двойной клик: primary-кнопка дизейблится на время мутации (как в `012`); даже
  если запрос уйдёт дважды, второй вернёт `409` (guard), который показывается как
  conflict без потери UX;
* `409` «предыдущий запуск ещё не завершён» / «нет предыдущего» показывается
  рядом с действием человекочитаемо, без internal exception names.

Доска по-прежнему строится слиянием `flight-definitions` + dispatcher
`flight-occurrences`. Наложение статуса теперь учитывает, что у карточки за дату
может быть несколько occurrence — для отображения статуса берётся **последний по
`sequenceNumber`**.

## Out of Scope

* `FlightSchedule` и автоматическое создание occurrence по расписанию;
* идентичность occurrence по времени вылета (см. Open decision ниже);
* клиентский/серверный `idempotencyKey` на запуск;
* отмена occurrence из панели (`cancelled` как явное действие диспетчера);
* история/список всех прогонов карточки за день в UI (показываем только статус
  последнего);
* ролевое разграничение доступа.

## Acceptance Criteria

Backend:

* `POST /api/v1/flight-occurrences:start-next-manual` создаёт occurrence с
  `sequenceNumber = max + 1`, когда последний прогон карточки за дату терминален;
* при не-терминальном последнем прогоне возвращает `409`, occurrence не создаётся;
* при отсутствии предыдущего прогона возвращает `409/404`, occurrence не создаётся;
* `sequenceNumber` назначается сервером; клиентское значение игнорируется/запрещено;
* параллельные вызовы не приводят к дублю бизнес-ключа;
* добавлены application/API-тесты на все ветки.

Frontend:

* на завершённой/отменённой карточке доступно «Новый запуск»;
* «Новый запуск» создаёт следующий прогон, карточка возвращается под первичные
  действия, доска обновляется;
* статус карточки берётся из последнего occurrence за дату;
* двойной клик не создаёт второй прогон (кнопка дизейблится, `409` обрабатывается);
* конфликтные ответы показываются рядом с действием без internal names.

## Verification

Backend:

```bash
php bin/phpunit tests/Unit tests/Application
docker compose exec -T aeroflow_core_app php bin/phpunit tests/Functional
```

Frontend:

```bash
npm test
npm run lint
npm run build
```

Manual QA:

```text
1. Войти, открыть /dispatcher, провести рейс через полный цикл до completed
   (открыть регистрацию → закрыть → посадка для departure; прибытие для arrival).
2. На завершённой карточке нажать «Новый запуск»: создаётся новый прогон, карточка
   снова доступна под «Начало регистрации» / «Прибытие».
3. Сразу второй раз нажать «Новый запуск» (предыдущий прогон ещё scheduled):
   получить 409, дубликат не создан.
4. Провести второй прогон тоже до completed и убедиться, что можно открыть третий.
5. Попробовать «Новый запуск» на карточке без единого прогона за день: 409/404.
```

## Documentation Updates

До реализации (Draft → Ready):

* `domain-model.md`, раздел `FlightOccurrence`: добавить инвариант «следующий
  manual-occurrence за дату создаётся только когда последний прогон терминален»
  и описать `StartNextManualFlightOccurrence` (серверный инкремент
  `sequenceNumber`) рядом с `Create`/`EnsureManualFlightOccurrence`;
* `domain-model.md`, «Открытые решения»: зафиксировать, что для ручного потока
  допускается несколько occurrence в день; `sequenceNumber` остаётся
  синтетическим ключом, а при появлении `FlightSchedule` идентичность occurrence
  пересматривается в пользу времени вылета (operationalDateTime).

После реализации:

* `aeroflow-web/architecture.md`: дополнить описание dispatcher-среза
  возможностью повторного прогона карточки в тот же день («Новый запуск» после
  терминального occurrence, статус берётся из последнего прогона).

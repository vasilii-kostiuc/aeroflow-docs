# Flight Occurrences Core

Status: Done  
Bounded context: Flight Operations, Announcements  
Service: aeroflow-core  
Change type: Domain, Application, Infrastructure, API, Migration

## Goal

Ввести `FlightOccurrence` в `aeroflow-core` как конкретный operational рейс,
который хранит состояние регистрации, посадки и прибытия.

Цель — сделать единый механизм запуска рейсовых объявлений:

```text
ручной диспетчерский запуск
  -> FlightOccurrence
  -> Announcement
  -> playback request

будущий запуск по расписанию
  -> FlightOccurrence
  -> Announcement
  -> playback request
```

`FlightDefinition` остаётся постоянной карточкой/шаблоном рейса. `FlightOccurrence`
становится фактом конкретного вылета или прилёта в операционный день.

## Architectural Classification

Задача переводит `FlightOccurrence` из будущего расширения в реализуемую часть
`Flight Operations`.

Затрагиваемые агрегаты:

* `FlightDefinition` — источник постоянных данных рейса;
* `FlightOccurrence` — новый aggregate root для операционного состояния;
* `Announcement` — должен хранить `flightOccurrenceId` и immutable occurrence
  snapshot при создании рейсового объявления.

Изменение доменное и инфраструктурное.

Новые domain events:

* `FlightOccurrenceCreated`;
* `CheckInOpened`;
* `CheckInContinued`;
* `CheckInClosed`;
* `BoardingStarted`;
* `ArrivalAnnounced`;
* `FlightOccurrenceCompleted`;
* `FlightOccurrenceCancelled`, если отмена включается в реализацию.

`aeroflow-playback` и `aeroflow-agent` не затрагиваются. Playback продолжает
работать только с `PlaybackJob` и не знает о `FlightOccurrence`.

## Domain Model

### FlightOccurrence

`FlightOccurrence` представляет конкретный operational рейс.

Состояние:

* `id: UUID`;
* `flightDefinitionId: UUID`;
* `source: manual|schedule`;
* `direction: departure|arrival`;
* `operationalDate`;
* `flightNumberSnapshot`;
* `originAirportCodeSnapshot`;
* `destinationAirportCodeSnapshot`;
* `status`;
* `checkInCounterSnapshots`;
* `gateSnapshot`;
* `lastAnnouncementId`;
* `createdAt`;
* `updatedAt`;
* `completedAt`;
* `cancelledAt`.

Snapshots нужны, чтобы изменение `FlightDefinition`, справочника стоек или
выходов после запуска не переписало историю occurrence.

### Status Lifecycle

Для вылетов:

```text
scheduled
  -> check_in_open
  -> check_in_closed
  -> boarding
  -> completed
```

Для прилётов:

```text
scheduled
  -> arrival_announced
  -> completed
```

Отмена:

```text
scheduled|check_in_open|check_in_closed|boarding
  -> cancelled
```

Cancellation можно оставить out of scope, если для dispatcher panel она пока не
нужна.

### Invariants

* occurrence создаётся только для активного `FlightDefinition`;
* `direction` occurrence совпадает с direction `FlightDefinition`;
* `operationalDate` обязателен;
* для `source=manual` допускается отсутствие scheduled time;
* для `source=schedule` в будущем будет требоваться ссылка на schedule или
  scheduled time;
* нельзя открыть регистрацию дважды;
* нельзя продолжить регистрацию до открытия;
* нельзя закрыть регистрацию до открытия;
* нельзя начать посадку до закрытия регистрации;
* нельзя объявить прибытие для departure occurrence;
* нельзя открыть регистрацию или посадку для arrival occurrence;
* завершённый или отменённый occurrence нельзя менять;
* выбранные стойки и выход должны быть активными на момент перехода и
  сохраняются как snapshot.

### Business Key

На первом этапе:

```text
flightDefinitionId + operationalDate + source + sequenceNumber
```

`sequenceNumber` по умолчанию равен `1`.

Это позволяет позже поддержать несколько occurrence одного `FlightDefinition` в
один день без смены модели. Если пока UI не создаёт несколько рейсов одного
номера за день, API может скрывать `sequenceNumber` и использовать `1`.

## Application Flow

### Create Manual Occurrence

Use case:

```text
CreateManualFlightOccurrence
```

Input:

```json
{
  "flightDefinitionId": "uuid",
  "operationalDate": "2026-06-25",
  "sequenceNumber": 1
}
```

Handler:

1. загружает активный `FlightDefinition`;
2. проверяет уникальность business key;
3. создаёт `FlightOccurrence` со статусом `scheduled`;
4. сохраняет occurrence;
5. публикует `FlightOccurrenceCreated`.

Для dispatcher UI можно добавить idempotent use case:

```text
EnsureManualFlightOccurrence
```

Он возвращает существующий occurrence для business key или создаёт новый.

### Launch Announcement For Occurrence

Запуск объявления — это **действие над `FlightOccurrence`**, общее для ручного
диспетчера и будущего расписания. Оркестрация принадлежит
`Flight Operations\Application` (per-action use cases под action-эндпоинты):

```text
OpenCheckIn | CloseCheckIn | StartBoarding | AnnounceArrival
```

Input (для каждого действия; набор полей зависит от типа):

```json
{
  "flightOccurrenceId": "uuid",
  "languages": ["ro-MD", "ru"],
  "checkInCounterIds": ["uuid-1", "uuid-3"],
  "gateId": null,
  "idempotencyKey": "optional-client-key"
}
```

Handler:

1. загружает `FlightOccurrence` с блокировкой на запись;
2. агрегат `FlightOccurrence` — единственный авторитет допустимости перехода
   (direction + status); матрица перехода не дублируется в `Announcements`;
3. для регистрации получает активные стойки, для посадки — активный gate;
4. создаёт `Announcement` через consumer-owned port, принадлежащий
   `Flight Operations` (см. Context Boundary); порт возвращает `announcementId`;
5. применяет переход состояния occurrence, сохраняет snapshots стоек/выхода и
   записывает `lastAnnouncementId`;
6. сохраняет оба изменения в **одной локальной транзакции**;
7. публикует событие перехода occurrence и `AnnouncementCreated` после коммита.

Готовность объявления — **предусловие** перехода: если шаблон не резолвится в
`AudioSequence` (нет config/asset/prompt), переход отклоняется и **ничего не
персистится** — ни `Announcement`, ни смена статуса occurrence.

Нельзя сохранить `Announcement`, если переход occurrence не применился, и нельзя
применить переход без созданного `Announcement`.

Механизм атомарности — `doctrine_transaction` middleware на `command.bus`.
Domain events буферизуются внутри command transaction scope и публикуются после
успешного коммита.

### Context Boundary

`Flight Operations` и `Announcements` не должны импортировать агрегаты друг
друга напрямую.

Целевой вариант (обязательный):

* orchestration use case находится в `Flight Operations\Application` и работает с
  `FlightOccurrence` как владельцем состояния;
* он вызывает consumer-owned port `FlightOperations\Application\Port\Announcements`
  для создания `Announcement`;
* реализация порта находится в `Announcements\Infrastructure\Integration`
  (поставщик не зависит от порта потребителя);
* порт принимает occurrence snapshot и параметры запуска, возвращает только
  `announcementId` и нужные scalar/snapshot данные.

`Announcements` **не** обращается к write-side `FlightOccurrence`. Текущую
инвертированную реализацию нужно убрать: из `Announcements` уходят мутация и
проверка перехода occurrence (`FlightOccurrenceLookupInterface::assertCanLaunch`
и `recordAnnouncementLaunch`, вызывающие `openCheckIn/closeCheckIn/startBoarding/
announceArrival` на чужом агрегате). За `Announcements` остаётся сборка
`Announcement` и read-only справочные порты.

## HTTP API

### Occurrences

```http
POST /api/v1/flight-occurrences
GET  /api/v1/flight-occurrences
GET  /api/v1/flight-occurrences/{id}
```

Create payload:

```json
{
  "flightDefinitionId": "uuid",
  "operationalDate": "2026-06-25",
  "sequenceNumber": 1,
  "source": "manual"
}
```

List filters:

```text
operationalDate
flightDefinitionId
direction
status
source
```

### Dispatcher Read Model

```http
GET /api/v1/dispatcher/flight-occurrences
```

Filters:

```text
operationalDate
announcementType
direction
includeUnavailable
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "flightDefinitionId": "uuid",
      "flightNumber": "5F 123",
      "direction": "departure",
      "airportCode": "IST",
      "airportName": "Istanbul",
      "operationalDate": "2026-06-25",
      "status": "scheduled",
      "eligible": true,
      "unavailableReason": null,
      "availableLanguages": ["ro-MD", "ru"]
    }
  ]
}
```

### Announcement Launch

Запуск моделируется как action-эндпоинты над `FlightOccurrence` (глагол = переход
lifecycle, объявление — результат действия):

```http
POST /api/v1/flight-occurrences/{id}/check-in:open
POST /api/v1/flight-occurrences/{id}/check-in:close
POST /api/v1/flight-occurrences/{id}/boarding
POST /api/v1/flight-occurrences/{id}/arrival
```

Payload (поля зависят от действия):

```json
{
  "languages": ["ro-MD"],
  "checkInCounterIds": ["uuid-1"],
  "gateId": null,
  "idempotencyKey": "optional-client-key"
}
```

Response:

```json
{
  "success": true,
  "data": {
    "occurrence": { "id": "uuid", "status": "check_in_open" },
    "announcementId": "uuid"
  }
}
```

Плоский `POST /api/v1/announcements` для рейсовых типов деприкейтится и убирается.
`flightDefinitionId`-путь создания рейсового объявления больше не поддерживается.
Если потребуется обратная совместимость, она должна быть явно отделена от нового
occurrence-based потока диспетчера.

## Readiness Rules

Dispatcher read model marks occurrence as eligible for an announcement type only
if:

* occurrence status allows the transition;
* occurrence direction is compatible with announcement type;
* `FlightDefinition` is active;
* active `FlightAnnouncementConfig` exists for the type;
* requested or default languages have enabled variants;
* template can be resolved into `AudioSequence`;
* required dynamic values are present or can be selected by dispatcher.

Readiness calculation belongs to backend. Frontend should not reconstruct domain
rules from raw configs.

## Out of Scope

* `FlightSchedule` CRUD and recurrence rules;
* automatic creation of occurrences from schedule;
* delay, cancellation and actual time management, unless cancellation is needed
  for an invariant during implementation;
* conflict detection for gate or check-in counter assignments;
* realtime updates;
* playback queue management;
* web dispatcher UI.

## Documentation Updates

После реализации обновить source-of-truth документы:

* `aeroflow-docs/domain-model.md` — перевести `FlightOccurrence` из будущего
  расширения в реализованную модель и описать lifecycle;
* `aeroflow-docs/architecture.md` — обновить ручной сценарий на
  `FlightDefinition -> FlightOccurrence -> Announcement`;
* `aeroflow-docs/README.md` — уточнить, что текущий dispatcher flow работает с
  operational рейсами, созданными вручную или будущим расписанием.

## Acceptance Criteria

Domain/Application:

* `FlightOccurrence` aggregate exists in `Flight Operations`;
* manual occurrence can be created from active `FlightDefinition`;
* occurrence lifecycle enforces valid transitions;
* departure and arrival lifecycles reject incompatible announcement types;
* selected check-in counters and gate are stored as snapshots;
* completed/cancelled occurrence cannot be mutated;
* announcement launch is orchestrated in `Flight Operations\Application`, not in
  `Announcements`;
* `Announcements` no longer mutates `FlightOccurrence` (`assertCanLaunch` /
  `recordAnnouncementLaunch` removed); transition authority is the aggregate;
* occurrence transition and `Announcement` creation happen atomically in one local
  transaction;
* announcement readiness is a precondition: an unpreparable template rejects the
  transition and persists nothing (no `Announcement`, no status change);
* announcement creation goes through a `Flight Operations`-owned port to
  `Announcements` returning only `announcementId` and scalars;
* old `flightDefinitionId` creation path for flight announcements is removed.

API:

* `POST /api/v1/flight-occurrences` creates manual occurrence;
* `GET /api/v1/flight-occurrences` lists occurrences with filters;
* `GET /api/v1/dispatcher/flight-occurrences` returns action-aware read model;
* `POST /api/v1/flight-occurrences/{id}/check-in:open|check-in:close|boarding|arrival`
  advance occurrence and return the new status plus `announcementId`;
* invalid transition returns 409/422 and does not create `Announcement`;
* unpreparable announcement returns 409/422 and changes nothing;
* duplicate create for same business key returns existing occurrence or a clear
  conflict, depending on chosen use case.

Tests:

* domain tests cover lifecycle transitions and rejected transitions;
* application tests cover manual creation and announcement launch;
* persistence tests cover snapshots and status changes;
* API tests cover occurrence creation, dispatcher read model and launch payload.

## Verification

```bash
php bin/phpunit
```

Verified:

```text
php bin/phpunit tests/Unit tests/Application
docker compose exec -T aeroflow_core_app php bin/phpunit tests/Functional
```

Manual QA:

```text
1. Create manual occurrence for an active departure FlightDefinition.
2. POST .../check-in:open with selected counters.
3. Verify occurrence status is check_in_open and the returned Announcement
   references the occurrence.
4. POST .../check-in:close on a scheduled occurrence (before open) on another
   occurrence.
5. Verify API rejects the transition and no Announcement is created.
6. POST .../check-in:open against a type without a ready template/config.
7. Verify API rejects it and persists nothing (no status change, no Announcement).
8. POST .../boarding after check-in:close with selected gate.
9. Verify dispatcher read model moves occurrence between action filters.
```

# Manage Flight Definitions

Status: Done  
Bounded context: Flight Operations  
Service: aeroflow-core  
Change type: Domain, Application, Infrastructure, API

## Goal

Администратор управляет каталогом рейсов, из которого диспетчер выбирает
конкретную карточку и запускает нужное объявление вручную.

`FlightDefinition` является многократно используемым описанием рейса. Новая запись
не создаётся для каждого фактического вылета.

## Scope

* создать `FlightDefinition`;
* создать Value Objects `FlightNumber` и `AirportCode`;
* создать enum `FlightDirection`;
* реализовать создание, получение, список и редактирование карточек;
* реализовать активацию и деактивацию карточки;
* хранить каталог в PostgreSQL;
* предоставить HTTP API;
* покрыть функциональность тестами.

## Out of Scope

* расписание;
* создание записи на каждый фактический вылет;
* состояние регистрации и посадки;
* Gate и CheckInCounter;
* Announcement и аудиофайлы;
* aeroflow-playback;
* frontend.

## Domain Model

`FlightDefinition` — aggregate root постоянной карточки рейса.

Состояние:

* `id` — UUID, создаётся доменом;
* `flightNumber` — `FlightNumber`;
* `direction` — `FlightDirection`;
* `originAirportCode` — аэропорт отправления;
* `destinationAirportCode` — аэропорт назначения;
* `active` — доступна ли карточка диспетчеру;
* `createdAt`;
* `updatedAt`.

Публичное поведение:

```php
FlightDefinition::create(
    FlightNumber $flightNumber,
    FlightDirection $direction,
    AirportCode $originAirportCode,
    AirportCode $destinationAirportCode,
): FlightDefinition

$flightDefinition->updateDetails(
    FlightNumber $flightNumber,
    FlightDirection $direction,
    AirportCode $originAirportCode,
    AirportCode $destinationAirportCode,
);
$flightDefinition->activate();
$flightDefinition->deactivate();
```

Редактирование проходит через единый метод агрегата, а не через публичные setters.
Если значения не изменились, событие обновления не публикуется.

## Business Rules

* UUID создаётся до публикации domain event.
* Номер рейса нормализуется в верхний регистр.
* Формат номера: код перевозчика из 2-3 латинских букв или цифр и номер из 1-4
  цифр, например `5F123`, `WZZ42` или `AFL100`.
* Код аэропорта содержит ровно 3 латинские буквы и нормализуется в верхний регистр.
* Аэропорты отправления и назначения должны различаться.
* Новая карточка активна по умолчанию.
* Деактивация идемпотентна.
* Удаление карточки не поддерживается: используем деактивацию, чтобы не ломать
  историю объявлений.
* Комбинация `flightNumber + direction + originAirportCode +
  destinationAirportCode` уникальна.

## Future Extension

Текущий ручной сценарий:

```text
FlightDefinition
    -> dispatcher presses a button
    -> Announcement
```

Будущий сценарий с расписанием:

```text
FlightDefinition
    -> FlightSchedule
    -> FlightOccurrence
    -> Announcement
```

* `FlightSchedule` задаст правило повторяемости и времени.
* `FlightOccurrence` представит конкретный вылет и его состояние.
* Добавление этих моделей не должно менять идентичность `FlightDefinition`.

## Domain Events

* `FlightDefinitionCreated`;
* `FlightDefinitionUpdated`;
* `FlightDefinitionActivated`;
* `FlightDefinitionDeactivated`.

События являются внутренними для `Flight Operations` и пока не отправляются
в другие сервисы.

## Application Use Cases

### CreateFlightDefinition

1. Создать и провалидировать Value Objects.
2. Проверить отсутствие дубликата бизнес-ключа.
3. Создать и сохранить `FlightDefinition`.
4. Опубликовать `FlightDefinitionCreated`.
5. Вернуть `FlightDefinitionResult`.

### GetFlightDefinition

Загрузить карточку по UUID или бросить `FlightDefinitionNotFoundException`.

### ListFlightDefinitions

Возвращать список с фильтрами:

* `active`;
* `direction`;
* поисковая строка по номеру рейса.
* `page`, начиная с `1`;
* `limit` от `1` до `100`, по умолчанию `20`.

Сортировка по умолчанию: `flightNumber ASC`.

Ответ содержит `items` и pagination metadata:

```json
{
  "items": [],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 0,
    "totalPages": 0
  }
}
```

### UpdateFlightDefinition

1. Загрузить карточку.
2. Провалидировать новые значения.
3. Проверить отсутствие другого дубликата.
4. Изменить карточку через поведение агрегата.
5. Сохранить и опубликовать `FlightDefinitionUpdated`.

### ActivateFlightDefinition / DeactivateFlightDefinition

Изменить доступность карточки диспетчеру. Повтор команды не должен менять
результат или публиковать повторное событие.

## API

```text
POST   /api/v1/flight-definitions
GET    /api/v1/flight-definitions
GET    /api/v1/flight-definitions/{id}
PUT    /api/v1/flight-definitions/{id}
POST   /api/v1/flight-definitions/{id}/activate
POST   /api/v1/flight-definitions/{id}/deactivate
```

Create/update payload:

```json
{
  "flightNumber": "5F123",
  "direction": "departure",
  "originAirportCode": "KIV",
  "destinationAirportCode": "FCO"
}
```

Response data:

```json
{
  "id": "uuid",
  "flightNumber": "5F123",
  "direction": "departure",
  "originAirportCode": "KIV",
  "destinationAirportCode": "FCO",
  "active": true,
  "createdAt": "RFC3339",
  "updatedAt": "RFC3339"
}
```

HTTP errors:

* `404` — карточка не найдена;
* `409` — дубликат бизнес-ключа;
* `422` — некорректный UUID, payload или доменное значение.

Все endpoints первого этапа доступны любому авторизованному пользователю.
Разделение прав администратора и диспетчера будет отдельной задачей User Access.

## Persistence

Таблица `flight_definition`:

```text
id                   UUID primary key
flight_number        VARCHAR(7) not null
direction            VARCHAR(16) not null
origin_airport_code  VARCHAR(3) not null
destination_airport_code VARCHAR(3) not null
active               BOOLEAN not null
created_at           TIMESTAMP WITHOUT TIME ZONE not null
updated_at           TIMESTAMP WITHOUT TIME ZONE not null
```

Уникальный индекс:

```text
flight_number + direction + origin_airport_code + destination_airport_code
```

Repository contract поддерживает:

* сохранение и поиск по UUID;
* проверку бизнес-ключа с исключением текущего UUID при редактировании;
* получение фильтрованного списка.

Для списка допустим отдельный query service, если repository начинает смешивать
загрузку агрегата и API-specific чтение.

## Code Organization

```text
src/FlightOperations/
  Domain/
    Entity/FlightDefinition.php
    Enum/FlightDirection.php
    Event/
    Exception/
    Repository/FlightDefinitionRepositoryInterface.php
    ValueObject/AirportCode.php
    ValueObject/FlightNumber.php
  Application/
    CreateFlightDefinition/
    GetFlightDefinition/
    ListFlightDefinitions/
    UpdateFlightDefinition/
    ActivateFlightDefinition/
    DeactivateFlightDefinition/
  Infrastructure/
    Persistence/Doctrine/FlightDefinitionRepository.php
  Api/
    Controller/
    Request/
```

## Test Scenarios

### Domain

* карточка создаётся активной и с UUID;
* номер и коды аэропортов нормализуются;
* некорректные значения отклоняются;
* одинаковые аэропорты отклоняются;
* update изменяет только разрешённые значения;
* activate/deactivate идемпотентны;
* публикуются корректные domain events.

### Application

* create сохраняет карточку и публикует событие;
* create/update отклоняют дубликат;
* get возвращает карточку или `not found`;
* list применяет фильтры;
* activation handlers сохраняют только реальное изменение состояния.

### Integration

* Doctrine repository сохраняет и загружает карточку;
* уникальный индекс защищает бизнес-ключ;
* фильтры списка работают на PostgreSQL;
* Doctrine schema validation проходит.

### Functional

* создание возвращает `201`;
* получение и список возвращают `200`;
* редактирование возвращает нормализованные значения;
* duplicate возвращает `409`;
* invalid payload возвращает `422`;
* неизвестный UUID возвращает `404`;
* deactivate скрывает карточку из списка `active=true`;
* повторные activate/deactivate остаются успешными и идемпотентными.

## Definition of Done

* миграция применяется на PostgreSQL;
* OpenAPI описывает endpoints;
* domain, application, integration и functional тесты проходят;
* `composer qa` проходит;
* controller и repository не содержат бизнес-правил;
* `domain-model.md`, `architecture.md` и `README.md` соответствуют реализации.

## Decisions

* `FlightDefinition` переиспользуется для многих фактических вылетов.
* Расписание и конкретные вылеты не реализуются на первом этапе.
* Исторические связи защищаются деактивацией вместо физического удаления.
* API явно использует термин `flight-definitions`, чтобы не смешивать карточку
  каталога с будущим конкретным вылетом.
* Список использует `page` и `limit`, а не raw offset.
* Optimistic locking не добавляется до появления реальных конфликтов редактирования.

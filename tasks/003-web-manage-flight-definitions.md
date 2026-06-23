# Web Manage Flight Definitions

Status: Done  
Bounded context: Flight Operations  
Service: aeroflow-web  
Change type: Presentation, Integration, Client Infrastructure

## Goal

Администратор управляет постоянным справочником карточек рейсов через существующий
API `aeroflow-core`. Диспетчер не управляет справочником: активные карточки будут
использоваться в его отдельных рабочих сценариях запуска объявлений.

`FlightDefinition` остаётся aggregate root в `aeroflow-core`. Frontend отображает
DTO и вызывает существующие use cases, но не дублирует доменную модель и не
публикует domain events.

## Scope

* список карточек с состояниями загрузки, ошибки и пустого результата;
* поиск по номеру рейса;
* фильтры направления и активности;
* пагинация с хранением состояния в URL;
* создание и редактирование карточки;
* активация и подтверждаемая деактивация;
* клиентская проверка стабильных форматов;
* отображение validation, conflict, not found и transport errors;
* типизированная интеграция с API;
* component и integration тесты.

## Out of Scope

* изменение `aeroflow-core`;
* удаление карточек;
* техническое ограничение endpoints и маршрута по роли администратора;
* `FlightSchedule` и `FlightOccurrence`;
* объявления, аудиофайлы и playback.

## Architectural Classification

Изменение относится к `Flight Operations` и является frontend-срезом:

```text
FlightDefinitionsPage
  -> flight-definitions feature
  -> authenticated API client
  -> existing FlightDefinition use cases in aeroflow-core
```

Затрагивается существующий агрегат `FlightDefinition` только через HTTP API.
Существующие события `FlightDefinitionCreated`, `FlightDefinitionUpdated`,
`FlightDefinitionActivated` и `FlightDefinitionDeactivated` публикуются core.

До реализации role-based access API и экран технически доступны любому
авторизованному пользователю, но продуктово являются административной функцией.

## API Contract

```text
POST /api/v1/flight-definitions
GET  /api/v1/flight-definitions
PUT  /api/v1/flight-definitions/{id}
POST /api/v1/flight-definitions/{id}/activate
POST /api/v1/flight-definitions/{id}/deactivate
```

Фильтры списка: `search`, `direction`, `active`, `page`, `limit`.

Create/update payload:

```json
{
  "flightNumber": "5F123",
  "direction": "departure",
  "originAirportCode": "KIV",
  "destinationAirportCode": "FCO"
}
```

## UI Decisions

* поиск, фильтры и страница хранятся в query parameters;
* изменение поиска или фильтра возвращает пользователя на первую страницу;
* создание и редактирование открываются в modal;
* деактивация требует подтверждения, активация выполняется сразу;
* физическое удаление отсутствует;
* `409` объясняется как существующий дубликат;
* `422` привязывается к полям формы, когда backend вернул имя поля;
* `404` при mutation закрывает устаревшее действие и обновляет список;
* network и `5xx` позволяют повторить запрос.

## Test Scenarios

* фильтры корректно преобразуются в query string;
* отображаются loading, empty, error и populated states;
* форма проверяет обязательность и стабильные доменные форматы;
* создание и редактирование обновляют список;
* `409` и server violations отображаются пользователю;
* деактивация подтверждается, активация доступна для неактивной карточки;
* фильтры и пагинация синхронизированы с URL.

## Definition of Done

* страница-заглушка заменена рабочим каталогом;
* прямые HTTP-вызовы отсутствуют в UI-компонентах;
* server state не хранится в Zustand;
* `lint`, `typecheck`, `test:run` и `build` проходят;
* backend-контракт и доменная модель не изменены.

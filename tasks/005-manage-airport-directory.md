# Manage Airport Directory

Status: Done  
Bounded context: Flight Operations  
Services: aeroflow-core, aeroflow-web  
Change type: Domain, Application, Infrastructure, API, Presentation

## Goal

Администратор управляет справочником аэропортов, а интерфейсы рейсов показывают
читаемый маршрут в формате `Кишинёв (RMO) → Рим (FCO)`.

## Domain Decisions

* `Airport` — отдельный справочный aggregate root;
* состояние: UUID, неизменяемый IATA-код, название аэропорта, город, ISO-код
  страны, активность и timestamps;
* `FlightDefinition` продолжает хранить `AirportCode`, без ORM-связей с Airport;
* изменение IATA-кода не поддерживается, чтобы не разрушать существующие ссылки;
* неиспользуемый аэропорт деактивируется, а не удаляется;
* отдельные domain events не добавляются: справочник пока не имеет внешних
  подписчиков или сложного lifecycle.

## API

```text
POST /api/v1/airports
GET  /api/v1/airports
GET  /api/v1/airports/{id}
PUT  /api/v1/airports/{id}
POST /api/v1/airports/{id}/activate
POST /api/v1/airports/{id}/deactivate
```

Поиск выполняется по IATA-коду, городу и названию аэропорта.

## Web

* отдельная административная страница `/airports`;
* создание, редактирование, поиск и активация/деактивация;
* формы FlightDefinition выбирают аэропорты из справочника;
* таблица FlightDefinition показывает город и IATA-код;
* при редактировании рейса ранее выбранный неактивный аэропорт остаётся видимым.

## Fixtures

Fixture `AirportFixtures` содержит аэропорты, используемые демонстрационными
рейсами Кишинёва, и входит в группы `airport-directory` и `flight-operations`.

## Out of Scope

* локализация названий;
* синхронизация с внешними авиационными каталогами;
* timezone, координаты и ICAO-коды;
* role-based access enforcement.


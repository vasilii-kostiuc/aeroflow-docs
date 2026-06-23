# Manage Announcements

Status: Done  
Bounded context: Announcements  
Service: aeroflow-core  
Change type: Domain, Application, Infrastructure, API

## Goal

Сохранить ручные рейсовые объявления и предоставить API для их создания,
просмотра, изменения языков и отмены.

Рейсовые объявления содержат идентификатор `FlightDefinition`. При добавлении
дополнительных и экстренных объявлений наличие этой ссылки будет определяться
типом объявления, без отдельной абстракции subject.

## Scope

* сохранить существующие рейсовые фабрики `Announcement`;
* добавить Doctrine mapping и миграцию;
* добавить repository и application handlers;
* проверять существование и активность `FlightDefinition` при создании;
* реализовать создание, получение и список объявлений;
* реализовать изменение количества и порядка языков;
* реализовать отмену;
* предоставить HTTP API;
* покрыть persistence, application и API тестами.

## Out of Scope

* frontend;
* дополнительное и экстренное объявления;
* `LocalizedContent`, шаблоны, `AudioAsset` и TTS;
* подготовка audio sequence;
* playback и RabbitMQ integration messages;
* редактирование типа, subject, стоек или выхода после создания;
* физическое удаление объявления.

## Domain Decisions

Инварианты:

* существующие рейсовые типы требуют корректный UUID `FlightDefinition`;
* при добавлении `Additional` и `Emergency` они не будут содержать ссылку на
  рейс;
* принадлежность к рейсу или аэропорту выводится из `AnnouncementType`;
* ссылка на рейс неизменяема после создания.

Языки хранятся как JSON-массив строк. JSON сохраняет переменное количество и
порядок; доменная модель продолжает отдавать `AnnouncementLanguages`.

## API

```text
POST /api/v1/announcements
GET  /api/v1/announcements
GET  /api/v1/announcements/{id}
PUT  /api/v1/announcements/{id}/languages
POST /api/v1/announcements/{id}/cancel
```

Create payload:

```json
{
  "type": "check_in_opening",
  "flightDefinitionId": "uuid",
  "languages": ["ro", "ru", "en"],
  "checkInCounterStart": 5,
  "checkInCounterEnd": 8,
  "gateCode": null
}
```

Для `boarding_invitation` обязателен `gateCode`; для `arrival` дополнительные
операционные параметры не передаются.

Update languages:

```json
{
  "languages": ["en", "ro"]
}
```

## Domain Events

* `AnnouncementCreated`;
* `AnnouncementLanguagesChanged`;
* `AnnouncementCancelled`.

События остаются внутренними. Integration messages не публикуются.

## Definition of Done

* Announcement сохраняется и восстанавливается Doctrine;
* порядок языков сохраняется после чтения из PostgreSQL;
* API защищён существующей аутентификацией;
* неактивная или отсутствующая FlightDefinition отклоняется;
* отмена и повторная установка языков идемпотентны;
* полный QA проходит.

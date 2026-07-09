# Core → Playback: RequestAnnouncementPlayback

Status: Done
Bounded context: Announcements (Core), Playback (aeroflow-playback)
Service: aeroflow-core, aeroflow-playback
Change type: Application, Infrastructure, Integration, Migration, новый сервис

## Goal

Связать `aeroflow-core` с `aeroflow-playback`: после запуска рейсового объявления
Core публикует integration command `RequestAnnouncementPlayback`, а playback
создаёт `PlaybackJob`.

```text
dispatcher action on FlightOccurrence
  -> FlightOccurrence transition + create Announcement   (одна локальная транзакция Core)
  -> commit
  -> AnnouncementCreated                                 (domain event, факт)
  -> RequestAnnouncementPlayback                         (integration command, RabbitMQ, после коммита)
  -> aeroflow-playback: создаёт PlaybackJob (Pending), идемпотентно по messageId
```

Это первый тонкий срез интеграции. Он доводит уже существующую функциональность
запуска объявлений до playback и поднимает `aeroflow-playback` как сервис. Очередь,
приоритетное прерывание, emergency mode и `aeroflow-agent` остаются следующими
срезами.

## Architectural Classification

Изменение интеграционное и инфраструктурное. Новых доменных правил в Core не
добавляется: `Announcement`, его `AudioSequence` и событие `AnnouncementCreated`
уже существуют.

Затрагиваемые контексты:

* `Announcements` (Core) — становится издателем `RequestAnnouncementPlayback`;
* `Playback` (aeroflow-playback) — новый сервис с aggregate root `PlaybackJob`.

`Flight Operations` не меняется и **не знает** про playback. Издателем является
контекст-владелец `Announcement` и готовой `AudioSequence` — `Announcements`.

## Decisions (зафиксированы в domain-model.md)

* издатель `RequestAnnouncementPlayback` — `Announcements`, реакция application
  layer на `AnnouncementCreated` **после** коммита локальной транзакции;
* транспорт между сервисами — RabbitMQ (Symfony Messenger, AMQP);
* durable transactional outbox **отложен**: post-commit публикация из application
  layer, известный риск потери при крахе между коммитом и публикацией — осознанный
  временный компромисс;
* первый срез playback создаёт `PlaybackJob` в статусе `Pending`, идемпотентно по
  `messageId`; очередь и emergency mode вне объёма.

## Integration Contract

`RequestAnnouncementPlayback` — публичный версионируемый контракт, нейтральный к
домену аэропорта (без `FlightDefinition`, `FlightOccurrence`, `Gate`,
`CheckInCounter`).

```json
{
  "messageId": "uuid",
  "correlationId": "announcement-uuid",
  "announcementId": "announcement-uuid",
  "type": "check_in_opening | check_in_closing | boarding_invitation | arrival",
  "priority": 100,
  "audioSequence": [
    {
      "languageCode": "ro-MD",
      "sortOrder": 0,
      "items": [
        { "type": "audio_asset", "audioAssetId": "uuid" },
        { "type": "pause", "durationMs": 500 }
      ]
    }
  ],
  "repeatRule": null,
  "occurredAt": "2026-06-26T10:00:00+00:00",
  "schemaVersion": 1
}
```

Правила контракта:

* `messageId` обеспечивает идемпотентность на стороне playback;
* `correlationId` = `announcementId` (задел под корреляцию сценария);
* `priority`: рейсовое объявление = `100` (enum 1000/100/50/0 из доков);
* `audioSequence` берётся из `Announcement::getAudioSequence()` как есть;
* `repeatRule` пока `null` (задел под `check_in_continuation`);
* контракт сериализуется как нейтральный DTO, доменная сущность не отправляется.

Контракт является общим артефактом двух сервисов. Размещение DTO/схемы
согласовать при реализации (общий пакет либо синхронизированная копия с проверкой
`schemaVersion`).

## Core Side (aeroflow-core)

### Публикация

* новый application event handler в `Announcements\Application\EventHandler` на
  `AnnouncementCreated`, который мапит `Announcement` в `RequestAnnouncementPlayback`
  и отправляет в async-транспорт;
* публикация идёт через существующий post-commit механизм
  (`DeferredDomainEventPublisher`): integration command отправляется только после
  успешного коммита локальной транзакции; на rollback — не отправляется;
* handler не обращается к write-side `FlightOccurrence` и не пересекает границу в
  обратную сторону.

Альтернатива (если `AnnouncementCreated` несёт не все нужные поля): публиковать
integration command явным шагом application layer запуска после коммита. Выбор
фиксируется при реализации, но издателем остаётся `Announcements`.

### Транспорт

* `config/packages/messenger.yaml`: добавить `async` AMQP-транспорт
  (`%env(MESSENGER_TRANSPORT_DSN)%`) и routing для `RequestAnnouncementPlayback`;
  доменные события остаются sync;
* `.env`: переключить `MESSENGER_TRANSPORT_DSN` на `amqp://...` (сейчас
  `doctrine://default`);
* `docker-compose.yml`: добавить сервис RabbitMQ (в core сейчас его нет).

## Playback Side (aeroflow-playback)

Сейчас `aeroflow-playback/` — пустая папка.

* scaffolding Symfony + Messenger + Doctrine + PostgreSQL + docker-compose;
* подписка на тот же RabbitMQ-обмен/очередь, что публикует Core;
* aggregate root `PlaybackJob` (минимум): `id`, `type`, `priority`, `status`,
  `scheduledAt` (nullable), `audioSequence`, `repeatRule`, `createdAt`;
* статус первого среза — `Pending`;
* message handler `RequestAnnouncementPlayback`:
  1. проверяет, не обработан ли `messageId` (таблица обработанных id либо unique
     constraint на бизнес-ключе);
  2. при повторе — возвращает уже достигнутый результат без дубликата;
  3. иначе создаёт и сохраняет `PlaybackJob` со статусом `Pending`;
* `PlaybackJob` не содержит аэропортовых сущностей и не знает про рейсы.

## Out of Scope

* приоритетная очередь, прерывание текущего задания, фоновая музыка;
* emergency mode (`ActivateEmergencyMode` / `DeactivateEmergencyMode`);
* `CancelAnnouncementPlayback` и отмена заданий;
* повторы `check_in_continuation` (`repeatRule`);
* обратные integration events playback → core
  (`AnnouncementPlaybackQueued/Started/Completed/...`);
* `aeroflow-agent` и реальное воспроизведение аудио;
* durable transactional outbox (отдельный последующий шаг);
* авторизация между сервисами.

## Documentation Updates

Решения уже зафиксированы до реализации в:

* `aeroflow-docs/domain-model.md` — издатель, транспорт, отложенный outbox,
  первый срез `PlaybackJob`, перенос outbox-вопроса в принятые решения;
* `aeroflow-docs/architecture.md` — уточнён шаг публикации в сценарии.

После реализации обновлено:

* `domain-model.md` — `PlaybackJob` (первый срез) переведён из планируемого в
  реализованную модель;
* `README.md` — отмечено, что запуск объявления доходит до playback как
  `PlaybackJob`.

## Acceptance Criteria

Core:

* при создании рейсового `Announcement` после коммита публикуется
  `RequestAnnouncementPlayback` в RabbitMQ;
* на rollback локальной транзакции integration command не публикуется;
* контракт содержит `messageId`, `correlationId`, `announcementId`, `type`,
  `priority`, `audioSequence`, `repeatRule`, `occurredAt`, `schemaVersion` и не
  содержит доменных сущностей;
* издатель — `Announcements`; `Flight Operations` не зависит от playback.

Playback:

* `aeroflow-playback` запускается как отдельный Symfony-сервис;
* consumer `RequestAnnouncementPlayback` создаёт `PlaybackJob` (`Pending`);
* повторная доставка того же `messageId` не создаёт второй `PlaybackJob`;
* `priority` и `audioSequence` сохраняются из контракта;
* playback не содержит `FlightDefinition`/`FlightOccurrence`/`Gate`/`CheckInCounter`.

Tests:

* Core: application test — `AnnouncementCreated` приводит к публикации
  `RequestAnnouncementPlayback` с корректным payload; rollback не публикует;
* Playback: handler test — создание `PlaybackJob` и идемпотентность по `messageId`;
* Playback: persistence test — маппинг `PlaybackJob`.

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-playback
php bin/phpunit
```

Manual QA:

```text
1. Поднять RabbitMQ, core и playback через docker-compose.
2. Запустить consumer playback.
3. В диспетчерской панели выполнить check-in:open для активного рейса.
4. Убедиться, что Core опубликовал RequestAnnouncementPlayback (RabbitMQ UI / лог).
5. Убедиться, что playback создал PlaybackJob со статусом Pending и той же
   audioSequence.
6. Повторно доставить то же сообщение (requeue) и убедиться, что второй
   PlaybackJob не создан.
7. Спровоцировать неготовый шаблон (422 при запуске) и убедиться, что ни
   Announcement, ни RequestAnnouncementPlayback, ни PlaybackJob не появились.
```

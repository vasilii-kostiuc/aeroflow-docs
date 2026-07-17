# Отмена ожидающего объявления (CancelAnnouncementPlayback)

Status: Done
Bounded context: Announcements (Core, издатель команды), Playback
(aeroflow-playback, отмена задания), dispatcher feature (aeroflow-web)
Service: aeroflow-core, aeroflow-playback, aeroflow-web
Change type: Application, Integration (первая вторая команда на очереди —
исполнение ADR 002), Frontend

## Goal

Дать диспетчеру возможность убрать **ожидающее** объявление из очереди
воспроизведения — срез B эпика экрана очереди (задача 017):

```text
Срез A (017)         экран очереди (read-only) — Done
Срез B (эта задача)  отмена ожидающего — CancelAnnouncementPlayback
Срез C (019)         Стоп — остановка текущего звука через агента
```

Legacy-контекст (вторичный источник): в окне `Статус` старой САО диспетчер
удалял строку ожидающего процесса; кнопка `Стоп` жила отдельно. Разделение
«удалить ожидающее» / «остановить текущее» сохраняется: этот срез отменяет
только ожидающее задание и не трогает уже звучащее.

## Уже существует (используется, не создаётся заново)

* `Announcement.cancel()` — идемпотентный переход в `Cancelled`, событие
  `AnnouncementCancelled` только при первом переходе (задача 006/007);
* `POST /api/v1/announcements/{id}/cancel` — HTTP-команда отмены (задача 007);
* `PlaybackJobStatus::Cancelled` в enum playback — статус объявлен, переход не
  реализован;
* обратный канал `playback_events` и generic-приём receipts в core
  (`PlaybackEventSerializer` дискриминирует по `event`, `RecordPlaybackEventReceipt`
  идемпотентен по `messageId`) — событие `cancelled` не требует изменений
  приёмной стороны, только read-модели;
* заготовка ADR 002: серьёзное изменение этого среза — дискриминатор `command`
  на очереди `announcement_playback`.

## Decisions

1. **Отмена — действие над `Announcement`, а не над рейсом.** Состоянием рейса
   владеет `FlightOccurrence`; отмена строки очереди его не меняет (решение 8
   задачи 017). Точка входа — существующий use case `CancelAnnouncement`;
   новый агрегатный код в core не нужен.
2. **HTTP-эндпоинт переиспользуется.** Web вызывает существующий
   `POST /api/v1/announcements/{id}/cancel`. Отдельный dispatcher-эндпоинт не
   вводится: это тот же use case контекста `Announcements`, ролевой сепарации
   в API сейчас нет, а дублирующий маршрут пришлось бы сопровождать. Ответ
   идемпотентен (повторная отмена — `200` с тем же результатом).
3. **Издатель `CancelAnnouncementPlayback` — контекст `Announcements`,
   post-commit.** Новый handler `PublishAnnouncementPlaybackCancel` на
   `AnnouncementCancelled` (event.bus, тот же deferred-механизм, что и
   `PublishAnnouncementPlaybackRequest`). `Flight Operations` и web про
   playback по-прежнему не знают.
4. **Исполнение ADR 002 — дискриминатор `command` в теле.** На очереди
   `announcement_playback` появляется второй тип сообщения:
   * `RequestAnnouncementPlayback` получает `command: announcement_playback.request`,
     `schemaVersion` контракта запроса инкрементируется до `2`;
   * новый контракт `CancelAnnouncementPlayback`: `command:
     announcement_playback.cancel`, `messageId`, `correlationId` (=
     `announcementId`), `announcementId`, `occurredAt`, `schemaVersion: 1` —
     без `audioSequence` и прочего;
   * `AnnouncementPlaybackSerializer` в playback: `match` по `command` с
     `MessageDecodingFailedException` на неизвестное значение. Тело **без**
     поля `command` временно трактуется как request (сообщение старого core,
     оставшееся в durable-очереди на момент деплоя); fallback помечается для
     удаления следующим контрактным изменением.
5. **Семантика отмены в playback: отменяется только `Pending`.**
   * `PlaybackJob.cancel()`: `Pending -> Cancelled`, идемпотентно в стиле
     `complete()`/`fail()` — повторная отмена уже `Cancelled` возвращает
     `false` и события не публикует;
   * job в статусе `Playing` или терминальном — «опоздавшая отмена»: команда
     подтверждается (ack) без изменения состояния и без события; остановка
     звучащего — срез C. В core объявление при этом уже `Cancelled` —
     расхождение осознанное и видимое: экран очереди строится по receipts, и
     строка останется `playing`/`completed`;
   * job не найден по `announcementId` — ack + лог. Гонка «cancel обогнал
     request» практически исключена: оба типа идут через одну очередь
     `announcement_playback`, порядок внутри очереди сохраняется, а отмена
     запускается только для строки, уже видимой на экране (receipt `queued`
     существует — значит job создан);
   * конкурентная защита: кандидат на отмену грузится с write lock в той же
     транзакционной манере, что `findNextPendingForUpdate`, чтобы отмена не
     гонялась с запуском этого же job диспетчером очереди.
6. **Обратное событие `AnnouncementPlaybackCancelled`.** Публикуется post-commit
   при первом переходе `Pending -> Cancelled` (тот же буфер, что Queued/Started/
   Completed/Failed). Контракт идентичен остальным: `event:
   announcement_playback.cancelled`, `messageId`, `correlationId`,
   `announcementId`, `jobId`, `occurredAt`, `schemaVersion`. Приёмная сторона
   core не меняется (generic по имени события).
7. **Read-модель: отменённая строка уходит в «Недавние».**
   `ListPlaybackQueueHandler` учитывает `announcement_playback.cancelled` как
   терминальный факт: `state: cancelled`, `finishedAt` = время receipt. До
   прихода обратного события строка остаётся в waiting — eventual-точность
   экрана уже принята задачей 017 (поллинг сгладит задержку).
8. **Web — кнопка отмены только на строках «В очереди».** В drawer очереди у
   waiting-строк появляется кнопка «Убрать» с подтверждением; mutation зовёт
   cancel-эндпоинт, дизейблит кнопку на время запроса и инвалидирует query key
   очереди. У играющей строки кнопки нет (это срез C). Строка `cancelled` в
   «Недавних» подписывается «Отменено».

## Core

* `PublishAnnouncementPlaybackCancel` (`Announcements\Application\EventHandler`,
  event.bus) — на `AnnouncementCancelled` публикует `CancelAnnouncementPlayback`
  через существующий `PlaybackRequestPublisherInterface`-механизм (тот же
  integration.bus / транспорт `announcement_playback`);
* контракт `CancelAnnouncementPlayback` в `Announcements\Application\Playback`;
* `RequestAnnouncementPlayback`: поле `command`, `SCHEMA_VERSION = 2`;
* `ListPlaybackQueueHandler`: ветка `announcement_playback.cancelled`
  (терминальный, не failed), `state: cancelled`;
* `OnAnnouncementCancelled` (TODO audit) не трогаем — публикация живёт в
  отдельном handler, как у created/request.

## Playback

* `PlaybackJob::cancel(): bool` — `Pending -> Cancelled`, идемпотентно;
  `Playing`/`Completed`/`Failed` для отмены не являются ошибкой протокола и
  решаются на application-уровне (skip + лог), а не исключением агрегата;
* `HandleCancelAnnouncementPlayback` (application): поиск job по
  `announcementId` (новый метод репозитория, с write lock), отмена, post-commit
  публикация `AnnouncementPlaybackCancelled`;
* `AnnouncementPlaybackSerializer`: `match` по `command` (см. Decision 4);
* `AnnouncementPlaybackCancelled` в `Application\IntegrationEvent` +
  `PlaybackEventSerializer` (исходящий encode) — по образцу failed;
* очередь после отмены: отдельный вызов `dispatchNext()` не требуется —
  отменённый job был `Pending` и просто перестаёт быть кандидатом выбора.

## Web

* `dispatcherApi`: `cancelAnnouncement(announcementId)` →
  `POST /api/v1/announcements/{id}/cancel`;
* `PlaybackQueueDrawer`: кнопка «Убрать» на waiting-строках, подтверждение,
  дизейбл на время мутации, invalidate query key очереди;
* лейбл состояния `cancelled` → «Отменено» в «Недавних».

## Out of Scope

* остановка текущего звука (срез C, задача 019) — `StopCurrentPlayback` агенту;
* отмена/откат перехода `FlightOccurrence` — состоянием рейса срез не управляет;
* emergency mode и отдельный emergency-транспорт (ADR 002, уровень 2);
* durable outbox (прежний осознанный компромисс post-commit публикации);
* registry декодеров (ADR 002, уровень 3) — типов на очереди два, `match`
  достаточно;
* удаление legacy-fallback сериализатора (тело без `command`) — следующее
  контрактное изменение.

## Acceptance Criteria

Core:

* отмена подготовленного объявления публикует `CancelAnnouncementPlayback`
  (`command: announcement_playback.cancel`, `correlationId` = `announcementId`)
  после коммита; повторная отмена того же объявления команду не дублирует;
* `RequestAnnouncementPlayback` уходит с `command:
  announcement_playback.request` и `schemaVersion: 2`;
* receipt `announcement_playback.cancelled` переводит строку read-модели в
  `recent` со `state: cancelled`.

Playback:

* `Pending`-job по `announcementId` переходит в `Cancelled`, `Playing` его не
  подхватывает, публикуется `AnnouncementPlaybackCancelled`;
* повторная доставка cancel-команды не меняет состояние и не публикует событие
  повторно;
* cancel по job в `Playing`/`Completed`/`Failed`/неизвестному `announcementId`
  подтверждается без изменения состояния и без события;
* сообщение с неизвестным `command` отклоняется декодером; тело без `command`
  обрабатывается как request (переходный fallback).

Web:

* waiting-строка имеет кнопку «Убрать»; после подтверждения и обновления
  очереди строка исчезает из «В очереди» и появляется в «Недавних» как
  «Отменено»;
* у играющей строки кнопки отмены нет.

Tests:

* Core: unit — контракт cancel-издателя (публикация только при первом
  переходе); application — `cancelled`-receipt в read-модели;
* Playback: domain — переходы `cancel()`; application — идемпотентность,
  «опоздавшая отмена», публикация события; serializer — `match` по `command`,
  fallback без `command`, ошибка на неизвестный;
* Web: компонентный — кнопка на waiting, отсутствие на playing, лейбл
  «Отменено».

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-playback
php bin/phpunit

# aeroflow-web
npm run lint && npm run typecheck && npm test
```

Manual QA: запустить два объявления подряд; пока первое звучит, «Убрать»
второе — строка уходит из «В очереди» в «Недавние» («Отменено»), после
завершения первого агент молчит; попытка отменить уже стартовавшее объявление
через API оставляет его звучать, строка остаётся `playing`.

## Documentation Updates

Выполнено:

* `domain-model.md` — раздел «Третий срез: отмена ожидающего (задача 018)»;
  контракты `RequestAnnouncementPlayback` v2 (+`command`) и
  `CancelAnnouncementPlayback` описаны в Announcements; ADR-пункт открытых
  решений отмечен исполненным;
* `architecture.md` — Core → Playback: обе команды реализованы на одной очереди
  с дискриминатором `command`; обратный поток дополнен `Cancelled`;
* `README.md` — экран очереди: отмена ожидающего реализована, «остановить
  текущее» — следующий срез;
* `adr/002-inbound-message-discrimination.md` — отметка в Status: уровни 1 и 3
  реализованы, уровень 2 ждёт emergency;
* `aeroflow-web/architecture.md` — drawer очереди больше не read-only: описана
  кнопка «Убрать».

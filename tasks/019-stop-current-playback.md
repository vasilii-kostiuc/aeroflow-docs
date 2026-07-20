# Остановка текущего воспроизведения

Status: Done
Bounded context: Playback (состояние задания и оркестрация), Audio Execution
(физическая остановка), Announcements (операторская команда и read-модель),
dispatcher feature
Service: aeroflow-core, aeroflow-playback, aeroflow-agent, aeroflow-web
Change type: Application, Domain, Integration, Infrastructure, Frontend

## Goal

Дать диспетчеру возможность остановить **текущее звучащее** объявление из окна
«Статус». Это срез C эпика очереди после read-only экрана (017) и удаления
ожидающего задания (018).

```text
dispatcher: «Стоп» для playing-строки
  -> core: StopAnnouncementPlayback (операторская команда, Announcement не меняется)
  -> playback: найти Playing job по announcementId
  -> agent_commands: audio.stop (jobId)
  -> agent: остановить локальный плеер
  -> agent_events: audio.playback_interrupted
  -> playback: Playing -> Interrupted, событие в core, следующий job
  -> core read-модель: «Прервано» в Недавних
```

Как и в старой САО, «Убрать» и «Стоп» различны: первое отменяет ожидающее
объявление, второе прерывает физически звучащее. Остановка не отменяет рейс и
не переводит `Announcement` в `Cancelled`.

## Decisions

1. Владельцем перехода `Playing -> Interrupted` остаётся aggregate
   `PlaybackJob`. Переход выполняется только по факту
   `audio.playback_interrupted` от агента, а не в момент принятия HTTP-команды:
   транспорт асинхронный, поэтому до подтверждения агентом job всё ещё звучит.
2. Core принимает `POST /api/v1/dispatcher/playback-queue/{announcementId}/stop`
   и публикует нейтральную команду `StopAnnouncementPlayback` в
   `announcement_playback`. Это application-команда, не доменное событие и не
   изменение aggregate `Announcement`.
3. Playback обрабатывает `StopAnnouncementPlayback` идемпотентно: только
   совпадающий `Playing` job даёт агенту `audio.stop`; неизвестный, ожидающий или
   терминальный job подтверждается без эффекта. Повторная доставка той же
   команды не должна послать второй stop.
4. Контракт Core -> Playback остаётся на `announcement_playback` и добавляет
   `command: announcement_playback.stop`; `RequestAnnouncementPlayback` и
   `CancelAnnouncementPlayback` не меняют семантику. Сериализатор продолжает
   явно отклонять неизвестный `command`.
5. Контракт Playback -> Agent на `agent_commands`: `audio.stop` с
   `messageId`, `jobId`, `occurredAt`, `schemaVersion: 1`. У агента нет
   `announcementId`, приоритетов или иной аэропортовой модели.
6. Чтобы stop не встал за долгим `audio.play_sequence`, агент подтверждает play
   после запуска фонового выполнения и хранит один активный `jobId` с
   `CancellationTokenSource`. Stop совпадающего job отменяет token; второй play
   не запускается, пока активный не сообщил терминальный результат.
7. Отмена процесса плеера не является ошибкой: `SequencePlayer` возвращает
   отдельный результат `Interrupted`, агент публикует
   `audio.playback_interrupted`. Реальная реализация убивает дочерний процесс
   плеера, simulate-режим корректно реагирует на cancellation token.
8. Playback по `audio.playback_interrupted` переводит job в `Interrupted`,
   публикует `announcement_playback.interrupted` после коммита и вызывает
   dispatcher для следующего Pending job. Повторный interrupted идемпотентен.
9. Core generic-приём receipt не меняется; read-модель добавляет terminal state
   `interrupted`. Web показывает «Прервано» в Недавних и кнопку «Стоп» только
   для playing-строки, с подтверждением и polling-инвалидацией.

## Aggregates and events

* Затронутый aggregate: `PlaybackJob`; новый разрешённый переход —
  `Playing -> Interrupted`.
* `Announcement` и `FlightOccurrence` не затрагиваются.
* Agent не имеет агрегатов и исполняет лишь нейтральные команды по `jobId`.
* Новые integration messages: `StopAnnouncementPlayback`, `audio.stop`,
  `audio.playback_interrupted`, `AnnouncementPlaybackInterrupted`.

## Out of scope

* pause/resume, volume и emergency mode;
* отмена или откат `FlightOccurrence`;
* переход `Announcement` в отдельный playback-lifecycle;
* recovery Playing-job после гибели агента;
* отдельный transport emergency-команд и durable outbox.

## Acceptance Criteria

* В web у playing-строки есть «Стоп», у waiting и recent — нет.
* HTTP stop не меняет `Announcement`, отправляет одну `StopAnnouncementPlayback`.
* Playback отправляет `audio.stop` только активному совпадающему job; повтор и
  опоздавшая команда являются no-op.
* Агент останавливает play, публикует ровно один `audio.playback_interrupted`;
  процесс плеера не остаётся работать.
* Interrupted-job не может снова завершиться или упасть; следующее Pending
  задание стартует после interrupted-события.
* Core показывает interrupted-job в recent как «Прервано».
* Есть unit/application/serializer tests в Core и Playback, xUnit-тесты агента
  для stop/cancellation и компонентный тест Web.

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-playback
php bin/phpunit

# aeroflow-agent
dotnet test

# aeroflow-web
npm run lint && npm run typecheck && npm run test:run
```

Manual QA: запустить два объявления; во время первого нажать «Стоп»; первое
появляется в «Недавних» как «Прервано», а второе начинает играть. Повторный stop
и stop уже завершённого job не меняют очередь.

Выполнено локально сквозным прогоном через реальные контейнеры (core, playback,
agent, RabbitMQ, обе Postgres): синтетический `RequestAnnouncementPlayback` дошёл
до job, `playback:job:complete --fail` (аварийный стаб) освободил очередь, агент
принял и отработал команду play, а `queued`/`started`/`failed` receipts легли в
core-БД — цепочка `core -> playback -> agent -> playback -> core` подтверждена
живьём, а не только unit-тестами.

## Documentation Updates

Выполнено:

* `domain-model.md` — раздел «Четвёртый срез: остановка звучащего (задача 019)»
  у `PlaybackJob`; статус `Audio Execution` дополнен вторым срезом агента
  (`audio.stop`, единый активный `CancellationTokenSource`, освобождение слота
  до публикации терминального события); открытые решения зафиксировали, что
  переход `Playing -> Interrupted` управляется только фактом от агента;
* `architecture.md` — Core → Playback: третья команда на очереди
  `announcement_playback`; Playback → Agent: второй срез агента (`audio.stop`);
  Playback → Core: обратный поток дополнен `Interrupted`;
* `README.md` — очередь воспроизведения: `Interrupted` в списке статусов, срез
  019 описан рядом с 018, «Остановка текущего» убрана из списка будущих срезов;
* `adr/002-inbound-message-discrimination.md` — подтверждение, что третий тип на
  одной очереди (уровень 1 и 3) не потребовал registry decoders;
* `aeroflow-web/architecture.md` — drawer очереди: кнопка «Стоп» на играющей
  строке, eventual-переход в «Прервано».

# Playback: приоритетная очередь и жизненный цикл задания

Status: Ready
Bounded context: Playback (aeroflow-playback), Announcements (Core) — только приём событий
Service: aeroflow-playback, aeroflow-core
Change type: Domain, Application, Infrastructure, Integration, Migration

## Goal

Довести `aeroflow-playback` от «создать `PlaybackJob(Pending)`» до реальной
последовательной очереди: сервис выбирает следующее задание по приоритету,
переводит его в проигрывание, завершает и публикует обратные integration events в
`aeroflow-core`.

```text
RequestAnnouncementPlayback -> PlaybackJob(Pending)          (срез 013, уже есть)
  -> dispatcher выбирает следующий job по приоритету
  -> Pending -> Playing (не более одного активного на audio output)
  -> AnnouncementPlaybackStarted                              (integration event -> core)
  -> сигнал завершения audio output
  -> Playing -> Completed
  -> AnnouncementPlaybackCompleted                            (integration event -> core)
  -> dispatcher выбирает следующий job
```

Это второй тонкий срез playback. Он делает очередь наблюдаемой и продвигаемой, но
**без** приоритетного прерывания, emergency mode, отмены и реального агента.

## Architectural Classification

Изменение доменное и интеграционное **внутри `aeroflow-playback`**. Появляется
поведение у `PlaybackJob` (переходы статуса) и доменный сервис/политика выбора
следующего задания.

`aeroflow-core` затрагивается только как получатель обратных integration events
(новый inbound-контекст в `Announcements` или отдельный read-model апдейтер).
`Flight Operations` не меняется. Playback по-прежнему **не знает** про рейсы,
стойки, выходы — работает только с нейтральным `PlaybackJob` и его
`audioSequence`.

## Decisions (приняты, зафиксировать в domain-model.md до реализации)

1. **Без прерывания в этом срезе.** По докам обычные объявления не прерывают друг
   друга (прерывает только emergency). Значит 015 = строго последовательная
   очередь: активный job доигрывается, следующий выбирается только после его
   завершения. Прерывание переносится в срез emergency mode.
2. **Порядок выбора:** сначала по `priority` (100 > 50 > 0), при равенстве —
   FIFO по `createdAt` (задел `scheduledAt` учитывается, если задан).
3. **Single-active инвариант:** не более одного `PlaybackJob` в статусе `Playing`
   на один audio output. В первой версии audio output один (глобальный).
4. **Триггер завершения — вариант A.** Без `aeroflow-agent` реального «конца
   аудио» нет. Завершение моделируется явной application-командой
   `CompletePlaybackJob(jobId)` (аналогично `FailPlaybackJob`) — она представляет
   факт «audio output сообщил о завершении». Реальным источником в будущем станет
   событие агента `PlaybackCompleted`, которое ляжет в эту команду без изменения
   доменной модели; сейчас её в тестах подаёт стаб. Известное следствие: в проде
   до появления агента очередь останавливается на первом `Playing` — осознанное
   ограничение тонкого среза (как `Pending`-навсегда в 013).
5. **Outbound-команда агенту (`PlayAudioSequence`) отложена.** 015 остаётся
   playback-внутренним: переход в `Playing` не порождает внешней команды. Контракт
   агента проектируется в agent-срезе вместе с выбором транспорта.
6. **`AnnouncementPlaybackQueued` публикуется уже в 015** — при создании
   `PlaybackJob`; дёшево и даёт core ранний факт «запрос дошёл до playback».
7. **Core-сторона минимальна:** `Announcements` идемпотентно (по `messageId`)
   принимает обратные события и фиксирует факт (лог/таблица приёма). Статусы
   `Announcement` не меняются — расширение его жизненного цикла
   (`Playing`/`Played`) выносится в отдельную задачу.
8. **Фоновая музыка:** приоритет `0` остаётся только правилом порядка; продюсера
   музыки и idle-состояния в 015 нет.

## Queue & lifecycle

### Переходы статуса `PlaybackJob`

```text
Pending    -> Playing      (dispatcher выбрал и запустил)
Playing    -> Completed    (audio output завершил)
Playing    -> Failed       (audio output сообщил об ошибке)
```

Вне объёма 015 (следующие срезы): `Scheduled` (отложенный запуск),
`Interrupted` (emergency-прерывание), `Cancelled` (`CancelAnnouncementPlayback`).

### Инварианты

* не более одного `Playing` на audio output одновременно;
* запуск возможен только из `Pending`;
* `Completed`/`Failed` терминальны: повторный запуск запрещён;
* повторное завершение уже завершённого job идемпотентно (не бросает и не
  публикует событие повторно);
* приоритетный порядок соблюдается: пока есть `Playing`, новые `Pending` только
  ждут; следующий выбирается детерминированно (priority, затем FIFO).

### Dispatcher

Application-политика «выбрать и запустить следующий»:

* триггерится (а) после сохранения нового `PlaybackJob(Pending)` и (б) после
  завершения текущего job;
* если активный (`Playing`) job есть — ничего не делает;
* иначе выбирает верхний `Pending` по правилу порядка, переводит в `Playing` и
  публикует `AnnouncementPlaybackStarted`; внешняя команда на audio output не
  отправляется (решение #5 — контракт агента в следующем срезе);
* конкурентная защита: выбор следующего грузит кандидата на запись
  (optimistic/pessimistic lock), чтобы два процесса не запустили два задания.

## Обратные integration events (Playback -> Core)

Публикуются playback после соответствующего перехода (тот же post-commit
принцип, durable outbox по-прежнему отложен):

* `AnnouncementPlaybackQueued` — при создании `PlaybackJob`;
* `AnnouncementPlaybackStarted` — при `Pending -> Playing`;
* `AnnouncementPlaybackCompleted` — при `Playing -> Completed`.

Контракт нейтрален и версионируется: `messageId`, `correlationId` =
`announcementId`, `announcementId`, `jobId`, `occurredAt`, `schemaVersion`.
Не содержит аэропортовых сущностей.

Core-сторона (решение #7): `Announcements` идемпотентно (по `messageId`)
принимает эти события и фиксирует факт приёма (лог/таблица приёма). Статусы
`Announcement` не меняются — у него остаются `Prepared`/`Cancelled`; расширение
жизненного цикла по playback-событиям выносится в отдельную задачу. Транспорт —
RabbitMQ (обратное направление, отдельный обмен/очередь).

## Out of Scope

* приоритетное **прерыв­ание** активного задания;
* emergency mode (`ActivateEmergencyMode` / `DeactivateEmergencyMode`);
* `CancelAnnouncementPlayback` и статус `Cancelled`;
* отложенный запуск (`Scheduled`, `scheduledAt`-планирование);
* фоновая музыка как реальный источник (нет продюсера priority `0`);
* повторы `check_in_continuation` (`repeatRule` по-прежнему `null`);
* реальный `aeroflow-agent` и физическое воспроизведение (в том числе
  outbound-команда `PlayAudioSequence` — решение #5);
* расширение статусов `Announcement` по playback-событиям (решение #7 —
  отдельная задача);
* durable transactional outbox;
* авторизация между сервисами.

## Acceptance Criteria

Playback:

* при создании `PlaybackJob` публикуется `AnnouncementPlaybackQueued`;
* при отсутствии активного задания dispatcher переводит верхний по приоритету
  `Pending` в `Playing` и публикует `AnnouncementPlaybackStarted`;
* пока есть `Playing`, новые `Pending` не запускаются (single-active инвариант);
* завершение активного job (`Completed`) публикует `AnnouncementPlaybackCompleted`
  и запускает следующий по порядку (priority, затем FIFO);
* порядок детерминирован: `100` раньше `50` раньше `0`; при равенстве — раньше
  созданный;
* повторное завершение того же job идемпотентно;
* playback не содержит `FlightDefinition`/`FlightOccurrence`/`Gate`/`CheckInCounter`.

Core:

* `Announcements` идемпотентно (по `messageId`) принимает обратные события и
  фиксирует факт приёма; статусы `Announcement` не меняются (решение #7).

Tests:

* Domain: переходы `PlaybackJob` и инварианты (single-active, терминальность,
  идемпотентное завершение).
* Application: dispatcher выбирает верное следующее задание при нескольких
  `Pending` разного приоритета и времени; не запускает второе при активном.
* Application: завершение публикует `AnnouncementPlaybackCompleted` и продвигает
  очередь.
* Integration/persistence: маппинг новых статусов; конкурентный выбор не
  запускает два задания.
* Core: приём обратного события идемпотентен и фиксирует факт приёма.

## Verification

```bash
# aeroflow-playback
php bin/phpunit

# aeroflow-core
php bin/phpunit tests/Unit tests/Application
```

Manual QA:

```text
1. Поднять RabbitMQ, core и playback.
2. Запустить два рейсовых объявления подряд (priority 100) для разных рейсов.
3. Убедиться: играет только одно (Playing), второе ждёт (Pending); опубликован
   AnnouncementPlaybackStarted для первого.
4. Завершить первое (CompletePlaybackJob / стаб) — убедиться, что опубликован
   AnnouncementPlaybackCompleted и стартовало второе.
5. Проверить порядок: добавить дополнительное (50) и рейсовое (100) при пустой
   очереди — первым стартует 100.
```

## Documentation Updates

До реализации зафиксировать в `aeroflow-docs`:

* `domain-model.md` — перевести соответствующую часть `PlaybackJob` (жизненный
  цикл, dispatcher, single-active) в реализованную модель; описать обратные
  integration events и зафиксировать, что статусы `Announcement` по ним пока не
  меняются (решение #7);
* `architecture.md` — уточнить обратный поток Playback -> Core;
* `README.md` — отметить, что задание доигрывается и core получает результат.

После реализации — перевести статус задачи в Done и обновить перечисленное выше.

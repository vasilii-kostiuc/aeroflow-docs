# Видимость тиков повтора в очереди диспетчера

Status: Done
Bounded context: Playback (публикация факта перевзвода), Announcements (приём
receipt и read-модель очереди), dispatcher feature
Service: aeroflow-core, aeroflow-playback, aeroflow-web
Change type: Application, Integration, Infrastructure, Frontend

## Goal

Показать диспетчеру повторяемое объявление таким, какое оно есть: звучащим в
момент тика и ожидающим следующего тика между тиками. Сейчас серия
«продолжение регистрации» выглядит на экране «Статус» непрерывно играющей от
открытия до закрытия регистрации.

```text
playback: тик доиграл -> Playing -> Scheduled (re-arm)
  -> playback_events: announcement_playback.rescheduled (nextAt)
  -> core: receipt (generic приём, без изменений)
  -> read-модель: строка уходит из «Сейчас звучит» в «В очереди»
  -> web: «Следующий повтор в HH:MM»
  -> следующий wake -> started -> строка снова «Сейчас звучит»
```

Задача закрывает известное ограничение пятого среза (задача 020,
`domain-model.md`, «Пятый срез: повтор continuation-задания»), где сознательно
решили не вводить новый тип события. Ограничение оказалось не косметическим —
см. Decision 1.

## Проблема

1. **Фантомная playing-строка вытесняет реальную.** `ListPlaybackQueueHandler`
   держит один слот «Сейчас звучит» и присваивает его без разбора
   (`$playing = $row`). Повторяемый job после первого `started` навсегда
   остаётся «начатым без завершения», поэтому пока серия жива, он и любое
   реально звучащее объявление конкурируют за слот, и победитель зависит от
   порядка receipts. Диспетчер может не увидеть то, что действительно играет.
   Это баг, а не только неточная подпись.
2. **Каденция повтора не видна.** Нет ни признака «идёт серия», ни времени
   следующего тика; отличить живой повтор от зависшего job на экране нельзя.

## Decisions

1. Playback публикует новый обратный integration event
   `announcement_playback.rescheduled` — факт перевзвода повторяемого job
   (`Playing -> Scheduled`). Это меняет решение задачи 020 «повтор не добавляет
   новый тип события»; изменение вносится в `domain-model.md` до реализации, а
   не по ходу правки handler.
2. Точка публикации — ветка `PlaybackCompletionResult::Rearmed` в
   `HandleCompletePlaybackJob`: рядом с уже существующим `planWake`, после
   коммита, тем же публикатором, что и остальные события. Терминация серии
   по-прежнему публикует `announcement_playback.completed`; `failed`,
   `cancelled`, `interrupted` не меняются.
3. Контракт `AnnouncementPlaybackEvent` получает необязательное поле `nextAt`
   (ATOM, момент следующего тика = `scheduledAt` job). Поле заполняет только
   `rescheduled`; у остальных событий оно `null`, как `reason` у `failed`.
   `schemaVersion` остаётся `1`: добавление необязательного поля обратно
   совместимо, старый потребитель его игнорирует.
4. Core-приём остаётся generic по имени события. Расширяются только
   `PlaybackEventSerializer` (decode/encode `nextAt`), `PlaybackIntegrationEvent`,
   `PlaybackEventReceipt` (nullable-колонка `next_at` + миграция) и
   `PlaybackEventReceiptView`. Новый handler не вводится.
5. Read-модель `ListPlaybackQueue` переходит с накопления флагов на
   **last-event-wins по `jobId`**: состояние строки определяет последнее по
   `receivedAt` событие. Это обязательно: повторяемый job переиспользует один
   `jobId` на всю серию, поэтому при накоплении флагов однажды выставленный
   `finishedAt` навсегда прижимает строку к «Недавним», а следующий `started`
   его не снимает. Порядок чтения receipts (`receivedAt ASC, id ASC`) уже
   детерминированный, менять reader не нужно.
6. Появляется нетерминальное состояние строки `rescheduled`. Оно попадает в
   секцию «В очереди», а не в «Недавние»: серия жива, объявление ещё прозвучит.
   Терминальные `completed`/`failed`/`cancelled`/`interrupted` работают как
   раньше.
7. Сортировка «В очереди» остаётся по моменту постановки; для `rescheduled`
   строки ключом сортировки служит `nextAt`, чтобы ожидающий тик стоял в очереди
   по времени, когда он реально прозвучит.
8. Web показывает у `rescheduled` строки подпись «Следующий повтор в HH:MM» и
   не показывает кнопку «Стоп» (останавливать нечего — ничего не звучит).
   Кнопка «Убрать» на такой строке также не показывается: серию завершает
   закрытие регистрации, а не удаление строки из очереди.

## Aggregates and events

* Агрегаты не затрагиваются: `PlaybackJob` уже умеет re-arm (задача 020),
  переходы и инварианты не меняются.
* `Announcement` и `FlightOccurrence` не затрагиваются.
* Новый integration event: `AnnouncementPlaybackRescheduled`
  (`announcement_playback.rescheduled`).
* Новых команд, очередей и транспортов нет.

## Out of scope

* изменение жизненного цикла `Announcement` по playback-событиям;
* остановка отдельного тика (стоп звучащего тика уже работает через задачу 019);
* счётчик прозвучавших тиков и история серии;
* realtime-транспорт вместо поллинга;
* emergency mode и приоритетное прерывание.

## Acceptance Criteria

* После завершения тика повторяемого job playback публикует ровно один
  `announcement_playback.rescheduled` с корректным `nextAt`; терминация серии
  по-прежнему публикует `completed`, а не `rescheduled`.
* Одноразовый job не публикует `rescheduled` ни при каких условиях.
* Повторная доставка завершения (`NoOp`) не публикует второй `rescheduled` и не
  переносит `nextAt`.
* Core сохраняет `next_at` в receipt; сериализатор round-trip'ит поле, а
  сообщение без `next_at` декодируется без ошибки.
* Read-модель показывает строку серии как `playing` во время тика и как
  `rescheduled` между тиками; по завершении серии строка уходит в «Недавние»
  как «Завершено».
* Реально звучащее объявление занимает слот «Сейчас звучит» даже когда
  continuation-серия активна — фантомная строка больше не конкурирует за слот.
* Web показывает время следующего повтора и не показывает «Стоп»/«Убрать» на
  `rescheduled` строке.
* Есть тесты: application-тест публикации в playback, тест сериализатора и
  read-модели в core (включая сценарий `started -> rescheduled -> started`),
  компонентный тест drawer'а в web.

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-playback
php bin/phpunit

# aeroflow-web
npm run lint && npm run typecheck && npm run test:run
```

Manual QA: открыть регистрацию рейсу с настроенным `repeatEveryMinutes = 1`;
на экране «Статус» строка продолжения регистрации чередует «Сейчас звучит» и
«В очереди — следующий повтор в HH:MM». Во время ожидания тика запустить
посадку другого рейса: она занимает «Сейчас звучит», строка серии остаётся в
«В очереди». Закрыть регистрацию — серия уходит в «Недавние» как «Завершено».

Выполнено локально сквозным прогоном через реальные контейнеры (core, playback,
agent, RabbitMQ, обе Postgres): синтетический `RequestAnnouncementPlayback` с
`repeatRule.everyMinutes = 1` создал повторяемый job, wake разбудил тик, агент
отыграл последовательность, а re-arm опубликовал `rescheduled` — receipt с
`next_at` лёг в core-БД и строка отобразилась в `ListPlaybackQueue` как
ожидающая следующего повтора.

По ходу реализации нашлась и починена смежная поломка web: `PlaybackQueueRow`
типизировал `announcementType` как `DispatcherActionType`, где нет
`check_in_continuation`, поэтому строка серии рисовалась вообще без названия
типа (`actionLabels[...] === undefined`). Введён тип `AnnouncementType` и карта
`announcementTypeLabels`; список ручных действий диспетчера не изменился.

## Documentation Updates

Выполнено:

* `domain-model.md` — «Пятый срез: повтор continuation-задания»: снять
  ограничение «re-arm никакого события не публикует» и описать
  `announcement_playback.rescheduled`; дополнить список публикуемых
  integration events `PlaybackJob`; в «Открытых решениях» отметить, что решение
  задачи 020 об отсутствии нового типа события пересмотрено этой задачей;
* `architecture.md` — раздел «Playback → Core»: новый обратный поток и
  необязательное поле `nextAt` в контракте;
* `README.md` — очередь воспроизведения: экран «Статус» показывает каденцию
  повтора;
* `aeroflow-web/architecture.md` — drawer очереди: состояние `rescheduled`,
  подпись следующего повтора, отсутствие «Стоп»/«Убрать» на такой строке.

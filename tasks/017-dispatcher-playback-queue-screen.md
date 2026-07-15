# Экран очереди воспроизведения (наследник окна «Статус»)

Status: Done
Bounded context: Announcements (Core, read-модель), Playback (aeroflow-playback,
событие failed), dispatcher feature (aeroflow-web)
Service: aeroflow-core, aeroflow-playback, aeroflow-web
Change type: Application (CQRS read), Integration (одно событие), Frontend

## Goal

Дать диспетчеру видимость очереди воспроизведения — UX-наследник окна `Статус`
старой САО: что играет сейчас, что ждёт, что только что отыграло. Экран
read-only; действия над строками — следующие срезы одного эпика:

```text
Срез A (эта задача)  экран очереди (read-only)
Срез B (018)         отмена ожидающего — CancelAnnouncementPlayback
Срез C (019)         Стоп — остановка текущего звука через агента
```

Legacy-контекст (вторичный источник): окно `Статус` показывало строки ожидающих
процессов (рейс, вид объявления, параметры), кнопка `Стоп` жила отдельно.
Разделение «остановить текущее» / «удалить ожидающее» сохраняется и в новой
системе (вывод зафиксирован в самом legacy-гайде).

## Decisions

1. **Источник данных — read-модель в core, не API playback.** Обратные события
   задачи 015 уже дают core всё нужное: `playback_event_receipt` хранит
   `Queued`/`Started`/`Completed` с `announcementId` и `jobId`. Web продолжает
   ходить только в core (архитектурная диаграмма `web -> core`); второй
   публичный API у playback не появляется. Точность eventual — приемлемо для
   экрана статуса; playback остаётся источником истины порядка.
2. **Вывод состояния из receipts:** `queued` без `started` — ждёт; `started` без
   терминального — играет; `completed`/`failed` — завершено. Это локальный
   CQRS-read контекста `Announcements` (прецедент —
   `ListConfiguredAnnouncementLanguages`); агрегаты не загружаются, чужие
   таблицы не читаются.
3. **Playback добавляет обратное событие `AnnouncementPlaybackFailed`.** Без
   него упавшее задание навсегда выглядело бы «играющим» на экране. Контракт
   тот же (`event: announcement_playback.failed` + `reason`); приёмная сторона
   core generic по имени события и не меняется. Это осознанное расширение
   минимальной core-стороны 015, вызванное реальной потребностью экрана.
4. **Состав экрана:** текущее играющее (0..1), ожидающие в порядке очереди,
   последние завершённые/упавшие (ограниченное число, для уверенности «оно
   отыграло»). Порядок ожидающих аппроксимируется как в playback: priority
   desc, затем время `queued`.
5. **Строка экрана** обогащается данными `Announcement` по `announcementId`:
   номер рейса (snapshot), тип объявления, языки, стойки/выход (snapshots),
   времена queued/started. «Время до следующего автозапуска» из legacy —
   будущий срез повторов (`repeatRule`), не здесь.
6. **Обновление — поллинг** TanStack Query (`refetchInterval` несколько секунд,
   только пока экран открыт). Realtime-транспорт — отдельное будущее решение
   (web architecture.md); при ошибке запроса интерфейс показывает, что данные
   могут быть неактуальны.
7. **Web-форма — кнопка `Статус` на панели диспетчера** (знакомое диспетчеру
   имя из старой САО), открывающая drawer/модал с очередью. Отдельный маршрут
   не требуется.
8. **Отмена строки из этого экрана НЕ отменяет рейс.** В legacy удаление строки
   использовали как суррогат отмены рейса; у нас состоянием рейса владеет
   `FlightOccurrence`. Срез B отменит только объявление/задание.

## Core: read-модель

* Query `ListPlaybackQueue` в `Announcements\Application` (command.bus, как
  остальные reads);
* источник: `playback_event_receipt` (агрегация по `announcementId`/`jobId` по
  последнему событию) + данные `Announcement` для отображения;
* результат-DTO: `playing` (0..1), `waiting[]`, `recent[]` (последние N=10
  завершённых/упавших за сегодня);
* поля строки: `announcementId`, `jobId`, `flightNumber`, `announcementType`,
  `languages`, `checkInCounters`/`gate` (если есть), `state`, `queuedAt`,
  `startedAt`, `finishedAt`, `failureReason?`;
* HTTP: `GET /api/v1/dispatcher/playback-queue` (JWT диспетчера, как остальные
  dispatcher-эндпоинты).

## Playback: событие failed

* `HandleFailPlaybackJob` при переходе `Playing -> Failed` публикует
  `AnnouncementPlaybackFailed` (post-commit, тот же буфер);
* контракт: `event: announcement_playback.failed`, `reason` (из события агента,
  если было), остальные поля как у Queued/Started/Completed;
* идемпотентность прежняя: повторный fail не публикует событие повторно.

## Web: dispatcher feature

* кнопка `Статус` на панели диспетчера открывает drawer;
* секции: «Сейчас звучит», «В очереди», «Недавние»;
* поллинг только при открытом drawer; индикатор устаревших данных при ошибке;
* фича `dispatcher`, новый api-модуль `playbackQueueApi` + query key.

## Out of Scope

* отмена ожидающего (срез B, задача 018) и остановка текущего (срез C, 019);
* действия над строками вообще — экран read-only;
* realtime-транспорт (WebSocket/SSE);
* таймер «до следующего автозапуска» (повторы, `repeatRule`);
* история/аудит за произвольные даты.

## Acceptance Criteria

Core:

* `GET /api/v1/dispatcher/playback-queue` возвращает playing/waiting/recent,
  выведенные из receipts, с данными объявления;
* объявление с `queued`-receipt без `started` — в waiting; со `started` без
  терминального — playing; `completed`/`failed` за сегодня — в recent (N=10);
* запрос не загружает агрегат `FlightOccurrence` и не читает таблицы playback.

Playback:

* `Playing -> Failed` публикует `AnnouncementPlaybackFailed` с `reason` после
  коммита; повторный fail события не дублирует;
* core фиксирует receipt `announcement_playback.failed` без изменений своего
  кода приёма.

Web:

* кнопка `Статус` открывает drawer с тремя секциями;
* упавшее задание видно в «Недавних» с причиной;
* при ошибке запроса показывается индикатор неактуальности данных.

Tests:

* Core: unit/application — вывод состояний из набора receipts (queued-only,
  started, completed, failed, порядок waiting);
* Playback: fail публикует событие, повторный fail — нет;
* Web: компонентный тест секций по мок-ответу.

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-playback
php bin/phpunit

# aeroflow-web
npm run lint && npm run typecheck && npm test
```

Manual QA: запустить два объявления подряд, открыть `Статус` — первое в
«Сейчас звучит», второе в «В очереди»; после завершения оба в «Недавних»;
убить агента посреди звука и добить job стабом `--fail` — строка в «Недавних»
с причиной.

## Documentation Updates

Выполнено:

* `domain-model.md` — `Failed` добавлен в реализованные обратные события
  (с `reason`); зафиксирован read `ListPlaybackQueue`;
* `architecture.md` — обратный поток: `Failed` реализован, упомянута read-модель
  экрана очереди;
* `README.md` — упомянут экран очереди диспетчера (кнопка «Статус»).

# Дополнительное объявление (supplementary, priority 50)

Status: Done
Bounded context: Announcements (Core — новый не-рейсовый тип и путь запуска),
Playback (без изменений домена — только приоритет в контракте), dispatcher
feature (aeroflow-web)
Service: aeroflow-core, aeroflow-web
Change type: Domain, Application, Integration, Frontend

## Goal

Дать диспетчеру возможность запустить **информационное объявление аэропорта**, не
привязанное к рейсу, с приоритетом `50`. Оно ставится в очередь обычным заданием,
играет, когда очередь свободна (позади рейсовых `100`, впереди фоновой музыки `0`),
и видно на экране «Статус» наравне с рейсовыми.

```text
диспетчер: текст + языки
  -> Announcements: Announcement(type=supplementary, flightDefinitionId=null, priority=50)
  -> RequestAnnouncementPlayback (priority 50)
  -> playback: обычная последовательная очередь (без изменений домена playback)
```

Это первый не-рейсовый (аэропортовый) путь объявления в системе. Он же —
**фундамент core-стороны для будущего emergency-режима** (priority `1000`): тот
переиспользует ту же способность «создать объявление вне `FlightOccurrence`»,
добавляя лишь преемпцию в playback и глобальную приостановку очереди (см. Out of
Scope и задачи 025/026).

README уже перечисляет «Дополнительное объявление» как тип
(`README.md`, раздел «Типы объявлений», приоритет `50` в «Приоритеты
воспроизведения»). Doc-модель уже допускает аэропортовую принадлежность:
«принадлежность объявления к рейсу или всему аэропорту определяется его типом, без
отдельного `AnnouncementSubject`» (`domain-model.md`, инварианты `Announcement`).
Этот срез — первая реализация этой ветки.

## Уже существует (используется, не создаётся заново)

* `Announcement::createPrepared(...)` и агрегат `Announcement` с состоянием
  `Prepared`/`Cancelled` — расширяются, не пишутся заново
  (`src/Announcements/Domain/Entity/Announcement.php`);
* `AnnouncementType` enum с методами `requiresCheckInCounters()`,
  `requiresGate()`, `isRepeatable()` — добавляется кейс и методы приоритета/
  привязки к рейсу;
* пост-коммит издатель `PublishAnnouncementPlaybackRequest` (реагирует на
  `AnnouncementCreated`, `event.bus`) и контракт `RequestAnnouncementPlayback` —
  сейчас захардкоживает `FLIGHT_ANNOUNCEMENT_PRIORITY = 100`
  (`src/Announcements/Application/EventHandler/PublishAnnouncementPlaybackRequest.php:51`);
* TTS-озвучивание текста: `SpeechAssetGeneratorInterface` +
  `TextSegmentSpeechResolver` (задача 022) — переиспользуются для синтеза
  ad-hoc-текста дополнительного объявления в `AudioSequence`;
* отмена (`POST /api/v1/announcements/{id}/cancel`, задача 018) и остановка
  звучащего (`POST /api/v1/dispatcher/playback-queue/{announcementId}/stop`,
  задача 019) — работают для любого `Announcement`, поэтому дополнительное
  объявление получает «Убрать»/«Стоп» на экране очереди **бесплатно**;
* read-модель `ListPlaybackQueue` и drawer «Статус» — не меняются: новая строка
  приходит теми же receipts, отличается только приоритетом сортировки.

## Decisions

1. **Модель: дополнительное — не-рейсовый `Announcement`, отличается типом.**
   Отдельный агрегат или `AnnouncementSubject` не вводится — doc-модель уже
   говорит, что аэропортовая принадлежность определяется типом. Конкретно:
   * `AnnouncementType::Supplementary = 'supplementary'`;
   * `requiresCheckInCounters()` / `requiresGate()` / `isRepeatable()` → `false`;
   * новый метод `requiresFlight(): bool` — `true` для пяти рейсовых типов,
     `false` для `Supplementary`;
   * `Announcement::$flightDefinitionId` становится **nullable**;
     `createPrepared()` требует валидный `flightDefinitionId` для типов с
     `requiresFlight() === true` и запрещает его (или игнорирует) для
     supplementary. `flightOccurrenceId` у дополнительного всегда `null`;
   * `AnnouncementCreated.flightDefinitionId` меняется на `?string`. Приёмники
     события (`PublishAnnouncementPlaybackRequest`, audit-TODO) не полагаются на
     ненулёвость — проверить.
2. **Приоритет выводится из типа, а не из константы.** На `AnnouncementType`
   добавляется `playbackPriority(): int`: рейсовые → `100`, `Supplementary` →
   `50` (значения из `README.md` «Приоритеты воспроизведения»).
   `PublishAnnouncementPlaybackRequest` берёт `priority` из типа объявления, а не
   из `FLIGHT_ANNOUNCEMENT_PRIORITY`. Контракт `RequestAnnouncementPlayback` не
   меняется (поле `priority` уже есть, `schemaVersion` не растёт — значение поля,
   не форма).
> **Revision (реализация):** Decision 3 пересмотрено после сверки с
> `legacy-sao-user-guide.md` — в старой САО раздел «Дополнительно» был **набором
> заранее записанных пресетов** (грипп, такси, безопасность, …), которые диспетчер
> выбирал и запускал, а не свободным текстом. Реализация идёт через **пресеты**:
> новый aggregate root `SupplementaryAnnouncementTemplate` (имя + языковые варианты
> `languageCode -> audioAssetId`), admin CRUD, dispatcher выбирает пресет + языки
> (`POST /dispatcher/supplementary-announcements { templateId, languages }`). Аудио
> готовится заранее (загрузка/задача 021), TTS **не** на пути запуска. Первый срез —
> один пресет за запуск. Ниже Decision 3 (ad-hoc текст) оставлен как исходный
> замысел; актуальная модель зафиксирована в `domain-model.md`.

3. **Контент — ad-hoc свободный текст диспетчера, озвученный через TTS.** У
   рейсового объявления есть редактируемый `FlightAnnouncementConfig`
   (шаблон × рейс); у аэропортового дополнительного постоянного шаблона нет — это
   разовое сообщение. Диспетчер вводит текст и выбирает языки; core синтезирует
   речь на каждый язык через существующий `SpeechAssetGeneratorInterface`
   (та же машинерия задачи 022, кэш по `hash(text+language+voice+model)`) и
   собирает неизменяемую `AudioSequence` из одного `text`-сегмента на язык.
   * TTS **на пути запуска** здесь допустим (в отличие от рейсового, где синтез —
     пре-шаг сохранения конфига): у дополнительного нет атомарного перехода
     `FlightOccurrence`, который синтез мог бы заблокировать. Недоступность TTS
     блокирует только сам запуск дополнительного объявления (ответ `422`/`503`) и
     не угрожает состоянию рейса. Кэш делает повтор того же текста дешёвым;
   * один и тот же текст на всех выбранных языках в первом срезе (без
     per-language текста). Отдельный текст на язык — возможное расширение.
4. **Точка входа — dispatcher-эндпоинт, отдельный use case.** Запуск —
   действие диспетчера, поэтому
   `POST /api/v1/dispatcher/supplementary-announcements { text, languages }`.
   Новый use case `LaunchSupplementaryAnnouncement`
   (`Announcements\Application`): резолвит текст → `AudioSequence`, создаёт
   `Announcement(type=supplementary)`, сохраняет; пост-коммит публикация
   `RequestAnnouncementPlayback` идёт **тем же** механизмом на `AnnouncementCreated`
   (Decision 2), а не отдельной веткой. Оркестратор `FlightOperations`,
   `FlightOccurrence` и его порт **не участвуют** — дополнительное объявление не
   является действием над рейсом.
5. **Playback не меняется на уровне домена.** `PlaybackJob` уже хранит и
   сортирует по `priority`; `50` естественно встаёт между `100` и `0`.
   Приоритетного **прерывания** здесь нет — supplementary только встаёт в очередь.
   Отдельного среза playback не требуется: приходит обычный
   `RequestAnnouncementPlayback` с `priority: 50`.
6. **Отмена и стоп — бесплатно.** Дополнительное объявление — обычный
   `Announcement`, поэтому кнопки «Убрать» (Pending) и «Стоп» (Playing) на экране
   «Статус» работают без изменений. `Announcement.cancel()` идемпотентен как и
   раньше.
7. **Web — действие «Дополнительное объявление» в панели диспетчера.** Не строка
   рейсовой доски: отдельная кнопка/модалка с `textarea` и пикером языков (языки —
   из `/api/v1/languages`, без привязки к рейсу × типу). Запуск зовёт новый
   dispatcher-эндпоинт, инвалидирует query key очереди; строка появляется в
   drawer «Статус» позади рейсовых.

## Core

* `AnnouncementType`: кейс `Supplementary`; методы `requiresFlight(): bool` и
  `playbackPriority(): int`; `requiresCheckInCounters/requiresGate/isRepeatable`
  → `false` для нового кейса;
* `Announcement`: `flightDefinitionId` nullable; `createPrepared()` — валидация
  «рейсовый тип ⇒ есть `flightDefinitionId`, supplementary ⇒ нет»; ветка без
  стоек/выхода/рейса;
* `AnnouncementCreated.flightDefinitionId: ?string`;
* `LaunchSupplementaryAnnouncement` (command + handler,
  `Announcements\Application`): резолв текста в `AudioSequence` через
  `SpeechAssetGeneratorInterface`/резолвер, `createPrepared`, `save`;
* `PublishAnnouncementPlaybackRequest`: `priority` из
  `announcement.getType()->playbackPriority()` вместо константы;
* HTTP: `POST /api/v1/dispatcher/supplementary-announcements` в dispatcher-
  контроллере Announcements/dispatcher.

## Web

* `dispatcherApi`: `launchSupplementaryAnnouncement({ text, languages })`;
* панель диспетчера: кнопка «Дополнительное объявление» → модалка (textarea +
  языковой пикер), submit-мутация, дизейбл на время запроса, инвалидация query
  key очереди;
* серверные `422/503` (пустой текст, TTS недоступен) показываются в форме.

## Out of Scope

* **Приоритетное прерывание** (преемпция) — задача 025. Сейчас `50` только
  встаёт в очередь и никого не прерывает;
* **Emergency mode** (priority `1000`, глобальная приостановка очереди, agent
  pause/resume, отдельный emergency-транспорт ADR 002 уровень 2) — задача 026.
  Диспозиция прерванного преемпцией job (**новый статус `Preempted` → requeue**)
  зафиксирована для 026 в `domain-model.md` «Открытые решения», здесь не
  реализуется;
* ~~переиспользуемый **каталог** дополнительных объявлений~~ — **втянут в scope**
  (см. Revision выше): пресеты `SupplementaryAnnouncementTemplate` и стали основным
  путём вместо ad-hoc текста;
* **мультивыбор** нескольких пресетов с явным порядком за один запуск (как в старой
  САО) — следующий срез; первый срез запускает один пресет;
* ~~**web-экран администрирования** каталога пресетов~~ — **реализован**: feature
  `supplementary-templates` (`/supplementary-announcements`), таблица + форма с
  языковыми вариантами (язык + аудиофайл), CRUD и активация/деактивация;
* per-language текст (свой текст на каждый язык);
* выбор голоса под дополнительное объявление сверх дефолта;
* фоновая музыка (`priority 0`) — отдельный будущий продюсер.

## Acceptance Criteria

Core:

* `LaunchSupplementaryAnnouncement` создаёт `Announcement(type=supplementary,
  flightDefinitionId=null)` с озвученной `AudioSequence` и публикует
  `RequestAnnouncementPlayback` с `priority: 50` после коммита;
* рейсовые объявления по-прежнему уходят с `priority: 100`
  (маппинг по типу, не регрессия);
* попытка создать рейсовый тип без `flightDefinitionId` отклоняется; supplementary
  с `flightDefinitionId` отклоняется (или код игнорируется — зафиксировать один
  вариант);
* пустой текст → `422`; недоступность TTS → `422`/`503`, `Announcement` не
  создаётся.

Playback:

* задание `priority 50` играет после `priority 100` и перед `priority 0`;
  доменного изменения нет — проверяется существующей логикой отбора.

Web:

* кнопка «Дополнительное объявление» запускает объявление; строка появляется на
  экране «Статус» позади рейсовых; «Убрать»/«Стоп» доступны как у рейсовой строки.

Tests:

* Core: domain — `AnnouncementType::playbackPriority/requiresFlight`,
  `createPrepared` для supplementary и валидация рейсовый⇒flightId; application —
  `LaunchSupplementaryAnnouncement` (создание + приоритет издателя), регрессия
  приоритета `100` для рейсового;
* Web: компонентный — модалка запуска, показ серверной ошибки, инвалидация
  очереди.

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-web
npm run lint && npm run typecheck && npm test
```

Manual QA: запустить рейсовое объявление; пока оно звучит, запустить
дополнительное — строка `50` встаёт в «В очереди» позади играющего рейсового,
играет после его завершения; «Убрать» и «Стоп» работают на строке
дополнительного как на рейсовой.

## Documentation Updates

При реализации:

* `domain-model.md` — раздел `Announcements`: тип `Supplementary`, nullable
  `flightDefinitionId`, аэропортовая (не-рейсовая) принадлежность по типу,
  приоритет по типу; «Открытые решения» — зафиксировать supplementary как
  реализованное и **преемпцию** (новый статус `Preempted` → requeue) как принятое
  решение для будущего emergency (задача 026);
* `README.md` — «Дополнительное объявление» из перечисления типов становится
  реализованным путём;
* `architecture.md` — Core → Playback: `RequestAnnouncementPlayback` несёт
  приоритет по типу объявления (`50` для дополнительного);
* `aeroflow-web/architecture.md` — панель диспетчера: действие запуска
  дополнительного объявления.

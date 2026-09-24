# Экстренный режим (emergency mode, priority 1000)

Status: Done
Bounded context: Announcements (Core — новый тип объявления, пресет, запуск,
активация/деактивация режима), Playback (persistent-состояние режима, применение
преемпции из задачи 025 и приостановка обычной очереди), Audio Execution — без
изменений, dispatcher feature (aeroflow-web)
Service: aeroflow-core, aeroflow-playback, aeroflow-web
Change type: Domain, Application, Integration, Infrastructure, Frontend

## Goal

Дать диспетчеру запустить экстренное объявление максимального приоритета и явно
снять экстренный режим, когда угроза миновала. Пока режим активен, обычная
очередь приостановлена — не только не прерывается точечно (это уже даёт задача
025), но и не начинает новые обычные задания, даже если audio output свободен.

```text
диспетчер: пресет + языки, «Запустить экстренное объявление»
  -> Announcements: Announcement(type=emergency, priority=1000)
  -> RequestAnnouncementPlayback (announcement_playback, без изменений контракта)
  -> ActivateEmergencyMode (новый транспорт emergency_commands, ADR 002 уровень 2)
  -> playback: преемпция звучащего (025) + persistent-флаг «режим активен»
  -> обычная очередь не стартует новые задания, пока флаг не снят
  -> диспетчер: «Деактивировать экстренный режим»
  -> DeactivateEmergencyMode -> playback снимает флаг -> очередь продолжается
```

Это второй не-рейсовый путь объявления после supplementary (024) и последний
нереализованный тип из README («Экстренное объявление» — единственный оставшийся
пункт из «Типы объявлений»).

## Уже существует (используется, не создаётся заново)

* `AnnouncementType`, `Announcement::createPrepared()`, пост-коммит издатель
  `PublishAnnouncementPlaybackRequest` (реагирует на `AnnouncementCreated`,
  берёт `priority` из `getType()->playbackPriority()`) — расширяются кейсом, не
  переписываются (`src/Announcements/Domain/Enum/AnnouncementType.php`,
  `src/Announcements/Application/EventHandler/PublishAnnouncementPlaybackRequest.php`);
* паттерн пресета целиком: `SupplementaryAnnouncementTemplate`,
  `LaunchSupplementaryAnnouncementHandler`, admin CRUD
  (`SupplementaryTemplateController`), web-форма `SupplementaryTemplateForm` —
  зеркалятся для emergency один в один
  (`src/Announcements/Application/LaunchSupplementaryAnnouncement/LaunchSupplementaryAnnouncementHandler.php`);
* `PlaybackJob` с преемпцией и статусом `Preempted` — задача 025, здесь только
  получает реального инициатора (приоритет `1000`);
* `StopAnnouncementPlayback` (019) — диспетчер по-прежнему может точечно
  остановить конкретное звучащее экстренное объявление, не трогая режим;
* ADR 002 — уровень 2 (отдельный транспорт для по-настоящему control-plane
  команд) явно зарезервирован под этот срез.

## Decisions

1. **Контент — пресет, отдельный aggregate root `EmergencyAnnouncementTemplate`**
   (`Announcements\Domain`), параллельный `SupplementaryAnnouncementTemplate`, а
   не его расширение флагом: у emergency-пресета другое поведение запуска (он
   всегда активирует режим целиком, supplementary — никогда), это достаточно
   значимое различие, чтобы не смешивать в одном типе. Форма идентична: имя,
   языковые варианты `{languageCode -> audioAssetId}`, активность; тот же admin
   CRUD-паттерн (контроллер/репозиторий/события Created/Updated/Activated/
   Deactivated), что у `SupplementaryAnnouncementTemplate`.
2. **`AnnouncementType::Emergency`** — новый кейс; `requiresFlight()` → `false`
   (как `Supplementary`); `playbackPriority()` → `1000`; `isRepeatable()` →
   `false` в первом срезе (авто-повтор, пока режим активен, — будущее
   расширение по аналогии с continuation, задача 020; здесь диспетчер при
   необходимости запускает пресет повторно вручную).
3. **Запуск — одно атомарное диспетчерское действие с двумя эффектами.**
   `POST /api/v1/dispatcher/emergency-announcements { templateId, languages }` →
   `LaunchEmergencyAnnouncementHandler` (зеркало
   `LaunchSupplementaryAnnouncementHandler`): резолвит пресет в `AudioSequence`,
   создаёt `Announcement(type=emergency)`, сохраняет. Пост-коммит на
   `AnnouncementCreated` публикует уже существующим механизмом
   `RequestAnnouncementPlayback` (`priority: 1000`, контракт и транспорт не
   меняются — точно то же решение, что и для supplementary в 024: только
   значение поля) **и**, отдельным новым обработчиком того же события (когда
   `type === Emergency`), `ActivateEmergencyMode` через новый порт
   `EmergencyModePublisherInterface`. Одно действие диспетчера атомарно даёт и
   контент, и активацию режима — тот же паттерн, что и в задаче 020, где
   открытие регистрации одним действием атомарно запускает разовое и
   continuation-объявления.
4. **Деактивация — отдельное явное действие, не привязанное к `Announcement`.**
   `POST /api/v1/dispatcher/emergency-announcements:deactivate` публикует
   `DeactivateEmergencyMode`. Ни один `Announcement` не меняется — нейтральная
   операторская команда, как `StopAnnouncementPlayback`/`StopAnnouncementRepeat`.
   Деактивация не воздействует на уже играющее/ожидающее экстренное задание —
   только снимает guard из Decision 7; если нужно прервать конкретное звучащее
   экстренное объявление раньше, диспетчер пользуется существующим Stop (019).
5. **Транспорт — ADR 002, уровень 2.** `ActivateEmergencyMode`/
   `DeactivateEmergencyMode` идут по новому выделенному транспорту
   `emergency_commands` (отдельная очередь Symfony Messenger, свой DSN и
   сериализатор), не по `announcement_playback` — ровно тот случай, который ADR
   держал открытым («ждёт emergency-среза»). `RequestAnnouncementPlayback` (в
   т.ч. с `priority: 1000`) остаётся на `announcement_playback` без изменений:
   это обычное задание, просто дорогое по приоритету, а не control-plane
   сигнал, который не должен стоять в очереди позади bulk-трафика.
6. **Playback хранит режим как persistent singleton.** Новый агрегат
   `EmergencyMode` (`Playback\Domain\Entity`) — одна строка с фиксированным id,
   `status: active|inactive`, `activatedAt`. `HandleActivateEmergencyMode`/
   `HandleDeactivateEmergencyMode` — идемпотентные хендлеры (как
   `HandleCancelAnnouncementPlayback`): повторная активация/деактивация —
   no-op, повторное событие не публикуется. Переживает рестарт процесса (не
   in-memory флаг).
7. **`PlaybackQueueDispatcher::dispatchNext()` получает вторую guard-проверку,
   независимую от преемпции 025.** Пока `EmergencyMode` активен, при выборе
   `next` из `findNextPendingForUpdate()` пропускаются (не стартуют) задания с
   приоритетом ниже преемптивного порога, даже если `findActive() === null`
   (audio output свободен между тиками). Технически — необязательный параметр
   `findNextPendingForUpdate(?int $minPriority = null)` в
   `PlaybackJobRepositoryInterface`, используемый только когда режим активен.
   Это отдельная от 025 ветка: 025 только прерывает уже играющее, 026 ещё и не
   даёт стартовать новому обычному, пока режим активен.
8. **Web.** Диспетчерская панель получает: кнопку «Экстренное объявление»
   (визуально выделенную, по аналогии с `SupplementaryAnnouncementModal`, но
   другим акцентом) с тем же UX «пресет + языки»; persistent-баннер в шапке,
   когда режим активен, с кнопкой «Деактивировать экстренный режим»
   (подтверждение, дизейбл на время запроса, как у Stop/Cancel).
9. **Read-модель `ListPlaybackQueue`.** Строка emergency-объявления показывается
   как обычная (приоритет `1000` сортирует её первой). Прерванная чужая строка
   получает состояние `preempted`, отличное от `interrupted` (019) — диспетчер
   должен видеть разницу «сам остановил» vs «прервано экстренным». Core-приём
   события остаётся generic (по имени события, задача 023), меняется только
   маппинг состояния в read-модели.

## Core

* `AnnouncementType`: кейс `Emergency`; `playbackPriority()` → `1000`;
* `EmergencyAnnouncementTemplate` (aggregate root) + repository + admin
  контроллер `/api/v1/admin/emergency-announcement-templates` (CRUD,
  активация/деактивация) — зеркало `SupplementaryTemplateController`;
* `LaunchEmergencyAnnouncement` (command + handler) — зеркало
  `LaunchSupplementaryAnnouncementHandler`;
* новый event handler на `AnnouncementCreated` (условие `type === Emergency`):
  публикует `ActivateEmergencyMode` через `EmergencyModePublisherInterface`
  (реализация — `emergency_commands` транспорт);
* `DeactivateEmergencyModeHandler` — command handler на прямой HTTP-запрос
  диспетчера (не на доменное событие — нет `Announcement`, который стоило бы
  создавать ради деактивации);
* HTTP: `POST /api/v1/dispatcher/emergency-announcements`,
  `POST /api/v1/dispatcher/emergency-announcements:deactivate` в dispatcher-
  контроллере Announcements/dispatcher;
* `GET /api/v1/dispatcher/emergency-announcements/templates` (активные пресеты,
  зеркало supplementary) и способ узнать, активен ли режим сейчас (поле в
  read-модели очереди или отдельный лёгкий эндпоинт — решить при реализации).

## Playback

* `EmergencyMode` (aggregate) + repository;
* `HandleActivateEmergencyMode`/`HandleDeactivateEmergencyMode` (идемпотентные);
* `emergency_commands` транспорт (Messenger config, сериализатор с `command`-
  дискриминатором по образцу `AnnouncementPlaybackSerializer`);
* `PlaybackQueueDispatcher::dispatchNext()` — guard по активному режиму
  (Decision 7), поверх уже готовой преемпции (025);
* `AnnouncementPlaybackPreempted`/receipt-приём в core не меняется (023-стиль
  generic по имени события).

## Web

* `dispatcherApi`: `launchEmergencyAnnouncement({ templateId, languages })`,
  `deactivateEmergencyMode()`, запрос списка активных пресетов;
* панель диспетчера: кнопка «Экстренное объявление» → модалка (пресет + языки),
  submit-мутация, дизейбл на время запроса, инвалидация query key очереди;
  persistent-баннер режима с «Деактивировать» при активном режиме;
* admin-feature `emergency-templates` (по аналогии с `supplementary-templates`):
  таблица + форма с языковыми вариантами, CRUD, активация/деактивация;
* drawer «Статус»: новое состояние строки `preempted`, отличное от `interrupted`.

## Out of Scope

* Авто-повтор экстренного объявления, пока режим активен (аналог continuation,
  020) — future;
* Мультивыбор нескольких пресетов за один запуск — как и в 024, следующий срез;
* Таймаут/авто-деактивация режима — только явное действие диспетчера;
* Несколько audio output / несколько агентов — не меняется;
* Recovery зависшего режима после падения playback-процесса между Activate и
  Deactivate — состояние переживает рестарт (persistent), но отдельного
  мониторинга/алертинга нет.

## Acceptance Criteria

Core:

* `LaunchEmergencyAnnouncement` создаёт `Announcement(type=emergency)` с
  `priority: 1000` и атомарно (в рамках пост-коммит реакции на одно
  `AnnouncementCreated`) публикует и `RequestAnnouncementPlayback`, и
  `ActivateEmergencyMode`;
* деактивация публикует `DeactivateEmergencyMode`, не меняя ни один
  `Announcement`;
* пресет неактивен/не найден/asset недоступен → `404`/`422`, как у
  supplementary.

Playback:

* `ActivateEmergencyMode` идёт по `emergency_commands`, не по
  `announcement_playback`, и обрабатывается независимо от очереди обычных
  сообщений (проверяется на транспортном уровне);
* пока режим активен, обычное (не преемптивное) Pending-задание не стартует
  даже при свободном audio output;
* активный emergency-job преемптирует звучащее обычное (проверка регрессии
  025 в интеграции);
* деактивация снимает guard — следующее обычное Pending-задание стартует;
* повторная активация/деактивация — идемпотентны, без дублирования событий.

Web:

* кнопка «Экстренное объявление» запускает объявление и включает баннер режима;
* «Деактивировать» доступна только при активном режиме, снимает баннер после
  подтверждения playback;
* строка преемптированного объявления в drawer отличима от штатного «Стоп».

Tests:

* Core: domain (`AnnouncementType::playbackPriority`, `EmergencyAnnouncementTemplate`
  инварианты), application (`LaunchEmergencyAnnouncement`, деактивация,
  регрессия приоритетов `100`/`50`);
* Playback: application (`EmergencyMode` активация/деактивация идемпотентность,
  guard в `dispatchNext()`, интеграция с 025 преемпцией), serializer-тест
  нового транспорта;
* Web: компонентные — модалка запуска, баннер режима, деактивация.

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-playback
php bin/phpunit

# aeroflow-web
npm run lint && npm run typecheck && npm test
```

Manual QA: запустить рейсовое объявление; пока оно звучит, запустить экстренное
— рейсовое уходит в «Прервано экстренным» и требует
позже повторного запуска диспетчером; экстренное играет первым; баннер режима
активен; запустить второе обычное — не стартует, пока не активна деактивация;
деактивировать — обычная очередь продолжается.

## Уточнения, принятые при реализации

1. **Состояние режима возвращается в core обратным событием.** Открытый в
   Decision 9 / Core вопрос («поле в read-модели очереди или отдельный лёгкий
   эндпоинт») решён так: playback после смены состояния публикует на **ту же**
   очередь `playback_events` новое семейство фактов —
   `emergency_mode.activated` / `emergency_mode.deactivated` (это и есть
   `EmergencyModeActivated`/`EmergencyModeDeactivated` из `architecture.md`). Они
   не несут ни `announcementId`, ни `jobId` (режим глобален), поэтому core
   разбирает их отдельной веткой сериализатора в собственное сообщение и держит
   зеркало состояния одной строкой (`emergency_mode_record`). Диспетчер читает
   его через `GET /api/v1/dispatcher/emergency-announcements/mode`. Так баннер
   показывает то, что **действительно** включилось в playback, и корректен для
   диспетчера, который открыл панель уже во время режима или работает на другом
   рабочем месте.
2. **`preempted` — нетерминальное состояние строки очереди.** Задача 025
   возвращает прерванное задание в `Pending` с сохранением приоритета и
   `createdAt`, поэтому строка остаётся в секции «В очереди» с подписью
   «Прервано экстренным» и прозвучит сама после снятия режима. Строка черновика в
   Manual QA («требует позже повторного запуска диспетчером») этому противоречила
   и не соответствует поведению 025 — верно именно автоматическое
   до-воспроизведение.
3. **Отдельный worker, а не только отдельная очередь.** Изоляция уровня 2 ADR 002
   имеет смысл, только если команду кто-то немедленно обрабатывает, поэтому в
   compose playback появился второй контейнер
   (`aeroflow_playback_emergency_worker`), слушающий исключительно
   `emergency_commands`.
4. **Web переиспользует экраны пресетов, а не копирует их.** Модалка запуска,
   форма пресета и экран каталога вынесены в общие компоненты
   (`PresetAnnouncementModal`, feature `announcement-presets`), а
   supplementary- и emergency-фичи остались тонкими обёртками со своими api,
   ключами и мутациями. Backend при этом зеркалится один в один, как решено в
   Decision 1 — там дублируются доменные инварианты, а не верстка.
5. **`GET .../emergency-announcements/mode`** живёт в том же dispatcher-контроллере,
   что запуск и деактивация, вместо отдельного эндпоинта `/dispatcher/emergency-mode`.

## Documentation Updates

При реализации:

* `domain-model.md` — раздел `Announcements`: тип `Emergency`,
  `EmergencyAnnouncementTemplate`; раздел `PlaybackJob`/новый `EmergencyMode`:
  persistent-режим, guard в диспетчере очереди; «Открытые решения» — emergency
  mode помечается реализованным;
* `README.md` — «Экстренное объявление» из перечисления типов становится
  реализованным путём; убрать из списка нереализованного;
* `architecture.md` — Core → Playback: `ActivateEmergencyMode`/
  `DeactivateEmergencyMode` на новом транспорте; Playback → Core:
  `EmergencyModeActivated`/`EmergencyModeDeactivated` (если решено публиковать
  подтверждение обратно — уточнить при реализации);
* ADR 002 — статус меняется на «уровень 2 реализован задачей 026»;
* `aeroflow-web/architecture.md` — панель диспетчера: действие запуска
  экстренного объявления, баннер режима, admin-экран пресетов.

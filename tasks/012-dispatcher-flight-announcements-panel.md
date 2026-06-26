# Dispatcher Flight Announcements Panel

Status: Done  
Bounded context: Flight Operations, Announcements, Localization  
Service: aeroflow-web, aeroflow-core  
Change type: Frontend, small Application/API addition

## Goal

Сделать первый рабочий интерфейс диспетчера для запуска рейсовых объявлений.

Административный интерфейс уже отвечает за настройку рейсов (`FlightDefinition`),
конфигураций объявлений и справочников. Новый экран должен быть легче и ближе к
старой системе: единая рабочая панель, где тип объявления выбирается кнопочным
фильтром, а список рейсов и форма запуска перестраиваются внутри того же экрана.

Задача опирается на уже реализованную core-задачу `011-flight-occurrences-core.md`
(Status: Done). Доменный срез `FlightOccurrence`, action-эндпоинты и dispatcher
read model уже существуют. Эта задача — преимущественно frontend плюс одно
небольшое application/API-дополнение в core (см. `Backend Scope`).

## Ключевая модель экрана

Диспетчер **думает карточками рейсов (`FlightDefinition`)**, а не операционными
occurrence. `FlightOccurrence` — это деталь реализации состояния: он «рождается»
при первом действии диспетчера над карточкой и дальше хранит статус регистрации /
посадки / прибытия.

```text
Диспетчер видит активные FlightDefinition на операционный день
  ↓
Выбирает режим объявления (кнопочный фильтр)
  ↓
Видит только подходящие под режим рейсы
  ↓
Выбирает рейс и уточняет минимальные параметры
  ↓
Запускает объявление
  ↓
Core гарантирует FlightOccurrence для карточки+даты (ensure),
выполняет переход и создаёт Announcement в одной транзакции
```

Источник списка — каталог `FlightDefinition`. Источник **статуса** — сегодняшний
`FlightOccurrence`, если он уже создан; если occurrence ещё нет, рейс считается в
состоянии «не начат» (эквивалент `scheduled`).

Это сознательно отличается от черновика задачи: доска не строится из заранее
существующих occurrence, потому что расписания (`FlightSchedule`) ещё нет и
occurrence некому создавать заранее.

### Legacy reference

`aeroflow-docs/legacy-sao-user-guide.md` описывает рабочую панель старой САО и
подтверждает ключевые решения этой задачи: кнопочный action-first интерфейс,
карточки рейсов, перемещающиеся между окнами действий по статусу, и
автоматический (не ручной) характер продолжения регистрации. Этот гайд **нужно
учитывать** при проектировании UX, но он **не 100% правило, а материал для
обсуждения**: при противоречии приоритет у `README.md`, `architecture.md` и
`domain-model.md`.

## Architectural Classification

Задача затрагивает:

* `Flight Operations` — чтение `FlightDefinition` и `FlightOccurrence`, идемпотентное
  создание manual occurrence, переходы статуса;
* `Announcements` — создание рейсового `Announcement` (уже происходит внутри
  action use case, отдельной работы не требует);
* `Localization` — выбор языков объявления из справочника;
* `Audio Catalog` — только косвенно, через готовность конфигурации при запуске.

Затрагиваемые агрегаты:

* `FlightDefinition` — карточка рейса, источник доски и постоянных данных;
* `FlightOccurrence` — операционный рейс и источник текущего статуса;
* `FlightAnnouncementConfig` — косвенно, как предусловие готовности при запуске;
* `Announcement` — создаётся при запуске action.

`aeroflow-playback` и `aeroflow-agent` не затрагиваются. Если создание
`Announcement` уже публикует integration command в playback, dispatcher UI
использует существующее поведение без отдельной логики воспроизведения.

## Product Principle

Панель диспетчера не дублирует админку.

Отвечает на вопросы:

```text
Какое объявление диспетчер хочет запустить сейчас?
Какие рейсы подходят для выбранного объявления?
Какие минимальные данные нужно уточнить перед запуском?
```

Не отвечает на вопросы:

```text
Как устроен шаблон?
Какие сегменты входят в вариант?
Как настроить аудиоассеты?
```

## Action Set

Диспетчер работает с **четырьмя** действиями, ровно как умеет core
(`011`, action-эндпоинты):

| Кнопка-фильтр            | announcementType        | direction | требуемый статус occurrence            |
| ------------------------ | ----------------------- | --------- | -------------------------------------- |
| Начало регистрации       | `check_in_opening`      | departure | occurrence отсутствует или `scheduled` |
| Окончание регистрации    | `check_in_closing`      | departure | `check_in_open`                        |
| Посадка                  | `boarding_invitation`   | departure | `check_in_closed`                      |
| Прибытие                 | `arrival`               | arrival   | occurrence отсутствует или `scheduled` |

**«Продолжение регистрации» (`check_in_continuation`) в этой задаче исключено.**
В core нет соответствующего действия, а по доменной модели продолжение — это
будущий авто-повтор (process manager по интервалу), а не ручное действие
диспетчера. Кнопки «Продолжение регистрации» на панели нет.

Первичные действия (`check_in_opening`, `arrival`) допустимы для карточек, у
которых occurrence на день ещё не создан. Остальные действия требуют уже
существующего occurrence в нужном статусе.

## User Experience

Экран компактный и пригодный для повторяющейся работы.

### Action-First Workflow

```text
Кнопочные фильтры действий сверху:
  Начало регистрации
  Окончание регистрации
  Посадка
  Прибытие

Контекст дня:
  выбор операционной даты (по умолчанию сегодня)

Общий список рейсов ниже:
  только рейсы, подходящие под выбранный фильтр
  поиск по номеру рейса, маршруту, авиакомпании

Панель запуска:
  выбранный рейс
  короткая форма параметров текущего действия
  одна primary-кнопка запуска
```

Переключение типа объявления не уводит на другую страницу и сразу перестраивает
список. Это единое рабочее пространство, а не набор отдельных экранов.

### Layout

```text
Верхняя панель:
  кнопочный фильтр из 4 действий + выбранная операционная дата

Основная область:
  только подходящие рейсы
  поиск по номеру рейса, направлению, авиакомпании
  компактные карточки или плотный список

Панель запуска справа (desktop) или снизу (mobile/tablet):
  выбранный рейс
  выбранное действие
  минимальная форма параметров
  кнопка запуска

Служебная зона:
  статус результата последнего запуска (announcementId или ошибка)
```

### Flight Card

Показывать только диспетчерски полезное:

* номер рейса;
* маршрут: город — название аэропорта (CODE), направление departure/arrival;
* авиакомпания, если доступна;
* текущий операционный статус (из occurrence или «не начат»).

Названия аэропортов берутся через уже существующую web-feature `airports`
(как в `flight-definitions`), потому что dispatcher read model возвращает в поле
`airportName` сам код, а не название. Формат — `Город — Название аэропорта (CODE)`.

Не показывать UUID, segment details, admin-only controls.

### Eligibility и недоступные рейсы

Внутри выбранного действия показывать только подходящие рейсы (см. таблицу в
`Action Set`). Eligibility v1 определяется **только по lifecycle-статусу**
(direction + статус occurrence + активность карточки).

Готовность конфигурации (наличие активного `FlightAnnouncementConfig`, языковых
вариантов, резолва `AudioSequence`) на этом этапе **не вычисляется заранее**:
dispatcher read model её не считает (`availableLanguages` приходит пустым). Если
конфигурация не готова, запуск вернёт `422` и причина показывается у формы (см.
`Launch Flow`). Предварительное обогащение readiness на backend — отдельный
follow-up, в этой задаче не делается.

Опциональный переключатель «показать недоступные» может выводить рейсы, не
подходящие по статусу, с человекочитаемой причиной из read model
(`unavailableReason`), без внутренних exception names.

### Minimal Parameters

Общие:

* `languages` — `MultiSelect` из read-only справочника `/api/v1/languages`;
* порядок выбора языков сохраняется в payload;
* по умолчанию — первый активный язык.

Для `check_in_opening`:

* выбор одной или нескольких стоек регистрации из активного справочника
  `CheckInCounter` (`checkInCounterIds`).

Для `check_in_closing`:

* дополнительных параметров нет (объявление окончания регистрации);
  стойки не требуются.

Для `boarding_invitation`:

* выбор одного выхода из активного справочника `Gate` (`gateId`).

Для `arrival`:

* дополнительных параметров нет.

## Backend Scope

Большая часть бэкенда уже готова в `011`. Переиспользуются как есть:

```text
GET  /api/v1/flight-definitions?active=true&direction=...   (доска: список карточек)
GET  /api/v1/dispatcher/flight-occurrences?operationalDate=...  (наложение статуса)
POST /api/v1/flight-occurrences/{id}/check-in:open
POST /api/v1/flight-occurrences/{id}/check-in:close
POST /api/v1/flight-occurrences/{id}/boarding
POST /api/v1/flight-occurrences/{id}/arrival
GET  /api/v1/languages
GET  /api/v1/admin/check-in-counters?active=true
GET  /api/v1/admin/gates?active=true
```

Формы ответов (фактические, проверены по коду):

* dispatcher read model item: `{ id, flightDefinitionId, flightNumber, direction,
  airportCode, airportName, operationalDate, status, eligible, unavailableReason,
  availableLanguages }` — плоский объект, без вложенного `announcementReadiness[]`;
* launch response: `{ success, data: { occurrence: FlightOccurrenceResult,
  announcementId } }`;
* gate / counter item: `{ id, code, displayName, sortOrder, active, createdAt,
  updatedAt }`.

### Единственное backend-дополнение: EnsureManualFlightOccurrence

Action-эндпоинты требуют `flightOccurrenceId`, а доска definition-driven: для
первичных действий occurrence ещё не существует. Текущий `POST /flight-occurrences`
на дубликат бизнес-ключа кидает `409` (см.
`CreateFlightOccurrenceHandler`), идемпотентного «ensure» нет.

Добавить идемпотентный use case `EnsureManualFlightOccurrence`, который возвращает
существующий occurrence по бизнес-ключу `flightDefinitionId + operationalDate +
source=manual + sequenceNumber=1` или создаёт новый:

```http
POST /api/v1/flight-occurrences:ensure-manual
```

```json
{ "flightDefinitionId": "uuid", "operationalDate": "2026-06-26" }
```

Response: `200` с `FlightOccurrenceResult` (существующий или только что созданный).
Этот use case уже предусмотрен в `011` как опция и является линчпином
definition-driven потока. Без него frontend пришлось бы делать хрупкую
последовательность `POST → 409 → GET по фильтру → action`, уязвимую к гонке и
двойному клику.

Альтернатива (если backend-дополнение нежелательно): оставить ensure целиком на
frontend через `POST /flight-occurrences` + обработку `409` + `GET
/flight-occurrences?flightDefinitionId=...&operationalDate=...`. Не рекомендуется.

### Backend Requirements

* `EnsureManualFlightOccurrence` идемпотентен по бизнес-ключу и не создаёт
  дубликатов при повторном вызове;
* недопустимый переход (например, закрытие регистрации до открытия) по-прежнему
  отклоняется в action use case до создания `Announcement` (уже реализовано);
* неготовая конфигурация (нет config/asset/prompt/языкового варианта) возвращает
  `422` и ничего не персистит (уже реализовано как предусловие перехода);
* ошибки запуска возвращаются в виде, пригодном для показа рядом с формой;
* покрыть application/API-тестами новый ensure use case.

`POST /api/v1/announcements` для рейсовых типов не используется (деприкейтнут в
`011`). Запуск идёт только через action-эндпоинты `FlightOccurrence`.

## Frontend Scope

Добавить dispatcher feature в `aeroflow-web`. Структура по аналогии с
существующими features (`flight-definitions`, `airports`):

```text
src/pages/dispatcher/DispatcherPage.tsx
src/features/dispatcher/api/dispatcherApi.ts
src/features/dispatcher/api/dispatcherKeys.ts
src/features/dispatcher/model/types.ts
src/features/dispatcher/model/board.ts            # merge definitions + occurrences, eligibility
src/features/dispatcher/hooks/useDispatcherBoard.ts
src/features/dispatcher/hooks/useLaunchAnnouncement.ts
src/features/dispatcher/ui/DispatcherActionFilter.tsx
src/features/dispatcher/ui/DispatcherFlightList.tsx
src/features/dispatcher/ui/DispatcherLaunchPanel.tsx
```

Переиспользовать существующие features:

* `airports` — названия аэропортов для карточек;
* `languages` — `MultiSelect` языков;
* `flight-definitions` API — список активных карточек для доски.

Добавить недостающие read-only API-клиенты для справочников выбора:

* `/api/v1/admin/check-in-counters?active=true`;
* `/api/v1/admin/gates?active=true`.

(на web этих фич ещё нет; достаточно тонких api+hook без CRUD-UI).

### Board composition (frontend)

Доска собирается слиянием на клиенте:

1. `GET /flight-definitions?active=true` (+ фильтр direction по выбранному действию);
2. `GET /dispatcher/flight-occurrences?operationalDate=<date>` — статусы уже
   начатых рейсов на день;
3. merge по `flightDefinitionId`: карточке без occurrence присваивается статус
   «не начат»; карточке с occurrence — его `status`;
4. фильтрация по выбранному действию согласно таблице `Action Set` (тонкая
   lifecycle-логика, допустимо для v1).

### Routing

Добавить защищённый route:

```text
/dispatcher
```

Доступен авторизованным пользователям. Ролевое разграничение (диспетчер/админ) —
follow-up: сейчас в роутере нет per-role gating, вводить его в этой задаче не
требуется. Добавить пункт в навигацию `AppLayout`.

### Launch Flow

```text
выбран рейс + действие + параметры
  ↓
ensureOccurrence(flightDefinitionId, operationalDate)  → occurrenceId
  ↓
POST /flight-occurrences/{occurrenceId}/<action>  с payload действия
  ↓
успех: показать announcementId/статус, инвалидация доски и статусов дня
ошибка 422/409: показать причину рядом с формой, параметры не теряются
```

Payload действия (поля зависят от типа), фактический контракт:

```jsonc
// check-in:open
{ "languages": ["ro-MD", "ru"], "checkInCounterIds": ["uuid-1", "uuid-3"] }

// check-in:close
{ "languages": ["ro-MD"] }

// boarding
{ "languages": ["ro-MD"], "gateId": "uuid" }

// arrival
{ "languages": ["ro-MD"] }
```

`idempotencyKey` в текущем `LaunchOccurrenceAnnouncementRequest` не принимается.
Защита от двойного клика на v1 — дизейбл primary-кнопки на время мутации.
Серверный idempotency key — отдельный follow-up.

После успешного запуска инвалидировать query-ключи доски и dispatcher-occurrences
за выбранную дату, чтобы рейс переместился между фильтрами без ручной правки.

### UI States

* загрузка доски;
* пустой список активных карточек;
* выбранный режим без подходящих рейсов;
* недоступное действие с причиной (если включён показ недоступных);
* валидационная ошибка формы;
* успешный запуск (announcementId / краткий статус);
* ошибка сети или 5xx без потери введённых параметров.

### Design Requirements

* избегать больших административных таблиц; плотные карточки/список;
* одна primary-кнопка на выбранный тип объявления;
* экстренные/дополнительные сценарии не смешивать с рейсовыми действиями;
* не показывать editor controls для template segments;
* не использовать marketing-style hero layout.

## Out of Scope

* `check_in_continuation` и авто-повторы продолжения регистрации;
* `FlightSchedule` и автоматическое создание occurrence по расписанию;
* предварительный расчёт config-readiness и `availableLanguages` в read model;
* ролевое разграничение доступа к `/dispatcher`;
* серверный idempotency key для запуска;
* queue/playback control screen;
* экстренные и дополнительные объявления;
* настройка шаблонов и аудиоассетов;
* создание/редактирование `FlightDefinition` и справочников;
* отмена occurrence (`cancelled`) из панели;
* audit действий диспетчера;
* realtime-обновления;
* интеграция с табло.

## Acceptance Criteria

Backend:

* `POST /api/v1/flight-occurrences:ensure-manual` идемпотентно возвращает
  существующий или создаёт новый manual occurrence для карточки+даты;
* повторный вызов с тем же бизнес-ключом не создаёт дубликат и возвращает тот же
  occurrence;
* недопустимый переход возвращает `409/422` и не создаёт `Announcement`
  (существующее поведение, покрыто);
* неготовая конфигурация возвращает `422` и ничего не меняет (существующее);
* добавлен application/API-тест на ensure use case.

Frontend:

* `/dispatcher` открывает рабочий экран диспетчера, есть пункт в навигации;
* 4 действия отображаются кнопочным фильтром внутри одного экрана;
* переключение фильтра сразу перестраивает общий список подходящих рейсов;
* доска строится из активных `FlightDefinition` с наложением статуса occurrence
  за выбранную дату;
* диспетчер находит и выбирает рейс без перехода между страницами;
* маршрут показывается с названием аэропорта (через feature `airports`), не голым
  кодом;
* языки выбираются из справочника, стойки и выход — из активных справочников;
* запуск выполняет ensure occurrence → action и создаёт `Announcement`;
* успешный запуск показывает подтверждение и `announcementId`/статус, доска
  обновляется;
* ошибки `422/409` показываются рядом с формой, введённые параметры не теряются;
* экран не содержит controls администрирования конфигураций.

## Verification

Backend:

```bash
php bin/phpunit tests/Unit tests/Application
docker compose exec -T aeroflow_core_app php bin/phpunit tests/Functional
```

Frontend:

```bash
npm test
npm run lint
npm run build
```

Manual QA:

```text
1. Войти, открыть /dispatcher (дата по умолчанию — сегодня).
2. Нажать «Начало регистрации»: видны активные departure-карточки без начатой
   регистрации.
3. Запустить открытие регистрации с двумя стойками и одним языком; убедиться, что
   создан occurrence и Announcement, рейс ушёл из фильтра «Начало регистрации».
4. Нажать «Окончание регистрации»: виден тот же рейс (статус check_in_open).
   Запустить окончание регистрации.
5. Нажать «Посадка» после закрытия регистрации, выбрать выход, запустить.
6. Нажать «Прибытие»: видны активные arrival-карточки; запустить прибытие.
7. Проверить рейс с выключенной/неготовой конфигурацией: запуск возвращает 422,
   причина показана у формы, статус не изменился.
8. Повторно запустить то же первичное действие (двойной клик): дубликат occurrence
   не создаётся.
```

## Documentation Updates

После реализации:

* `aeroflow-web/architecture.md` — описать dispatcher как третий реализованный
  срез `Flight Operations` (definition-driven доска, ensure occurrence при
  запуске, lifecycle-only eligibility v1);
* при добавлении ensure use case отметить его в `domain-model.md` рядом с
  `FlightOccurrence` (idempotent manual creation).

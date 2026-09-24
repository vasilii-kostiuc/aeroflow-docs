# Server-Side Operational Day

Status: Done  
Bounded context: Flight Operations  
Service: aeroflow-core, aeroflow-web  
Change type: Application/API change + Read model + small Frontend

## Goal

Операционную дату нового ручного `FlightOccurrence` определяет **сервер** по
часам аэропорта, а не диспетчер и не браузер. Доска диспетчера показывает рейсы
текущего операционного дня и **не теряет** рейс, начатый до полуночи и ещё не
доведённый до финального статуса.

## Проблема

`operationalDate` — атрибут конкретного occurrence и часть его бизнес-ключа
(`flightDefinitionId + operationalDate + source + sequenceNumber`). До этой задачи
её выбирал клиент:

* панель диспетчера держала поле «Операционная дата», значение которого уходило в
  `ensure-manual` / `start-next-manual`;
* значение по умолчанию вычислялось в браузере как UTC-дата
  (`toISOString().slice(0, 10)`), поэтому в часовом поясе аэропорта (UTC+2/+3) с
  полуночи до 02:00–03:00 панель открывалась на *вчерашней* дате;
* ручной выбор другой даты создавал occurrence «на завтра», хотя объявление
  звучит сейчас; законного сценария для этого у диспетчера нет;
* core дату не сверял ни с чем.

Одновременно доска фильтровала occurrence строго по одной дате. Рейс, у которого
регистрацию открыли в 23:30, после полуночи пропадал с доски: карточка выглядела
«не начатой», закрыть регистрацию было нельзя, а «Начало регистрации» создавало бы
второй прогон. Единственным обходом было ручное переключение даты назад.

## Ключевые решения

### Операционный день

```text
Операционный день = календарная дата по часовому поясу аэропорта в момент действия.
```

* часовой пояс аэропорта — конфигурация core (`AIRPORT_TIMEZONE`, IANA-имя,
  по умолчанию `Europe/Chisinau`);
* вычисление инкапсулировано портом `OperationalDayProviderInterface`
  (`FlightOperations\Application\OperationalDay`), реализация — в
  `FlightOperations\Infrastructure`; тесты подставляют фиксированный день;
* граница суток — полночь; настраиваемый сдвиг границы (например, 04:00) не
  вводится — перенос незавершённых прогонов (ниже) закрывает практический случай.

В ручном режиме `operationalDate` означает «операционный день, в который прогон
начат». При появлении `FlightSchedule` идентичность occurrence пересматривается в
пользу времени вылета (см. `domain-model.md`, «Открытые решения») — это решение
тому не противоречит.

### Создание manual-occurrence

`EnsureManualFlightOccurrence` и `StartNextManualFlightOccurrence` **не принимают**
`operationalDate`: оба используют текущий операционный день. Клиент шлёт только
`flightDefinitionId`.

`EnsureManualFlightOccurrence` при создании **нового** первого прогона дня
отклоняет запрос `409`, если у карточки есть **незавершённый прогон предыдущего
операционного дня** (`check_in_open` / `check_in_closed`): у карточки не может
быть двух живых прогонов, и сначала надо довести начатый. Возврат уже
существующего occurrence (идемпотентная ветка) не меняется.

### Незавершённый (in-progress) прогон

```text
in-progress = прогон начат (вышел из scheduled), но не достиг финального статуса
            = check_in_open | check_in_closed
```

Прилёт (`scheduled -> arrival_announced`) не имеет промежуточного статуса и
поэтому никогда не бывает in-progress. `scheduled`-прогон — лишь созданная запись
без действия; через полночь он не переносится.

### Доска диспетчера

Новый read-endpoint:

```http
GET /api/v1/dispatcher/board
```

```json
{
  "operationalDate": "2026-09-24",
  "occurrences": [ /* DispatcherFlightOccurrenceResult */ ]
}
```

На каждую карточку — не более одного «текущего» occurrence:

1. незавершённый прогон **предыдущего** операционного дня, если есть;
2. иначе последний (по `sequenceNumber`) прогон текущего операционного дня;
3. иначе ничего — карточка «не начата».

Перенос ограничен одним днём назад: более старые незавершённые прогоны считаются
брошенными и на доску не попадают (явного действия отмены пока нет).

Существующий `GET /api/v1/dispatcher/flight-occurrences?operationalDate=` не
меняется и остаётся фильтром по произвольной дате.

## Architectural Classification

Затрагивает только `Flight Operations`. Агрегат `FlightOccurrence` не меняется:
статусы, переходы и бизнес-ключ прежние. Новые domain events не вводятся.
`Announcements`, `aeroflow-playback`, `aeroflow-agent` не затрагиваются.

## Backend Scope

* `OperationalDayProviderInterface` + реализация по `AIRPORT_TIMEZONE`;
* `EnsureManualFlightOccurrence`: дата — из провайдера; guard на незавершённый
  прогон предыдущего дня при создании;
* `StartNextManualFlightOccurrence`: дата — из провайдера;
* request-DTO `ensure-manual` / `start-next-manual` без `operationalDate`
  (лишнее поле в теле игнорируется);
* `GetDispatcherBoard` query + `GET /api/v1/dispatcher/board`.

## Frontend Scope

* убрать поле «Операционная дата» с панели диспетчера. Текущий операционный день
  отдельно не показывается: диспетчер работает «сейчас», а единственный случай,
  где день важен, — рейс через полночь — помечен на самой карточке;
* доска строится из `GET /dispatcher/board` и перезапрашивается поллингом, чтобы
  смена суток и действия с других рабочих мест подхватывались без перезагрузки;
* перенесённая карточка помечается датой своего прогона («с 23.09»);
* `ensure-manual` / `start-next-manual` вызываются только с `flightDefinitionId`.

## Out of Scope

* настраиваемая граница операционных суток;
* явное завершение/отмена брошенного прогона из панели;
* `sequenceNumber` в запросе `ensure-manual` (остаётся как было);
* часовой пояс в `FlightSchedule` — появится вместе с расписанием.

## Acceptance Criteria

Backend:

* `ensure-manual` / `start-next-manual` создают occurrence на текущий операционный
  день аэропорта независимо от присланного клиентом значения;
* `ensure-manual` отвечает `409`, если у карточки есть незавершённый прогон
  предыдущего дня, и ничего не создаёт;
* `GET /dispatcher/board` возвращает текущий операционный день и по одному
  occurrence на карточку по правилу выше; незавершённый вчерашний прогон виден,
  завершённый вчерашний и `scheduled`-вчерашний — нет;
* покрыто application-, integration- и functional-тестами.

Frontend:

* на панели нет поля выбора даты;
* рейс, начатый до полуночи, после полуночи остаётся на доске с пометкой даты, и
  по нему можно закрыть регистрацию и начать посадку;
* lint, typecheck, tests, build зелёные.

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application
docker compose exec -T aeroflow_core_app php bin/phpunit tests/Functional

# aeroflow-web
npm run lint && npm run typecheck && npm run test:run && npm run build
```

Сценарий полуночи целиком (открыть регистрацию до полуночи → после полуночи
прогон остаётся на доске со своей датой, новый прогон отклоняется `409` →
закрыть регистрацию и начать посадку → новый прогон уже на новый день)
автоматизирован в `tests/Functional/FlightOperations/Api/MidnightCarryOverTest.php`:
в test-окружении порт операционного дня подменён
`Tests\Support\FrozenOperationalDayProvider`, который тест «переводит» через
полночь. Правила выбора текущего прогона на реальном PostgreSQL проверяет
`tests/Integration/.../DbalDispatcherBoardQueryTest.php`.

Manual QA (web):

```text
1. Открыть /dispatcher: поля даты нет.
2. Открыть регистрацию рейса. Сдвинуть операционный день (AIRPORT_TIMEZONE или
   дата в БД occurrence на вчера) — карточка остаётся под «Окончание регистрации»
   с пометкой «с <дата>»; закрыть регистрацию и начать посадку.
3. После посадки карточка снова доступна под «Начало регистрации»; запуск создаёт
   прогон уже на текущий день.
```

## Documentation Updates

* `domain-model.md`, `FlightOccurrence` → «Создание occurrence»: операционный день
  определяет сервер; перенос незавершённого прогона; guard в
  `EnsureManualFlightOccurrence`;
* `aeroflow-web/architecture.md`: dispatcher-срез — доска из
  `/dispatcher/board`, без выбора даты;
* `RUNNING.md`: переменная `AIRPORT_TIMEZONE`.

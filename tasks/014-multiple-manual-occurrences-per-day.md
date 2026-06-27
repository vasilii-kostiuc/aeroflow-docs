# Multiple Manual Flight Occurrences Per Day

Status: Ready  
Bounded context: Flight Operations  
Service: aeroflow-core, aeroflow-web  
Change type: Domain invariant + Application/API addition + small Frontend

## Goal

Дать диспетчеру возможность запустить **новый операционный рейс по той же
карточке `FlightDefinition` в тот же день**, после того как предыдущий
`FlightOccurrence` прошёл свой цикл — достиг финального статуса действия
(`boarding` для вылета, `arrival_announced` для прилёта), либо терминального
`completed`/`cancelled`.

> Важно: в текущем ручном потоке `completed`/`cancelled` **недостижимы** (нет
> действия завершения/отмены); доковый переход `... -> boarding -> completed` в
> коде не реализован. Поэтому фактическим триггером нового прогона служит
> финальный статус действия цикла — `boarding`/`arrival_announced`. Терминальные
> статусы включены в правило на будущее, когда появится действие завершения.

Сейчас панель диспетчера definition-driven и обеспечивает occurrence через
идемпотентный `EnsureManualFlightOccurrence` с фиксированным
`sequenceNumber=1` (см. `012`). Это означает фактически **один запуск рейса в
день**: после завершения карточка «застревает», и второй прогон того же рейса
(вторая ротация, повторный рейс) сделать нельзя.

Доменная модель это уже допускает: бизнес-ключ occurrence —
`flightDefinitionId + operationalDate + source + sequenceNumber`, то есть
несколько occurrence за дату различаются `sequenceNumber`. Не хватает только
use case, который выделяет следующий `sequenceNumber`, и UX-входа в него.

## Не делаем

Не «оживляем» прошедший цикл occurrence. Инвариант `завершённый или отменённый
occurrence нельзя менять` остаётся в силе, lifecycle линейный. Повторный прогон —
это **новый** occurrence с бóльшим `sequenceNumber`, а не возврат старого назад по
циклу (например из `boarding` в `check_in_open`).

## Ключевое решение: серверный инкремент + state-guard

`sequenceNumber` назначает **сервер**, а не клиент. Клиент шлёт только
`flightDefinitionId` и `operationalDate`.

Защита от двойного нажатия строится не на синтетическом ключе, а на **доменном
состоянии**:

```text
Новый запуск разрешён, только если последний occurrence карточки за дату
достиг финального статуса цикла (boarding / arrival_announced, либо
completed / cancelled). Иначе — 409.
```

Это автоматически даёт защиту от двойного клика, без idempotency key:

* клик 1 → создаётся `sequenceNumber = max + 1`, статус `scheduled` (не финальный);
* клик 2 → последний occurrence ещё не достиг финального статуса → `409 Conflict`.

Так дедупликацию обеспечивает само состояние, а правило совпадает с интуицией
диспетчера «новый прогон только после завершения предыдущего».

Рассмотренная альтернатива — клиентский `idempotencyKey` — отклонена для v1: она
перекладывает дисциплину на фронт и нужна только для сценариев параллельных
легитимных запусков, которых в ручном потоке пока нет. Оставлена как возможный
follow-up.

## Architectural Classification

Затрагивает только `Flight Operations`.

Затрагиваемые агрегаты:

* `FlightOccurrence` — новый инвариант на создание следующего manual-прогона;
* `FlightDefinition` — только как источник карточки (без изменений).

`Announcements`, `Audio Catalog`, `Localization`, `aeroflow-playback`,
`aeroflow-agent` не затрагиваются. Запуск объявлений по новому occurrence идёт
через уже существующие action-эндпоинты (`011`/`012`) без изменений.

## Backend Scope

### Новый use case: StartNextManualFlightOccurrence

```http
POST /api/v1/flight-occurrences:start-next-manual
```

```json
{ "flightDefinitionId": "uuid", "operationalDate": "2026-06-27" }
```

Поведение (одна локальная транзакция, последний occurrence грузится на запись):

1. Найти последний manual-occurrence по `flightDefinitionId + operationalDate +
   source=manual`, упорядоченный по `sequenceNumber` убыванию.
2. Если его нет — это путь «первого запуска», обслуживаемый
   `EnsureManualFlightOccurrence`; `start-next-manual` возвращает `409` с
   причиной «нет предыдущего запуска». Сознательно **не** дублируем здесь создание
   seq=1, чтобы у каждого входа была одна ответственность.
3. Если последний occurrence **не достиг финального статуса своего цикла**
   (т.е. находится в `scheduled`, `check_in_open` или `check_in_closed`) →
   `409 Conflict`, ничего не создаётся. Финальные статусы: `boarding` (вылет),
   `arrival_announced` (прилёт), а также терминальные `completed`/`cancelled`.
4. Иначе создать occurrence с `sequenceNumber = max + 1`, статус `scheduled`,
   `source=manual`, direction наследуется от `FlightDefinition`.

Response: `201` с `FlightOccurrenceResult` (новый occurrence).

### Разграничение с EnsureManualFlightOccurrence

* `EnsureManualFlightOccurrence` (seq=1, идемпотентный) — **первый** прогон
  карточки за дату; остаётся как есть, панель использует его без изменений;
* `StartNextManualFlightOccurrence` — **последующие** прогоны, серверный
  инкремент, guard по достижению финального статуса цикла.

### Конкурентность

`FlightOccurrence` получает optimistic/pessimistic lock на чтение последнего
прогона при создании следующего, чтобы две параллельные команды не выделили
один и тот же `sequenceNumber`. Уникальный индекс по бизнес-ключу остаётся
последней защитой: проигравшая гонку команда получает конфликт, а не дубликат.

### Backend Requirements

* `start-next-manual` создаёт occurrence только когда предыдущий достиг
  финального статуса цикла (`boarding`/`arrival_announced`/`completed`/`cancelled`);
* при предыдущем, не достигшем финального статуса, возвращает `409` и ничего не
  персистит;
* при отсутствии предыдущего возвращает `409` (не создаёт seq=1);
* следующий `sequenceNumber` назначает сервер, клиентское значение не принимается;
* параллельные вызовы не создают дубликат бизнес-ключа;
* покрыть application/API-тестами: happy path (seq 2 после `boarding`/`arrival_announced`),
  отказ при незавершённом цикле, отказ при отсутствии предыдущего, защита от
  дубля при двойном вызове.

## Frontend Scope

Расширить feature `dispatcher`.

* На карточке, чей сегодняшний occurrence достиг финального статуса цикла
  (`boarding`/`arrival_announced`, либо `completed`/`cancelled`), показать действие
  **«Новый запуск»** (через выделенный фильтр действия, см. ниже);
* «Новый запуск» вызывает `POST /flight-occurrences:start-next-manual`
  (`flightDefinitionId`, `operationalDate`), получает новый occurrence в статусе
  `scheduled`;
* после успеха инвалидировать query-ключи доски и dispatcher-occurrences за дату,
  чтобы карточка снова появилась под фильтрами первичных действий («Начало
  регистрации» / «Прибытие»);
* двойной клик: primary-кнопка дизейблится на время мутации (как в `012`); даже
  если запрос уйдёт дважды, второй вернёт `409` (guard), который показывается как
  conflict без потери UX;
* `409` «предыдущий запуск ещё не завершён» / «нет предыдущего» показывается
  рядом с действием человекочитаемо, без internal exception names.

Доска по-прежнему строится слиянием `flight-definitions` + dispatcher
`flight-occurrences`. Наложение статуса теперь учитывает, что у карточки за дату
может быть несколько occurrence — для отображения статуса берётся **последний по
`sequenceNumber`**.

## Out of Scope

* `FlightSchedule` и автоматическое создание occurrence по расписанию;
* идентичность occurrence по времени вылета (см. Open decision ниже);
* клиентский/серверный `idempotencyKey` на запуск;
* отмена occurrence из панели (`cancelled` как явное действие диспетчера);
* история/список всех прогонов карточки за день в UI (показываем только статус
  последнего);
* ролевое разграничение доступа.

## Acceptance Criteria

Backend:

* `POST /api/v1/flight-occurrences:start-next-manual` создаёт occurrence с
  `sequenceNumber = max + 1`, когда последний прогон карточки за дату достиг
  финального статуса цикла;
* при последнем прогоне, не достигшем финального статуса, возвращает `409`,
  occurrence не создаётся;
* при отсутствии предыдущего прогона возвращает `409`, occurrence не создаётся;
* `sequenceNumber` назначается сервером; клиентское значение игнорируется/запрещено;
* параллельные вызовы не приводят к дублю бизнес-ключа;
* добавлены application/API-тесты на все ветки.

Frontend:

* на карточке с финальным статусом цикла доступно «Новый запуск»;
* «Новый запуск» создаёт следующий прогон, карточка возвращается под первичные
  действия, доска обновляется;
* статус карточки берётся из последнего occurrence за дату;
* двойной клик не создаёт второй прогон (кнопка дизейблится, `409` обрабатывается);
* конфликтные ответы показываются рядом с действием без internal names.

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
1. Войти, открыть /dispatcher, провести рейс через полный цикл до финального
   статуса (открыть регистрацию → закрыть → посадка для departure; прибытие для
   arrival).
2. В фильтре «Новый запуск» нажать на этой карточке: создаётся новый прогон,
   карточка снова доступна под «Начало регистрации» / «Прибытие».
3. Сразу второй раз нажать «Новый запуск» (предыдущий прогон уже scheduled):
   получить 409, дубликат не создан.
4. Провести второй прогон тоже до финального статуса и убедиться, что можно
   открыть третий.
5. Попробовать «Новый запуск» на карточке без единого прогона за день: 409.
```

## Documentation Updates

До реализации (Draft → Ready) — выполнено:

* `domain-model.md`, раздел `FlightOccurrence`: добавлен инвариант «следующий
  manual-occurrence за дату создаётся только когда последний прогон достиг
  финального статуса цикла» и описан `StartNextManualFlightOccurrence` (серверный
  инкремент `sequenceNumber`) рядом с `Create`/`EnsureManualFlightOccurrence`;
* `domain-model.md`, «Открытые решения»: зафиксировано, что для ручного потока
  допускается несколько occurrence в день; `sequenceNumber` остаётся
  синтетическим ключом, а при появлении `FlightSchedule` идентичность occurrence
  пересматривается в пользу времени вылета (operationalDateTime).

После реализации:

* `aeroflow-web/architecture.md`: дополнить описание dispatcher-среза
  возможностью повторного прогона карточки в тот же день («Новый запуск» по
  достижении финального статуса, статус карточки берётся из последнего прогона,
  read model отдаёт последний occurrence per definition).

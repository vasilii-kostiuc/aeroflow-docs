# Повтор объявления «продолжение регистрации»

Status: Ready
Bounded context: Playback (repeat-цикл задания и оркестрация), Announcements
(repeatRule в запросе и стоп-повтора), Flight Operations (граница open/close как
триггеры), dispatcher read-model
Service: aeroflow-core, aeroflow-playback, aeroflow-web
Change type: Application, Domain, Integration, Infrastructure

## Goal

Пока регистрация рейса открыта, объявление «продолжение регистрации» должно
повторяться автоматически через настроенный интервал и прекращаться, когда
диспетчер закрывает регистрацию. Это первый срез механизма повторов эпика
очереди — после read-only экрана (017), удаления ожидающего (018) и остановки
звучащего (019).

```text
CheckInOpened
  -> core: Announcement(check_in_continuation) + RequestAnnouncementPlayback{ repeatRule:{everyMinutes:N} }
  -> playback: Pending -> Playing -> (complete) Scheduled(+N) -> [delayed wake] Pending -> Playing -> ...
CheckInClosed
  -> core: StopAnnouncementRepeat (нейтральная команда по announcementId)
  -> playback: repeatCancelled; текущий тик доигрывает -> Completed (терминал), цикл не взводится
```

Повтором **владеет playback**: интервал — это ответ на вопрос «когда
воспроизвести», а он принадлежит playback. Core за всю серию шлёт ровно два
сообщения: запрос с `repeatRule` на открытии и стоп-повтора на закрытии. Playback
не знает про регистрацию и рейсы — он лишь циклит задание, пока ему не сказали
прекратить.

## Ключевое расхождение с текущей domain-model.md

`domain-model.md` сейчас описывает continuation как **per-tick Announcement**:
каждый повтор — отдельное объявление, создаваемое core через «process manager
авто-повторов», с доменным событием `CheckInContinued` на каждый тик; под это уже
написан `FlightOccurrence::continueCheckIn()`.

Эта задача сознательно меняет модель на **один Announcement, повторяемый
playback'ом**. Обоснование: контент continuation от тика к тику не меняется
(динамических слотов, требующих перерезолва, у него нет), поэтому плодить
`Announcement` и `CheckInContinued` на каждый гудок незачем; тайминг повтора по
правилам разделения ответственности принадлежит playback, а не core. Доменным
фактом становится «серия повторов запущена/остановлена», а не «сыграл N-й раз».

Следствия для core (фиксируются в доках при реализации):

* `FlightOccurrence::continueCheckIn()` и событие `CheckInContinued` как триггер
  повтора **устаревают**; действие «продолжение регистрации» перестаёт быть
  отдельным диспетчерским переходом. Метод/событие удаляются или помечаются
  устаревшими вместе с обновлением `domain-model.md`.
* Интервал берётся из уже существующего `FlightAnnouncementConfig.repeatEveryMinutes`
  (валиден только для `CheckInContinuation`).
* `Announcement` типа `check_in_continuation` получает небольшое расширение
  lifecycle: активная серия повтора завершается фактом `AnnouncementRepeatEnded`
  (не `Cancelled`). Это парная к `AnnouncementCreated` точка post-commit-издания
  (см. Decision 11).

## Decisions

1. **Повтором владеет playback, границей — core.** Playback не знает про рейсы
   (правило #2), поэтому условие остановки «регистрация закрыта» не может жить в
   playback. `repeatRule` открытый (без счётчика/конца); серию завершает core
   нейтральной integration-командой при переходе occurrence в `check_in_closed`.

2. **Серия continuation заводится в действии открытия регистрации, тем же
   оркестратором.** `LaunchOccurrenceAnnouncementHandler` в ветке
   `check_in_opening` в **той же локальной транзакции** запускает через порт
   `AnnouncementLauncherInterface` **два** объявления: `check_in_opening`
   (одноразовое) и `check_in_continuation` (повторяемое, с `repeatRule` из
   конфига). Отдельного диспетчерского действия под continuation нет —
   готовность обоих шаблонов остаётся предусловием перехода occurrence в
   `check_in_open`. Опубликованное `check_in_opening` играет один раз, серия
   continuation стартует с задержкой в один интервал (Decision 4).

3. **Одна циклящаяся джоба, а не child-job на тик.** Повторяемый `PlaybackJob`
   ходит по кругу `Playing -> Scheduled -> Pending -> Playing`; при каждом
   завершении прогона ему двигают `scheduledAt = completedAt + everyMinutes`.
   Никаких новых сущностей; экран очереди остаётся eventual.

4. **Первый тик — не мгновенный, а через один интервал; джоба рождается
   `Scheduled`.** При `repeatRule` playback создаёт `PlaybackJob` сразу в статусе
   `Scheduled` с `scheduledAt = now + everyMinutes` (задействуется уже
   существующий nullable-параметр `scheduledAt` конструктора). «Продолжение»
   напоминает через интервал после открытия, а не одновременно с приветственным
   `check_in_opening`. Каденция **completion-relative**: следующий `scheduledAt`
   отсчитывается от фактического завершения тика, поэтому длительность
   проигрывания и ожидание в очереди сдвигают ритм — «догона» пропущенных тиков
   нет (см. Out of scope).

5. **Конец прогона повторяемой джобы — не терминальный `Completed`, а re-arm в
   `Scheduled`.** `complete()` разветвляется:

   ```text
   repeatRule есть И серию не остановили -> Scheduled (scheduledAt = completedAt + everyMinutes)
   иначе                                 -> Completed (терминал, как сейчас)
   ```

   Это единственное изменение самого агрегата помимо флага остановки. `fail()` и
   `interrupt()` остаются терминальными и для повторяемой джобы (сбой/остановка
   гасят серию — повтор не «лечит» упавший тик).

6. **Остановка серии — идемпотентный флаг `repeatCancelled` по образцу
   `stopRequested` (019).** `StopAnnouncementRepeat` находит джобу по
   `announcementId` под write-lock и ставит флаг. Флаг **не глушит текущий тик** —
   он доигрывает; лишь следующий `complete()` идёт в терминал вместо re-arm. Если
   джоба уже `Scheduled`/`Pending` (между тиками), флаг переводит её в терминал
   без ожидания: `Scheduled|Pending -> Completed`. Повторная доставка — no-op.

7. **Пробуждение очереди в `scheduledAt` — delayed Messenger-сообщение.** Между
   тиками ничего не играет и `Scheduled`-джоба с будущим `scheduledAt` не является
   кандидатом отбора. При рождении джобы и на каждый re-arm playback диспатчит
   delayed-команду («переоценить джобу X») с `DelayStamp(everyMinutes)`. При её
   приёме `Scheduled -> Pending`, дальше — обычный приоритетный отбор; если джоба
   уже терминальна (пришёл стоп), delayed-команда — no-op. Cron не вводится;
   тик-движок целиком внутри playback.

8. **Отбор и приоритеты не меняются.** Подошедший тик заходит `Pending` под своим
   приоритетом (100 для рейсового) и конкурирует по priority + FIFO(`createdAt`).
   Повтор **не прерывает** активное задание и не прыгает через очередь. Пока
   джоба `Scheduled`-и-не-время, аудиовыход свободен для других `Pending`;
   single-active-инвариант цел.

9. **Контракт `RequestAnnouncementPlayback` не расширяется структурно** — `repeatRule`
   в нём уже есть (сейчас `null`). Для continuation core кладёт
   `{ everyMinutes: N }` из `FlightAnnouncementConfig`. `schemaVersion`
   инкрементируется, поле остаётся опциональным; playback без `repeatRule`
   работает как раньше (одноразовая джоба).

10. **`StopAnnouncementRepeat` — новая нейтральная операторская команда, не
    перегрузка `CancelAnnouncementPlayback`.** Cancel (018) означает «отменить
    ожидающее объявление» и уводит `Announcement -> Cancelled`; здесь же серия
    **штатно завершается** на закрытии регистрации, а `Announcement` не ошибочен и
    в `Cancelled` не уходит. Команда идёт той же очередью `announcement_playback`
    четвёртым дискриминатором `command: announcement_playback.stop_repeat`
    (ADR 002, тот же `match` в сериализаторе).

11. **Оркестрация закрытия и издатель стоп-повтора (резолюция развилки).**
    Симметрично старту: закрытие регистрации оркеструет тот же
    `LaunchOccurrenceAnnouncementHandler`, а издаёт команду `Announcements`.

    * Порт `AnnouncementLauncherInterface` **переименовывается в
      `OccurrenceAnnouncementPort`**: добавление `endContinuationRepeat` делает
      имя «Launcher» узким, а оба метода — операции одного рода (синхронные
      мутации Announcements, co-committing с переходом occurrence в той же
      Doctrine-транзакции). Интерфейс остаётся единым (не разводится на два):
      единственный потребитель — оркестратор `LaunchOccurrenceAnnouncementHandler`,
      который инжектит обе операции; ISP-разведение дало бы ceremony без изоляции.
      Адаптер `BusAnnouncementLauncher -> BusOccurrenceAnnouncements`,
      DTO `LaunchedAnnouncement` сохраняется. Миграция механическая (DI по
      интерфейсу, меняются typehints и имена файлов).
    * Новый метод `endContinuationRepeat(flightOccurrenceId): void`. В ветке
      `check_in_closing`, в той же транзакции после `closeCheckIn()`,
      оркестратор вызывает его. Flight Operations **не знает** playback-контракт —
      зовёт свой порт, как и на старте.
    * Реализация порта живёт в `FlightOperations\Infrastructure\Integration` и
      адаптирует `Announcements`. Она находит активную серию continuation **по
      `flightOccurrenceId`** (`Announcement` уже хранит `flightOccurrenceId`) —
      Flight Operations не носит `announcementId` серии, occurrence не меняется.
    * **«Серии нет» — штатный no-op, а не ошибка.** Если у рейса не настроен
      `repeatEveryMinutes`, открытие не создаёт continuation-объявление, поэтому
      закрытие не находит активной серии и просто ничего не делает. Это
      обязательное поведение: без него закрытие регистрации ломалось бы для рейсов
      без настроенного повтора. (Отличается от идемпотентной повторной доставки —
      там серия была, но уже завершена.)
    * `Announcement` continuation переходит `endRepeat()` → доменный факт
      `AnnouncementRepeatEnded` (идемпотентно — событие только при первом
      переходе, как `AnnouncementCancelled` в 018).
    * `Announcements` post-commit реагирует на `AnnouncementRepeatEnded` и
      публикует `StopAnnouncementRepeat{announcementId}` — тем же post-commit
      механизмом, что издаёт `RequestAnnouncementPlayback` на
      `AnnouncementCreated`. Владельцем контракта остаётся `Announcements`.

## Aggregates and events

* Затронутый aggregate `PlaybackJob`: рождение в `Scheduled` при `repeatRule`;
  новый переход `Playing -> Scheduled` (re-arm) и `Scheduled|Pending -> Completed`
  при остановке; поле `repeatCancelled`; активируются `Scheduled`-статус и
  `scheduledAt`.
* Затронутый aggregate `Announcement` (тип `check_in_continuation`): расширение
  lifecycle серией повтора — `endRepeat()` и доменный факт
  `AnnouncementRepeatEnded` (не `Cancelled`, идемпотентно). Прочие типы метод не
  вызывают.
* `FlightOccurrence` **не меняется**: `continueCheckIn()`/`CheckInContinued`
  устаревают как триггер (см. «Ключевое расхождение»), новых полей нет — серия
  находится по `flightOccurrenceId` на стороне `Announcements`.
* Порт `FlightOperations\Application\Port\Announcements\AnnouncementLauncherInterface`
  переименовывается в `OccurrenceAnnouncementPort` и получает метод
  `endContinuationRepeat(flightOccurrenceId): void` (no-op, если активной серии
  нет); адаптер `BusAnnouncementLauncher -> BusOccurrenceAnnouncements` в
  `FlightOperations\Infrastructure\Integration`, DTO `LaunchedAnnouncement` без
  изменений.
* Новые integration messages: `StopAnnouncementRepeat`
  (`announcement_playback.stop_repeat`, Core → Playback); внутренняя
  delayed-команда playback для пробуждения. Агент не участвует — контракт
  Playback → Agent не меняется, каждый тик это обычный `audio.play_sequence`.

## Out of scope

* pause/resume, volume, emergency mode и приоритетное прерывание;
* повторы прочих типов объявлений (дополнительные/экстренные) и произвольные
  `repeatRule` (счётчик N раз, cron-выражения) — только `{everyMinutes}`;
* «догон» пропущенных тиков, если очередь была занята дольше интервала (тик
  просто играет с задержкой, без компенсации количества);
* recovery повторяемой джобы после гибели playback/агента между тиками;
* frontend-действия: серия управляется автоматически, отдельной кнопки нет; экран
  очереди показывает повторяемую джобу существующими generic-средствами.

## Acceptance Criteria

* Открытие регистрации создаёт в одной транзакции оба объявления
  (`check_in_opening` + `check_in_continuation`); приветственное играет один раз,
  первый тик continuation звучит через один интервал, затем повторяется каждые N
  минут тем же job (двигается `scheduledAt`). Между тиками статус `Scheduled`,
  аудиовыход свободен для других заданий.
* Повтор не прерывает звучащее задание и заходит в очередь по приоритету/FIFO.
* Закрытие регистрации: `endContinuationRepeat` находит серию по
  `flightOccurrenceId`, `Announcement` даёт `AnnouncementRepeatEnded` (при первом
  переходе), core шлёт ровно одну `StopAnnouncementRepeat`; текущий тик
  доигрывает, следующий не планируется; джоба уходит в `Completed`. `Announcement`
  и `FlightOccurrence` не уходят в `Cancelled`.
* `endRepeat()`/`AnnouncementRepeatEnded` и `StopAnnouncementRepeat` идемпотентны:
  повтор и опоздавшая команда (джоба уже терминальна) — no-op без второго эффекта.
* `RequestAnnouncementPlayback` без `repeatRule` создаёт одноразовую джобу как
  раньше (регрессий нет).
* Рейс без настроенного `repeatEveryMinutes`: открытие не создаёт continuation,
  закрытие штатно проходит (`endContinuationRepeat` — no-op, серии нет), стоп в
  playback не шлётся.
* `fail`/`interrupt` терминальны и для повторяемой джобы — серия не
  самовосстанавливается.
* Есть unit-тесты `PlaybackJob` (рождение в `Scheduled`, re-arm, флаг остановки,
  терминализация) и `Announcement` (`endRepeat` идемпотентность),
  application-тесты стоп-повтора и delayed-пробуждения, serializer-тест
  четвёртого `command`, тесты Announcements на публикацию `repeatRule` и
  стоп-повтора.

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-playback
php bin/phpunit

# aeroflow-web
npm run lint && npm run typecheck && npm run test:run
```

Manual QA: открыть регистрацию рейса с настроенным `repeatEveryMinutes`;
убедиться, что continuation звучит повторно через интервал и виден в очереди;
закрыть регистрацию — текущий тик доигрывает, новые не появляются, строка уходит
в «Недавние» как завершённая. Сквозной прогон через реальные контейнеры (core,
playback, RabbitMQ, обе Postgres) с укороченным интервалом.

## Documentation Updates

При реализации:

* `domain-model.md` — раздел `PlaybackJob`: пятый срез (рождение в `Scheduled`,
  `Playing -> Scheduled` re-arm, `repeatCancelled`, delayed-пробуждение);
  `Announcements` — `repeatRule` для continuation, серия повтора
  `endRepeat()`/`AnnouncementRepeatEnded` и издание `StopAnnouncementRepeat`,
  второй метод `endContinuationRepeat` на Flight-Ops-порту; `FlightOccurrence` —
  пометить `continueCheckIn()`/`CheckInContinued` устаревшими; открытые решения —
  зафиксировать переход от per-tick Announcement к playback-owned repeat.
* `architecture.md` — Core → Playback: четвёртая команда на `announcement_playback`;
  Playback → Core: без изменений; отметить, что повторы теперь ответственность
  playback (активированы `Scheduled`/`scheduledAt`).
* `README.md` — «Продолжение/Окончание регистрации» описать через playback-повтор;
  `Scheduled` в статусах очереди; срез 020 рядом с 018/019.
* `adr/002-inbound-message-discrimination.md` — подтверждение четвёртого типа на
  одной очереди тем же `match`.

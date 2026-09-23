# Приоритетное прерывание (преемпция) в playback

Status: Done
Bounded context: Playback (только внутренняя механика — новый статус задания,
прерывание уже звучащего, requeue)
Service: aeroflow-playback
Change type: Domain, Application

## Goal

Дать `PlaybackJob` способность прерывать уже звучащее задание более приоритетным
вместо того, чтобы просто вставать в очередь позади него — предпосылка emergency
mode (задача 026). Сейчас в системе нет типа объявления с приоритетом выше уже
играющих (`100`/`50`/`0` никогда не прерывают друг друга, `architecture.md`,
раздел «Приоритеты»), поэтому у этой задачи нет наблюдаемого пользователем
эффекта до 026 — playback не знает слова «emergency», только зарезервированный
порог приоритета. Проверяется синтетическим `RequestAnnouncementPlayback` с
приоритетом на пороге, отправленным напрямую в интеграционных тестах.

```text
playback: активен Playing job (priority P1)
  -> приходит новый Pending job (priority P2 >= порог преемпции, P2 > P1)
  -> dispatcher просит агента остановить активный (тот же audio.stop, что в 019)
  -> agent: audio.playback_interrupted (без изменений контракта)
  -> playback: Playing -> Preempted -> Pending (requeue, не Interrupted)
  -> dispatcher выбирает следующего кандидата — теперь это P2
```

## Decisions

1. **Новый статус `PlaybackJobStatus::Preempted`.** Промежуточный, не терминальный:
   задание немедленно переводится обратно в `Pending` (requeue) в рамках того же
   application-хендлера, без задержки и без нового delayed-таймера (в отличие от
   повтора задачи 020, где ожидание интервала осмысленно). Приоритет и
   `createdAt` при requeue не меняются — задание не теряет своё место в
   FIFO-порядке своего приоритетного уровня и не наказывается за то, что было
   прервано.
2. **Флаг `preemptRequested` на `PlaybackJob`**, по образцу существующего
   `stopRequested` (задача 019, `src/Playback/Domain/Entity/PlaybackJob.php`).
   Новый метод `requestPreempt(): bool` — guard: только `Playing`-задание без уже
   выставленного флага. Взводится диспетчером очереди в момент решения
   прервать, а не в момент подтверждения агентом.
3. **`interrupt()` различает причину остановки.** Общий факт от агента
   (`audio.playback_interrupted`, обрабатывается как и раньше в
   `HandleInterruptPlaybackJob`/`HandleAgentPlaybackEvent`) один и тот же для
   операторского Stop (019) и преемпции — агенту не сообщается, зачем его
   попросили остановиться, и контракт `audio.stop`/`audio.playback_interrupted`
   не меняется вообще. Playback сам решает по своему флагу: если был выставлен
   `preemptRequested` — переход `Playing -> Preempted`; иначе — прежнее поведение
   `Playing -> Interrupted`. Единственная точка ветвления, никаких изменений в
   `aeroflow-agent`.
4. **Requeue — отдельный явный метод `PlaybackJob::requeue(): bool`**
   (`Preempted -> Pending`), вызываемый сразу вслед за `preempt()`-переходом в
   том же хендлере.
5. **`PlaybackQueueDispatcher::dispatchNext()` меняет early-return.** Сейчас:
   `if ($this->jobs->findActive() !== null) { return; }`
   (`src/Playback/Application/PlaybackQueueDispatcher.php`). Новая логика: если
   `active` есть, дополнительно смотрим `findNextPendingForUpdate()`; если он
   существует, его приоритет строго выше приоритета `active` **и** не ниже
   преемптивного порога (Decision 6) — вызываем `requestPreempt()` на `active`,
   публикуем `StopAudioPlayback::forJobId(...)` (тот же класс, что в 019) и
   выходим, **не** стартуя `next` в этом же проходе: он станет `Playing` только
   следующим вызовом `dispatchNext()`, вызванным после подтверждения
   `audio.playback_interrupted` — тем же паттерном, что уже использует
   `HandleInterruptPlaybackJob`.
6. **Порог преемпции — именованная константа, а не «любой более высокий
   приоритет».** Например `PlaybackJob::PREEMPTIVE_PRIORITY = 1000`. Без этого
   ограничения прерывание случайно распространилось бы на обычные приоритеты
   (`100`/`50`/`0`), что прямо противоречит уже задокументированному правилу
   («обычные объявления ставятся в очередь», `architecture.md`). Playback
   по-прежнему не обязан знать слово «emergency» — только число.
7. **Новое обратное событие `AnnouncementPlaybackPreempted`** (по образцу
   `AnnouncementPlaybackInterrupted`), публикуется при переходе
   `Playing -> Preempted`, до requeue. `AnnouncementPlaybackQueued` повторно не
   публикуется — задание не новое, просто возвращается в `Pending`.
8. **Идемпотентность.** Повторная/опоздавшая доставка `audio.playback_interrupted`
   для уже `Preempted`/requeued задания — no-op, как и сейчас для `Interrupted`
   (задача 019); второй `AnnouncementPlaybackPreempted` не публикуется.

## Aggregates and events

* `PlaybackJob` — новые переходы `Playing -> Preempted -> Pending`; новый флаг
  `preemptRequested`; новая именованная константа порога.
* Новый integration event `AnnouncementPlaybackPreempted` (Playback -> Core,
  очередь `playback_events`, тот же контракт-каркас, что у остальных receipt-
  событий: `event`, `messageId`, `correlationId`, `announcementId`, `jobId`,
  `occurredAt`, `schemaVersion`).
* `aeroflow-agent` — без изменений: ни контракта, ни кода.

## Out of Scope

* Реальный триггер преемпции — здесь приоритет на пороге никто не создаёт в
  проде, только синтетически в тестах; настоящий emergency-путь — задача 026.
* Глобальная приостановка обычной очереди, пока audio output свободен (что мешает
  стартовать НОВОМУ обычному заданию, а не только прервать уже играющее) — это
  отдельный, более широкий механизм задачи 026 (persistent emergency mode flag),
  не путать с точечным прерыванием одного уже играющего задания.
* Read-модель core / web (drawer «Статус») — не показывает `Preempted` отдельно
  от `Interrupted` до задачи 026.
* Pause/resume на агенте — не нужен, задание доигрывается заново с начала после
  requeue (обычный `PlayAudioSequence`, как для любого нового старта).

## Acceptance Criteria

* Pending-job с приоритетом ниже порога никогда не прерывает `Playing`-job —
  весь существующий пакет тестов (015/019/020/023) остаётся зелёным без
  изменения поведения.
* Pending-job на преемптивном приоритете, поставленный в очередь, пока играет
  job с более низким приоритетом, вызывает ровно один `StopAudioPlayback` для
  активного job.
* После `audio.playback_interrupted` активный job проходит
  `Playing -> Preempted -> Pending` (проверяется через репозиторий и через
  публикацию `AnnouncementPlaybackPreempted`), а не `Interrupted`.
* Тот же job, вернувшись в `Pending`, остаётся доступен для отбора и в итоге
  штатно завершается (`Completed`) при повторном запуске; приоритет и
  `createdAt` не изменились.
* Повторный/опоздавший `audio.playback_interrupted` для уже requeued job — no-op,
  второй `AnnouncementPlaybackPreempted` не публикуется.
* Обычный операторский Stop (019) на job без `preemptRequested` по-прежнему даёт
  `Interrupted`, не `Preempted` — оба пути различимы и не пересекаются.

## Verification

```bash
# aeroflow-playback
php bin/phpunit
```

Ручной сценарий не нужен — нет UI-триггера до задачи 026. Интеграционный тест
playback создаёт два синтетических `RequestAnnouncementPlayback` напрямую
(обычный приоритет + приоритет на пороге преемпции) и проверяет
последовательность переходов через репозиторий и опубликованные события.

## Documentation Updates

При реализации:

* `domain-model.md` — раздел `PlaybackJob`: новый статус `Preempted`, флаг
  `preemptRequested`, requeue-механика, новое событие
  `AnnouncementPlaybackPreempted`; «Открытые решения» — отметить playback-часть
  преемпции как реализованную, emergency mode как ещё не реализованный (026);
* `architecture.md` — Playback → Core: `AnnouncementPlaybackPreempted` в списке
  событий;
* ADR 002 — не меняется: уровень 2 (отдельный транспорт) всё ещё ждёт задачи
  026, здесь `RequestAnnouncementPlayback` не создаёт новых типов сообщений.

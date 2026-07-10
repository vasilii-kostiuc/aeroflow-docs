# Audio Agent: первый срез исполнения воспроизведения

Status: Ready
Bounded context: Audio Execution (aeroflow-agent), Playback (aeroflow-playback),
Audio Catalog (aeroflow-core — только download-контракт)
Service: aeroflow-agent (новый), aeroflow-playback, aeroflow-core
Change type: новый сервис, Application, Infrastructure, Integration

## Goal

Замкнуть цепочку до реального звука: `aeroflow-playback` при старте задания
отправляет агенту команду `PlayAudioSequence`, агент скачивает и проигрывает
аудиофайлы и сообщает результат, playback завершает задание и продвигает очередь
— вместо операторского стаба `playback:job:complete` из задачи 015.

```text
PlaybackJob: Pending -> Playing                       (dispatcher, 015)
  -> PlayAudioSequence                                (integration command -> agent)
  -> agent: download/cache файлов по audioAssetId     (download-контракт core)
  -> agent: последовательное воспроизведение          (ffplay; симуляция флагом)
  -> PlaybackStarted / PlaybackCompleted / PlaybackFailed   (agent -> playback)
  -> playback: CompletePlaybackJob | FailPlaybackJob  (та же команда, что в 015)
  -> очередь продвигается, core получает Completed
```

Решение #4 задачи 015 предусматривало это заранее: событие агента «ложится в
`CompletePlaybackJob` без изменения доменной модели» — данный срез реализует
именно этот шов.

## Decisions (приняты)

1. **Стек агента — C# / .NET.** Отдельный от PHP-сервисов стек осознанно:
   агент — on-prem процесс на машине у аудиовыхода (часто Windows, как у старой
   САО); .NET даёт один self-contained бинарник под Windows/Linux. Агент не
   содержит бизнес-логику, поэтому цена второго стека минимальна. Тесты — xUnit.
2. **Транспорт playback ↔ agent — RabbitMQ** (тот же брокер): очередь
   `agent_commands` (playback → agent) и `agent_events` (agent → playback).
   Сообщения переживают офлайн агента; фиксирует открытый вопрос architecture.md
   («RabbitMQ / WebSocket / HTTP») в пользу RabbitMQ.
3. **Контракты — нейтральный JSON с дискриминатором с первого дня** (ADR 002:
   для новых контрактов дискриминатор вводится сразу, т.к. `Stop/Pause/Resume/
   SetVolume` на той же очереди заведомо появятся): поле `command` в
   `agent_commands`, поле `event` в `agent_events`. PHP- и C#-классы друг про
   друга не знают.
4. **Файлы — download-контракт core + локальный кэш агента.** В `audioSequence`
   только `audioAssetId`; маппинг id → файл живёт в БД core. Core добавляет
   внутренний эндпоинт `GET /internal/v1/audio-assets/{id}/file` (стрим файла,
   правильный MIME, 404 для неизвестного/неактивного). Агент кэширует файлы на
   диске по `audioAssetId`; повторное воспроизведение не ходит в core. Это
   реализация «download contract», уже заявленного в Audio Catalog как будущее
   расширение.
5. **Авторизации между сервисами нет** (существующий открытый вопрос доков):
   эндпоинт доступен во внутренней сети без токена — известный временный
   компромисс, как post-commit publisher; отдельная задача при выходе за
   локальную установку.
6. **Воспроизведение — реальное через системный плеер** (`ffplay -nodisp
   -autoexit`, конфигурируемая команда) как дочерний процесс: завершение
   процесса = конец звука, ненулевой код = сбой. `AGENT_SIMULATE=1` — режим без
   звука (лог + задержка) для docker/CI.
7. **Агент без базы данных** (по architecture.md): дедупликация повторно
   доставленной `PlayAudioSequence` — по `messageId` в памяти процесса +
   маркер обработанного `jobId` в кэш-директории. Известное ограничение:
   гарантия «не проиграть дважды» не абсолютна; строгая идемпотентность живёт
   выше — `CompletePlaybackJob` в playback идемпотентен.
8. **Последовательность обработки** — агент потребляет `agent_commands` с
   prefetch 1 и играет строго последовательно; single-active инвариант
   обеспечивает playback (015), агент его не дублирует.

## Contracts

### playback → agent: очередь `agent_commands`

```json
{
  "command": "audio.play_sequence",
  "messageId": "uuid",
  "jobId": "uuid",
  "audioSequence": [
    {
      "languageCode": "ro-MD",
      "sortOrder": 0,
      "items": [
        { "type": "audio_asset", "audioAssetId": "uuid" },
        { "type": "pause", "durationMs": 500 }
      ]
    }
  ],
  "occurredAt": "2026-07-10T10:00:00+00:00",
  "schemaVersion": 1
}
```

* `audioSequence` передаётся из `PlaybackJob` как есть;
* агент играет группы в порядке `sortOrder`, внутри группы — items по порядку;
  `pause` — тишина `durationMs`;
* `Stop/Pause/Resume/SetVolume` — будущие `command` на этой же очереди, вне среза.

### agent → playback: очередь `agent_events`

```json
{
  "event": "audio.playback_started | audio.playback_completed | audio.playback_failed",
  "messageId": "uuid",
  "jobId": "uuid",
  "reason": "строка, только для failed",
  "occurredAt": "2026-07-10T10:00:05+00:00",
  "schemaVersion": 1
}
```

* `started` playback только логирует (Playing уже установлен dispatcher-ом);
* `completed` → внутренняя команда `CompletePlaybackJob(jobId)`;
* `failed` → `FailPlaybackJob(jobId)`; `reason` — в лог;
* обработка идемпотентна за счёт идемпотентности Complete/Fail (015).

### core → agent: download

```text
GET /internal/v1/audio-assets/{audioAssetId}/file
200: поток файла + Content-Type (audio/wav|mpeg|ogg)
404: asset неизвестен или неактивен
```

## Agent (aeroflow-agent, новый сервис)

* .NET Worker Service (`BackgroundService`), конфигурация через
  `appsettings.json` + env (`AGENT_AMQP_URI`, `AGENT_CORE_BASE_URL`,
  `AGENT_CACHE_DIR`, `AGENT_PLAYER_COMMAND`, `AGENT_SIMULATE`);
* консьюмер `agent_commands` (RabbitMQ.Client, prefetch 1, ack после отправки
  итогового события);
* `AudioCache`: файл по `audioAssetId` в кэш-директории; при промахе — скачивание
  с core (расширение из Content-Type); ошибка скачивания → `playback_failed`;
* `SequencePlayer`: разворачивает `audioSequence` в план «файл / пауза»,
  запускает плеер на файл, ждёт выхода процесса; интерфейс процесса абстрагирован
  (`IPlayerProcessRunner`) для тестов и симуляции;
* издатель `agent_events`;
* без бизнес-логики: агент не знает про рейсы, объявления и приоритеты — только
  `jobId` и последовательность.

## Playback (изменения)

* dispatcher после `Pending -> Playing` публикует `PlayAudioSequence` в
  `agent_commands` (снимает решение #5 задачи 015 «переход не порождает внешней
  команды»); публикация post-commit тем же буферным механизмом, что события;
* новый inbound-транспорт `agent_events` с decode-only сериализатором по `event`
  (зеркально ADR 002); хендлеры: `completed` → `CompletePlaybackJob`, `failed` →
  `FailPlaybackJob`, `started` → лог;
* консольный стаб `playback:job:complete` остаётся как аварийный инструмент
  оператора;
* `PlaybackJob` и доменная модель **не меняются**.

## Core (изменения)

* внутренний контроллер download-эндпоинта в `AudioCatalog` (стрим файла по id
  через существующий storage; без авторизации — решение #5);
* больше ничего: контракт `RequestAnnouncementPlayback` и приём `playback_events`
  не затрагиваются.

## Out of Scope

* `Stop/Pause/Resume/SetVolume` и прерывание звука (нужны emergency mode);
* несколько audio outputs / несколько агентов;
* авторизация download-эндпоинта и сообщений;
* предзагрузка/прогрев кэша, инвалидация при замене asset;
* durable state агента (переживание рестарта с точным анти-дублем);
* TTS.

## Acceptance Criteria

Agent:

* получив `PlayAudioSequence`, скачивает недостающие файлы, играет
  последовательность в порядке sortOrder/items и публикует `started`, затем
  `completed`;
* ошибка скачивания или ненулевой код плеера → `playback_failed` с `reason`;
* повторная доставка той же команды (тот же `messageId`) не проигрывает звук
  второй раз (в пределах ограничения решения #7);
* `AGENT_SIMULATE=1` проходит цепочку без аудиоустройства;
* агент не содержит доменных понятий аэропорта.

Playback:

* `Pending -> Playing` публикует `PlayAudioSequence` (post-commit; на rollback —
  нет команды);
* `audio.playback_completed` завершает job, публикует
  `AnnouncementPlaybackCompleted` в core и запускает следующий;
* `audio.playback_failed` фейлит job и запускает следующий;
* повторная доставка событий агента не ломает состояние (идемпотентность 015).

Core:

* download-эндпоинт отдаёт файл активного asset с корректным MIME и 404 для
  неизвестного.

Tests:

* agent (xUnit): план воспроизведения из audioSequence (порядок, паузы); маппинг
  результата процесса в события; дедуп по messageId; кэш (промах → скачивание,
  попадание → без HTTP) с фейковым HTTP-хендлером;
* playback: dispatcher публикует `PlayAudioSequence` при старте; сериализатор
  `agent_events`; хендлеры completed/failed → продвижение очереди;
* core: контроллер download (200 + MIME, 404).

## Verification

```bash
# aeroflow-agent
dotnet test

# aeroflow-playback
php bin/phpunit

# aeroflow-core
php bin/phpunit tests/Unit tests/Application tests/Functional
```

Manual QA:

```text
1. Поднять core + playback + rabbitmq; запустить агента на хосте (реальный звук)
   или в docker с AGENT_SIMULATE=1.
2. Диспетчерская панель: запустить объявление.
3. Убедиться: агент скачал файлы в кэш, звук проигран (или simulate-лог),
   PlaybackJob -> Completed без ручного playback:job:complete, очередь пошла
   дальше, в core появились receipts started/completed.
4. Запустить два объявления подряд: второе играет только после первого.
5. Убить агента до завершения звука -> job остаётся Playing; после рестарта
   агента доиграть вручную стабом (известное ограничение среза: recovery
   зависшего Playing — будущий срез).
6. Удалить файл asset на core -> failed, job Failed, очередь продвинулась.
```

## Documentation Updates

До реализации: этот документ. После реализации:

* `domain-model.md` — Audio Execution: первый срез реализован; транспорт агента
  зафиксирован (RabbitMQ); download-контракт Audio Catalog реализован;
* `architecture.md` — закрыть открытый вопрос транспорта Playback → Agent;
* `README.md` — цепочка доходит до реального аудиовыхода.

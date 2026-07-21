# Генерация AudioAsset через TTS (первый срез)

Status: Planned
Bounded context: Audio Catalog (Core — порт, адаптер, `AudioAsset.source`,
генерация), новый инфраструктурный сервис `aeroflow-tts` (stateless, вне доменных
контекстов)
Service: aeroflow-core, aeroflow-tts (новый)
Change type: Application, Domain, Infrastructure, Integration, New service

## Goal

Дать системе возможность создавать `AudioAsset` из текста синтезом речи, чтобы
`text`-сегмент конфигурации объявления можно было озвучить и довести конфиг до
готового состояния (сегмент `text` сейчас хранится, но без озвучивания делает
`FlightAnnouncementConfig` неготовым). Решение зафиксировано в `domain-model.md`,
раздел «Генерация assets через TTS».

Это **первый срез**: поднимается сам сервис `aeroflow-tts` и Core-сторона
(порт + адаптер + `source=generated` + кэш + ручной admin-эндпоинт генерации).
Автоматическое встраивание генерации в сохранение `text`-сегмента конфига —
следующий срез (см. Out of scope).

```text
admin: POST /api/v1/admin/audio-assets:generate { text, language, voice? }
  -> Core (Audio Catalog): hash(text+language+voice+modelVersion) уже был?
       да  -> вернуть существующий AudioAsset(source=generated)   (кэш-хит, TTS не зовём)
       нет -> TextToSpeechPort.synthesize(text, language, voice)
              -> HTTP POST aeroflow-tts /v1/synthesize -> audio/wav (+ modelVersion)
              -> сохранить WAV в тот же file storage, что upload
              -> AudioAsset(source=generated) + AudioAssetGenerated
```

`aeroflow-tts` **нейтрален к домену**: знает только `text`, `language`, `voice`.
Аэропортовые понятия (объявление, рейс, тип, приоритет, стойка, выход) внутрь не
протекают — сервис остаётся переиспользуемым и не является скрытым куском Audio
Catalog.

## Decisions

1. **Отдельный stateless-сервис `aeroflow-tts` (новый репозиторий).** Без БД, без
   доменных инвариантов, без Messenger. HTTP-контракт:

   ```text
   POST /v1/synthesize { text, language, voice? } -> 200 audio/wav
        заголовок ответа X-TTS-Model-Version: <строка версии голоса/модели>
   GET  /v1/voices   -> [ { voice, language, modelVersion } ]
   GET  /health      -> 200
   ```

   Пустой `text`, неизвестный `language`/`voice` — `400`/`422` с телом ошибки.
   Контракт версионируется (`schemaVersion` не нужен HTTP-синхрону, версия — в
   пути `/v1`).

2. **Стек сервиса — минимальный Symfony (HTTP-only).** Тот же язык и тулинг, что
   Core; никакого нового рантайма в «мозговой» части системы (решение развилки
   «.NET vs Symfony»: раз сервис живёт рядом с инфраструктурой Core, Symfony
   консистентнее, .NET-довод «как агент» был про физическую привязку агента к
   аудиовыходу, которой у TTS нет). Doctrine/Messenger не подключаются — сервис
   stateless.

3. **Движок синтеза — деталь за HTTP-контрактом; первый шаг — Piper.** Piper —
   нативный оффлайн-бинарник (ru/en голоса), вызывается процессом (Symfony
   Process) внутри сервиса. Замена движка (Silero/RHVoice/облако) не меняет
   контракт `/v1/synthesize`. Модели голосов кладутся в образ сервиса; пути —
   через ENV, не хардкодом.

4. **Simulate-режим для CI/сред без моделей (прецедент — агент, задача 016).**
   Флаг окружения переключает сервис в возврат детерминированного placeholder-WAV
   без вызова движка, чтобы тесты и pipeline не тянули бинарник и модели.

5. **Core обращается к сервису синхронно через порт `TextToSpeechPort`.** Порт —
   consumer-owned, в `AudioCatalog\Application\Port`; описывает потребность Core
   (`synthesize(text, LanguageCode, ?voice): SynthesizedAudio` с байтами +
   `modelVersion`), а не чужой API. Реализация `HttpTtsAdapter` — в
   `AudioCatalog\Infrastructure\Integration`. Синхронность оправдана: генерация —
   предусловие готовности конфигурации, нужен немедленный ответ; это не
   сервисная async-граница и не RabbitMQ.

6. **Кэш и идемпотентность генерации — в Core, не в сервисе.** Ключ
   `hash(text + language + voice + modelVersion)`. Перед вызовом сервиса Core ищет
   активный `AudioAsset(source=generated)` с этим ключом: есть — переиспользует
   (TTS не зовётся), нет — генерирует и сохраняет ключ на assets. Так сервис
   остаётся stateless. `modelVersion` берётся из ответа сервиса и входит в ключ —
   смена модели даёт новый asset, старый деактивируется, а не удаляется физически
   (инвариант «удаление используемого asset не должно молча ломать существующие
   объявления»).

7. **`AudioAsset` получает поле `source` (`uploaded` | `generated`) и фабрику
   `generate()`.** Сейчас у сущности только `register()`/`upload()` и нет
   `source`. Добавляются: `source`, фабрика
   `generate(name, language, storageKey, mimeType, sizeBytes, ttsMeta)` и
   доменное событие `AudioAssetGenerated` (парное к `AudioAssetUploaded`).
   `ttsMeta` (voice, modelVersion, textHash) хранится для кэша (Decision 6).
   Существующие uploaded-assets получают `source='uploaded'` миграцией (default).

8. **Выбор голоса под тип объявления — забота Core, не сервиса.** В первом срезе
   `voice` приходит опционально в admin-запросе; при отсутствии Core берёт
   дефолтный голос под `language`. Маппинг «тип объявления -> голос» — вопрос
   следующего среза (интеграция с конфигом), сервис о назначении текста не знает.

9. **Ручной admin-эндпоинт генерации — граница первого среза.**
   `POST /api/v1/admin/audio-assets:generate` создаёт (или возвращает из кэша)
   `AudioAsset(source=generated)` и отдаёт его так же, как upload-эндпоинт. Это
   делает срез end-to-end проверяемым, не трогая пока редактор конфигурации.

10. **Авторизация между Core и `aeroflow-tts` — нет (осознанный компромисс, как у
    download-контракта агента).** Сетевое доверие локальной on-prem установки;
    зафиксировано как известный временный компромисс, отдельная авторизация между
    сервисами остаётся открытым вопросом `architecture.md`.

## Aggregates and events

* Затронутый aggregate `AudioAsset` (Audio Catalog): новое поле `source`
  (`uploaded`|`generated`, default `uploaded` для существующих); фабрика
  `generate()`; поля кэша TTS (`voice`, `modelVersion`, `textHash`); новое
  доменное событие `AudioAssetGenerated`. `register()`/`upload()` не ломаются.
* Новый порт `AudioCatalog\Application\Port\TextToSpeechPort` (+ DTO
  `SynthesizedAudio`); реализация `AudioCatalog\Infrastructure\Integration\HttpTtsAdapter`.
* Новый use case генерации (`GenerateAudioAssetFromText`) с кэш-логикой по хешу.
* Новый сервис `aeroflow-tts` — без агрегатов и событий (stateless).
* Integration: синхронный HTTP Core → `aeroflow-tts` (`/v1/synthesize`). RabbitMQ
  не участвует.

## Out of scope

* **автогенерация при сохранении `text`-сегмента** `AnnouncementVariant`/
  `FlightAnnouncementConfig` и превращение неготового текстового сегмента в
  готовый `audio_asset` — следующий срез (этот срез даёт ручной admin-путь);
* маппинг «тип объявления/язык -> голос», UI выбора голоса;
* несколько движков одновременно, SSML/просодия, стриминг, пауза/громкость;
* кэш внутри сервиса, собственная БД сервиса, масштабирование/несколько инстансов;
* импорт legacy-каталога, категории и длительность `AudioAsset`;
* durable outbox / async-издание — взаимодействие синхронное по HTTP;
* авторизация между сервисами (остаётся открытым вопросом).

## Acceptance Criteria

* `aeroflow-tts` поднимается как отдельный сервис; `POST /v1/synthesize` с валидным
  текстом и языком отдаёт корректный WAV и заголовок `X-TTS-Model-Version`;
  `GET /v1/voices` и `GET /health` работают; пустой текст/неизвестный язык дают
  `4xx` с телом ошибки. Simulate-режим отдаёт placeholder-WAV без движка.
* `POST /api/v1/admin/audio-assets:generate` создаёт `AudioAsset(source=generated)`
  из текста, сохраняет WAV в тот же storage, что upload, и возвращает asset;
  повторный запрос с тем же `text+language+voice` при той же `modelVersion`
  возвращает **тот же** asset без повторного вызова TTS (кэш-хит).
* Смена `modelVersion` на стороне сервиса приводит к генерации нового asset;
  прежний деактивируется, существующие объявления на него не ломаются.
* `AudioAsset.source` присутствует; существующие uploaded-assets после миграции
  имеют `source='uploaded'`; `AudioAssetGenerated` издаётся при первом создании
  generated-asset.
* Недоступность `aeroflow-tts` даёт понятную ошибку admin-эндпоинта и **не**
  создаёт частичный/битый asset (генерация атомарна: нет WAV — нет asset).
* Тесты: unit `AudioAsset` (`generate()`, `source`, событие), application-тест
  use case с кэш-хитом/промахом (порт застаблен), тест `HttpTtsAdapter` на
  контракт (mock HTTP), тесты сервиса `aeroflow-tts` на `/v1/synthesize`
  (simulate) и валидацию входа.

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-tts
php bin/phpunit   # (или тестовый раннер выбранного шаблона сервиса)
```

Manual QA: поднять `aeroflow-tts` с реальным Piper и ru/en моделью; через
`audio-assets:generate` сгенерировать объявление на русском, скачать WAV,
прослушать; повторить тот же текст — убедиться в кэш-хите (asset тот же, TTS не
вызван). Сквозной прогон через реальные контейнеры (core + aeroflow-tts + Postgres
core).

## Documentation Updates

При реализации:

* `domain-model.md` — раздел «Генерация assets через TTS» уточнить фактом
  реализации (порт, `HttpTtsAdapter`, `AudioAsset.source`, `AudioAssetGenerated`,
  кэш по хешу, admin-эндпоинт); отметить, что автогенерация из конфига —
  следующий срез.
* `architecture.md` — добавить `aeroflow-tts` в компоненты/поток данных как
  инфраструктурный сервис в роли Audio Catalog (синхронный HTTP от Core, не через
  RabbitMQ); отметить компромисс отсутствия авторизации.
* `README.md` — добавить `aeroflow-tts` в список репозиториев и краткую роль.
* при выборе шаблона сервиса — README самого `aeroflow-tts` (эндпоинты, ENV,
  simulate-режим, требования к моделям Piper).

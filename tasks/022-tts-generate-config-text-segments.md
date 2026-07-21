# TTS-генерация текстовых сегментов конфигурации объявления

Status: Planned
Bounded context: Announcements (сегмент `text` получает сгенерированный asset,
оркестрация генерации при сохранении), Audio Catalog (генерация — уже готова,
задача 021)
Service: aeroflow-core (+ aeroflow-web — минимальное отображение), aeroflow-tts
(уже есть)
Change type: Application, Domain, Infrastructure, Integration

## Goal

Подключить TTS-генерацию (готовую в задаче 021) к **редактору конфигурации
объявления**: когда администратор сохраняет `text`-сегмент в `AnnouncementVariant`,
core автоматически синтезирует речь и превращает сегмент в готовый — тот начинает
резолвиться в `AudioSequence` наравне с `audio_asset`. Сейчас `text`-сегмент
делает конфиг неготовым (`FlightAnnouncementConfig::validationErrors()` возвращает
`text_segment_requires_tts`, а `AnnouncementTemplateResolver` бросает то же самое).

Задача 021 дала низ (сервис `aeroflow-tts`, порт `TextToSpeechPort`, use case
`GenerateAudioAsset`, `AudioAsset(source=generated)`, кэш в Core) и **ручной
admin-эндпоинт**. Здесь достраивается автоматическое подключение к конфигу —
второй срез, ранее вынесенный в Out of scope задачи 021.

```text
save text segment (Add/UpdateAnnouncementVariant)
  → Announcements: для каждого text-сегмента
      SpeechAssetGeneratorPort.generate(text, language) ──> audioAssetId
        (порт делегирует GenerateAudioAssetCommand в Audio Catalog; кэш 021 —
         одинаковый текст не пере-синтезируется)
  → text-сегмент сохраняется с resolved audioAssetId
  → resolver отдаёт этот asset в AudioSequence; text-сегмент больше не «неготов»
```

## Ключевое решение: где живёт результат генерации

`text`-сегмент **остаётся типом `text`** (редактируемый источник) и получает
**дополнительное поле `audioAssetId`** — ссылку на сгенерированный `AudioAsset`.
Это, а не конвертация в `audio_asset`-сегмент.

Обоснование: конвертация в `audio_asset` потеряла бы исходный текст, и его нельзя
было бы отредактировать/перегенерировать. Пара «text (источник) + audioAssetId
(результат)» сохраняет редактируемость: правка текста → новая генерация (кэш 021
по `hash(text)` даёт новый asset, прежний остаётся для других ссылок).

## Decisions

1. **Генерация — при сохранении сегмента, не при запуске объявления.** Держим
   решение задачи 021 (пре-шаг): `text` статичен (в отличие от динамических слотов
   стоек/выхода, которые резолвятся рантайм-параметрами), поэтому синтез идёт один
   раз при сохранении конфига, а не в горячем пути диспетчерского действия.
   Оркестрация — в `Announcements\Application` (`AddAnnouncementVariantHandler`,
   `UpdateAnnouncementVariantHandler`): перед/во время построения сегментов каждый
   `text`-сегмент резолвится в `audioAssetId`.

2. **Кросс-контекст — через consumer-owned порт.** Announcements не трогает
   агрегаты Audio Catalog (правило #11). Новый порт
   `Announcements\Application\Port\AudioCatalog\SpeechAssetGeneratorInterface`:
   `generate(string $text, string $languageCode): string /* audioAssetId */`.
   Реализация — в `Announcements\Infrastructure\Integration\AudioCatalog`,
   делегирует `GenerateAudioAssetCommand` (задача 021) через `ApplicationBus` и
   возвращает `AudioAssetResult->id`. Announcements application/domain не
   импортируют типы Audio Catalog.

3. **`AnnouncementTemplateSegment::text()` получает опциональный
   `audioAssetId`.** Домен хранит и `text` (источник), и `audioAssetId`
   (результат). Сегмент без резолва (ещё не сгенерирован или генерация отложена)
   остаётся невалидным; с резолвом — валиден. Миграция добавляет nullable-колонку
   (переиспользуем существующую `audio_asset_id`? — нет, у text другая семантика;
   отдельная колонка или та же — решить при реализации, но поведение: text-сегмент
   несёт свою ссылку на сгенерированный asset).

4. **Резолвер отдаёт сгенерированный asset вместо исключения.**
   `AnnouncementTemplateResolver`: ветка `Text` больше не бросает
   `text_segment_requires_tts` — она эмитит `{type: audio_asset, audioAssetId}` из
   resolved-ссылки, проверяя активность asset тем же `isActiveAsset()`, что и
   обычный `audio_asset`-сегмент. Отсутствующий/неактивный resolved-asset →
   `AudioAssetUnavailableException` (как для `audio_asset`).

5. **`validationErrors()` для text-сегмента меняет смысл.**
   `FlightAnnouncementConfig::validationErrors()` (и уровень `AnnouncementVariant`)
   считает text-сегмент неготовым только если у него **нет** resolved
   `audioAssetId` (например, генерация была отложена/упала), а не всегда. Готовый
   (сгенерированный) text-сегмент перестаёт быть причиной неготовности.

6. **Выбор голоса — дефолтный по языку.** Маппинг «тип объявления → голос»
   остаётся будущим (как и в 021). Порт зовёт генерацию без `voice` — сервис берёт
   дефолтный голос языка.

7. **Недоступность TTS блокирует сохранение сегмента, не запуск.** Синтез — часть
   сохранения варианта; падение `aeroflow-tts` возвращает `422`/`502` у
   save-эндпоинта, диспетчерское действие и запуск объявления не затрагиваются.
   Порядок: сгенерировать asset (свой коммит в Audio Catalog), затем сохранить
   вариант со ссылкой; осиротевший при последующем сбое asset безвреден (кэширован,
   переиспользуем).

8. **Идемпотентность — кэш задачи 021.** Повторное сохранение того же текста бьёт в
   кэш Core (`hash(text+language+voice+modelVersion)`), TTS не вызывается, тот же
   `audioAssetId`. Правка текста → кэш-промах → новый asset; прежний остаётся
   активным (может быть в других ссылках) — авто-деактивация по правке текста не
   вводится (как и в 021, чистка вне среза).

## Aggregates and events

* `AnnouncementTemplateSegment` (Announcements): `text`-сегмент получает
  resolved `audioAssetId`; фабрика `text()` принимает опциональный id; миграция
  добавляет колонку. Валидность text-сегмента зависит от наличия резолва.
* `FlightAnnouncementConfig` / `AnnouncementVariant`: `validationErrors()` для
  text-сегмента теперь по наличию resolved-asset, не безусловно.
* Новый порт `Announcements\Application\Port\AudioCatalog\SpeechAssetGeneratorInterface`
  + реализация в `Announcements\Infrastructure\Integration\AudioCatalog`
  (делегирует `GenerateAudioAssetCommand`).
* `AnnouncementTemplateResolver`: ветка `Text` эмитит asset вместо исключения.
* Новых доменных событий не требуется (генерация asset публикует
  `AudioAssetGenerated` внутри Audio Catalog — задача 021).

## Out of scope

* маппинг «тип объявления/язык → конкретный голос» и выбор голоса в UI (дефолт по
  языку);
* авто-деактивация/уборка осиротевших сгенерированных assets при правке текста;
* авто-перегенерация всех text-сегментов при апгрейде модели голоса (как в 021 —
  подхватывается при следующем сохранении);
* пакетная/фоновая генерация: синтез синхронный при сохранении (объёмы админские);
* богатый UX редактора (превью аудио, ручной выбор голоса) — сверх минимального
  отображения ошибки/готовности.

## Frontend (минимально)

Редактор `FlightAnnouncementConfig` (admin, aeroflow-web) уже позволяет добавлять
text-сегменты. Здесь достаточно: сохранение text-сегмента может вернуть
`422`/`502` (TTS недоступен или язык без голоса) — показать ошибку у формы, как
для прочих `422`. Явного «идёт генерация» индикатора не требуется (запрос
синхронный). Расширенный UX — вне среза.

## Acceptance Criteria

* Сохранение варианта с `text`-сегментом (`Add`/`UpdateAnnouncementVariant`)
  синтезирует речь через порт и сохраняет сегмент с resolved `audioAssetId`;
  повторное сохранение того же текста не вызывает TTS второй раз (кэш 021).
* `FlightAnnouncementConfig` с готовым (сгенерированным) text-сегментом больше не
  неготов из-за `text_segment_requires_tts`; `AnnouncementTemplateResolver`
  включает сгенерированный asset в `AudioSequence` в позиции сегмента.
* Неактивный/удалённый resolved-asset text-сегмента → `AudioAssetUnavailableException`
  (как для `audio_asset`).
* Недоступность `aeroflow-tts` или язык без голоса → сохранение варианта отдаёт
  `422`/`502`, конфиг не сохраняется с «висящим» неозвученным сегментом; запуск
  объявления не затронут.
* Announcements application/domain не импортируют агрегаты/типы Audio Catalog —
  только собственный порт.
* Тесты: unit `AnnouncementTemplateSegment` (text с/без resolved-asset,
  валидность), `AnnouncementTemplateResolver` (text-сегмент → asset; неактивный →
  ошибка), application-тест `Add/UpdateAnnouncementVariant` с застабленным портом
  (генерация вызвана, id сохранён, кэш-повтор), тест адаптера порта (делегирует
  `GenerateAudioAssetCommand`).

## Verification

```bash
# aeroflow-core
php bin/phpunit tests/Unit tests/Application

# aeroflow-web (если затронут)
npm run lint && npm run typecheck && npm run test:run
```

Manual QA: поднять core + aeroflow-tts; создать `FlightAnnouncementConfig` с
вариантом, содержащим `text`-сегмент; сохранить — убедиться, что сегмент получил
`audioAssetId`, конфиг стал готовым, а запуск объявления по этому конфигу отдаёт в
playback `AudioSequence` со сгенерированным asset в нужной позиции. Повторно
сохранить тот же текст — TTS не вызывается (кэш). Остановить `aeroflow-tts` и
сохранить новый текст — `502`, конфиг не портится.

## Documentation Updates

При реализации:

* `domain-model.md` — раздел `Announcements`: `text`-сегмент несёт resolved
  `audioAssetId`, генерируемый при сохранении через порт к Audio Catalog; убрать
  формулировку, что `text` безусловно делает конфиг неготовым (теперь — только без
  резолва); отметить, что автогенерация из конфига реализована (снять из Out of
  scope раздела «Генерация assets через TTS»).
* `architecture.md` — при необходимости уточнить, что подготовка `AudioSequence`
  включает уже сгенерированный из текста asset.
* `aeroflow-web/architecture.md` — если затронут редактор: text-сегмент
  озвучивается при сохранении, ошибка TTS показывается как `422`.

# Composable Announcement Templates

Status: Ready  
Bounded context: Announcements, Flight Operations, Audio Catalog  
Service: aeroflow-core  
Change type: Domain, Application, Infrastructure, API, Migration

## Goal

Развить настройку рейсовых объявлений в конструктор упорядоченных сегментов и
при ручном запуске формировать неизменяемую `AudioSequence`.

Поддержать выбор диспетчером отдельных стоек регистрации и одного выхода:

```text
стойки: 1, 3, 5
выход: A5
```

Решение следует
[`ADR 001`](../adr/001-composable-announcement-templates.md).

## Architectural Classification

Затрагиваемые агрегаты:

* `FlightAnnouncementConfig` и вложенные языковые варианты/сегменты;
* `Announcement`;
* новые справочники `CheckInCounter` и `Gate`;
* новый `AudioPrompt`;
* существующий `AudioAsset` только как цель ссылки.

Изменение доменное и инфраструктурное. Новые события публикуются для изменений
справочников и prompts. Разрешение шаблона не является самостоятельным доменным
событием: результат фиксируется внутри создаваемого `Announcement`.

`aeroflow-playback`, `aeroflow-agent`, TTS и frontend не входят в задачу.

## Domain Model

### Flight Operations directories

Добавить отдельные aggregate roots `CheckInCounter` и `Gate`.

Общее состояние:

* `id: UUID`;
* `code: string`;
* `displayName: string`;
* `sortOrder: positive int`;
* `active: bool`;
* `createdAt`;
* `updatedAt`.

Инварианты:

* нормализованный `code` непустой, не длиннее 16 символов и соответствует
  `^[A-Z0-9][A-Z0-9 -]{0,15}$`;
* код уникален внутри своего справочника без учёта регистра;
* `displayName` непустой и не длиннее 128 символов;
* `sortOrder >= 1`;
* использованная запись деактивируется, а не удаляется.

Поведение:

* создать;
* изменить код, название и порядок;
* активировать;
* деактивировать.

События:

* `CheckInCounterCreated`, `CheckInCounterUpdated`,
  `CheckInCounterActivated`, `CheckInCounterDeactivated`;
* `GateCreated`, `GateUpdated`, `GateActivated`, `GateDeactivated`.

### AudioPrompt

Добавить aggregate root `AudioPrompt`:

* `id: UUID`;
* `kind: check_in_counter_code|gate_code`;
* `value: string`;
* `languageCode`;
* `audioAssetId: UUID`;
* `active: bool`;
* `createdAt`;
* `updatedAt`.

Инварианты:

* `value` нормализуется по тем же правилам, что и код справочника;
* `languageCode` валиден;
* `audioAssetId` указывает на активный `AudioAsset`;
* одновременно существует не более одного активного prompt для
  `(kind, value, languageCode)`;
* prompt можно деактивировать без удаления.

События:

* `AudioPromptCreated`;
* `AudioPromptUpdated`;
* `AudioPromptActivated`;
* `AudioPromptDeactivated`.

### Announcement template segments

`AnnouncementVariant` сохраняет язык, порядок языков и активность, но вместо
единственного `sourceType/audioAssetId/text` владеет упорядоченной коллекцией
`AnnouncementTemplateSegment`.

Общие поля сегмента:

* `id: UUID`;
* `variantId`;
* `sortOrder: positive int`;
* `type`;
* type-specific payload;
* `createdAt`;
* `updatedAt`.

Типы и payload:

| type | payload |
|---|---|
| `audio_asset` | `audioAssetId: UUID` |
| `dynamic_slot` | `slot: check_in_counters\|gate_code` |
| `pause` | `durationMs: int` |
| `text` | `text: string` |

Инварианты:

* порядок сегментов стабилен и уникален внутри варианта;
* `audio_asset` ссылается на активный asset;
* `pause` имеет длительность `100..10000` мс;
* `text` после trim непустой;
* type-specific поля взаимоисключающие;
* dynamic slot совместим с типом объявления:
  * `check_in_counters` — только регистрационные объявления;
  * `gate_code` — только приглашение на посадку;
* активный вариант должен иметь минимум один сегмент;
* наличие `text` без TTS добавляет ошибку готовности
  `text_segment_requires_tts`.

Существующие события добавления/изменения варианта сохраняются для настроек
самого варианта. Для сегментов добавить:

* `AnnouncementTemplateSegmentAdded`;
* `AnnouncementTemplateSegmentUpdated`;
* `AnnouncementTemplateSegmentRemoved`.

### Announcement snapshot and AudioSequence

Удалить диапазон стоек и строковый gate из `Announcement`.

Добавить snapshot:

```text
checkInCounters: [{id, code}]
gate: {id, code}|null
audioSequence: [
  {languageCode, sortOrder, items: [...]}
]
```

Элемент последовательности:

```text
AudioAssetItem {audioAssetId}
PauseItem      {durationMs}
```

Snapshot и последовательность хранятся как JSON с доменными Value Objects,
которые валидируют форму при создании и восстановлении.

Изменение справочника, prompt, asset или шаблона после создания не меняет
сохранённые данные `Announcement`.

## Application Flow

### Create Announcement

API принимает:

```json
{
  "type": "check_in_opening",
  "flightDefinitionId": "uuid",
  "languages": ["ro-MD", "en"],
  "checkInCounterIds": ["uuid-1", "uuid-3", "uuid-5"],
  "gateId": null
}
```

Для `boarding_invitation` передаётся `gateId`, а `checkInCounterIds` отсутствует
или пуст.

Handler:

1. проверяет активную `FlightDefinition`;
2. загружает активную конфигурацию нужного типа;
3. проверяет, что запрошенные языки имеют активные варианты;
4. через порт Flight Operations получает выбранные активные справочники;
5. сохраняет их ID и коды как snapshot;
6. последовательно разрешает сегменты каждого языка;
7. через порт Audio Catalog проверяет assets и получает prompts;
8. при полном успехе создаёт `Announcement` с готовой `AudioSequence`;
9. сохраняет aggregate и публикует `AnnouncementCreated`.

Неизвестные, дублирующиеся или неактивные ID отклоняются. Для регистрации
требуется минимум одна стойка. Для посадки требуется ровно один gate.

Если prompt отсутствует, use case отклоняется одной диагностической ошибкой,
содержащей отсортированный список:

```text
kind, value, languageCode
```

Частичная последовательность не сохраняется.

### Consumer-owned ports

В `Announcements/Application/Port` добавить контракты:

```text
FlightOperations:
  resolveActiveCheckInCounters(ids) -> ordered snapshots
  resolveActiveGate(id) -> snapshot|null

AudioCatalog:
  resolveActiveAssets(ids) -> asset snapshots
  resolvePrompts(keys) -> prompt snapshots
```

Порты не возвращают агрегаты или типы поставщика. Реализации находятся только в
`Announcements/Infrastructure/Integration/{context}`.

## HTTP API

### Flight Operations admin API

Для обоих справочников:

```text
GET    /api/v1/admin/check-in-counters
POST   /api/v1/admin/check-in-counters
PATCH  /api/v1/admin/check-in-counters/{id}
POST   /api/v1/admin/check-in-counters/{id}/activate
POST   /api/v1/admin/check-in-counters/{id}/deactivate

GET    /api/v1/admin/gates
POST   /api/v1/admin/gates
PATCH  /api/v1/admin/gates/{id}
POST   /api/v1/admin/gates/{id}/activate
POST   /api/v1/admin/gates/{id}/deactivate
```

List endpoints поддерживают фильтр `active` и сортируют по
`sortOrder, code`.

### Audio Catalog admin API

```text
GET    /api/v1/admin/audio-prompts
POST   /api/v1/admin/audio-prompts
PATCH  /api/v1/admin/audio-prompts/{id}
POST   /api/v1/admin/audio-prompts/{id}/activate
POST   /api/v1/admin/audio-prompts/{id}/deactivate
```

List поддерживает фильтры `kind`, `value`, `languageCode`, `active`.

### Announcement config API

Variant endpoints продолжают управлять языком, порядком и активностью.
Поля `sourceType`, `audioAssetId`, `text` заменяются на `segments`.

```json
{
  "languageCode": "ro-MD",
  "sortOrder": 1,
  "enabled": true,
  "segments": [
    {"sortOrder": 1, "type": "audio_asset", "audioAssetId": "uuid"},
    {"sortOrder": 2, "type": "dynamic_slot", "slot": "check_in_counters"},
    {"sortOrder": 3, "type": "pause", "durationMs": 500}
  ]
}
```

### Announcement API breaking change

Удалить:

* `checkInCounterStart`;
* `checkInCounterEnd`;
* `gateCode`.

Добавить:

* `checkInCounterIds: list<UUID>`;
* `gateId: ?UUID`;
* snapshots и `audioSequence` в response.

## Persistence and Migration

Миграция:

1. создаёт таблицы `check_in_counter`, `gate`, `audio_prompt`,
   `announcement_template_segment`;
2. создаёт необходимые unique constraints и индексы;
3. для каждого существующего `announcement_variant` создаёт один сегмент:
   * `audio_asset` из `audio_asset_id`;
   * `text` из `text`;
4. добавляет JSON snapshot/sequence поля в `announcement`;
5. удаляет legacy-поля вариантов и объявления после переноса данных.

Миграция должна иметь явный `down()` для восстановления старых колонок и
односегментных вариантов. Откат допускается только если каждый вариант всё ещё
содержит ровно один совместимый сегмент; иначе `down()` должен завершаться с
понятной ошибкой, не теряя данные.

## Validation and Error Mapping

Ошибки валидации HTTP возвращают `400`, отсутствующие ресурсы — `404`,
конфликты уникальности — `409`, неготовая конфигурация или отсутствующие prompts
— `422`.

Сообщения об отсутствующих prompts не раскрывают storage paths и содержат только
семантические ключи.

## Tests and Acceptance Criteria

### Domain

* справочники нормализуют коды и защищают уникальность;
* `AudioPrompt` защищает семантический ключ;
* конструктор сохраняет порядок смешанных сегментов;
* несовместимые slots и некорректные payload отклоняются;
* `TextSegment` делает вариант неготовым без TTS;
* snapshot и `AudioSequence` неизменяемы.

### Application and integration

* можно выбрать независимые стойки `1`, `3`, `5`;
* неизвестные, неактивные и дублирующиеся ID отклоняются;
* правильные prompts разрешаются отдельно для каждого языка;
* отсутствие хотя бы одного prompt блокирует создание;
* изменение справочника или prompt не меняет созданное объявление;
* adapters являются единственным местом межконтекстных импортов.

### Persistence and API

* legacy variants мигрируют в один сегмент без потери данных;
* repositories восстанавливают все новые aggregates и Value Objects;
* CRUD API справочников и prompts соблюдает status codes;
* config API принимает и возвращает сегменты в стабильном порядке;
* announcement API использует только новые параметры;
* architecture dependency test остаётся зелёным.

### Quality gate

* весь PHPUnit suite проходит;
* PHPStan проходит без ошибок;
* Symfony container lint проходит;
* formatter dry-run не находит изменений;
* Doctrine schema validation проходит после миграции.

## Out of Scope

* frontend-конструктор;
* TTS и генерация аудио из текста;
* языковые fallback;
* автоматическая композиция составных кодов;
* слоты времени, номера рейса и аэропортов;
* RabbitMQ, PlaybackJob и отправка в `aeroflow-playback`;
* FlightSchedule и FlightOccurrence.

## Completion

После реализации:

* обновить `domain-model.md` и `architecture.md`;
* изменить статус задачи на `Done`;
* зафиксировать реализацию отдельными атомарными коммитами в core и docs;
* выполнить merge в `master` только после полного quality gate.


# Web Language Select and Segment Variants

Status: Done  
Bounded context: Localization, Announcements, Audio Catalog  
Service: aeroflow-core, aeroflow-web  
Change type: API, Frontend

## Goal

Убрать ручной ввод `languageCode` из административного интерфейса и привести
редактор вариантов рейсовых объявлений в `aeroflow-web` к текущей доменной
модели composable templates.

Администратор должен выбирать язык из управляемого списка, а не вводить код
вручную:

```text
ro-MD  Romana
ru     Русский
en     English
```

При этом write API продолжает принимать `languageCode` как строковый BCP 47 код.
Справочник языков нужен только как read model для UI и валидации пользовательского
ввода на стороне frontend.

## Architectural Classification

Задача затрагивает:

* `Localization` — новый supporting context/read model для доступных языков;
* `Announcements` — UI выбора языка для `AnnouncementVariant` и будущего запуска
  объявлений;
* `Audio Catalog` — UI выбора языка при загрузке `AudioAsset` и настройке
  `AudioPrompt`.

Изменение не является доменным изменением агрегатов. `LanguageCode` остаётся
value object и продолжает валидировать BCP 47-совместимый код.

Затрагиваемые агрегаты:

* `FlightAnnouncementConfig` — через языковые варианты;
* `AudioAsset` — через upload form;
* `AudioPrompt` — через prompt form, если он доступен в web;
* `Announcement` — будущий dispatcher UI должен использовать тот же список
  языков для `languages`.

Новые domain events не требуются.

`aeroflow-playback` и `aeroflow-agent` не затрагиваются.

## Current Problem

Сейчас в `aeroflow-web` язык местами задаётся через `TextInput` и дефолтное
значение `ro-MD`.

Это создаёт проблемы:

* администратор может ввести опечатку (`ro_MD`, `romd`, `ru-RU` вместо нужного
  кода);
* разные формы могут использовать разные наборы языков;
* UI не показывает человеку понятное название языка;
* редактор `AnnouncementVariant` в web всё ещё опирается на старую модель
  `sourceType/audioAssetId/text`, тогда как API уже ожидает `segments`.

## Backend Scope

Добавить в `aeroflow-core` read-only API для списка поддерживаемых языков.

На первом этапе без базы данных.

Источник данных:

* Symfony configuration parameter;
* или env/config файл, подключённый через Symfony container.

Пример конфигурации:

```yaml
aeroflow:
  languages:
    - code: ro-MD
      name: Romana
      nativeName: Romana
      active: true
      sortOrder: 1
    - code: ru
      name: Russian
      nativeName: Русский
      active: true
      sortOrder: 2
    - code: en
      name: English
      nativeName: English
      active: true
      sortOrder: 3
```

### API

Добавить endpoint:

```http
GET /api/v1/languages
```

или, если текущий routing prefix уже добавляет `/api`:

```http
GET /v1/languages
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "code": "ro-MD",
      "name": "Romana",
      "nativeName": "Romana",
      "active": true,
      "sortOrder": 1
    }
  ]
}
```

Только активные языки должны использоваться в select по умолчанию. Неактивные
можно возвращать, если UI должен показывать старые сохранённые значения, но
создание новых записей должно выбирать из активных.

### Backend Requirements

* каждый `code` валидируется через существующий `LanguageCode`;
* `code` уникален после нормализации;
* список сортируется по `sortOrder`, затем по `code`;
* endpoint не создаёт, не изменяет и не удаляет языки;
* существующие write endpoints не меняют payload:
  * `AnnouncementVariantRequest.languageCode`;
  * `CreateAnnouncementRequest.languages`;
  * `AudioPromptRequest.languageCode`;
  * `UploadAudioAssetCommand.languageCode`.

## Frontend Scope

Добавить в `aeroflow-web` feature для языков:

```text
src/features/languages/api/languageApi.ts
src/features/languages/hooks/useLanguages.ts
src/features/languages/model/types.ts
```

### Replace Manual Inputs

Заменить ручной ввод языка на `Select` или `MultiSelect`:

* upload `AudioAsset` — `Select`;
* create/update `AnnouncementVariant` — `Select`;
* future create `Announcement` dispatcher form — `MultiSelect`;
* `AudioPrompt` language field — `Select`, если prompt UI присутствует.

Default language:

* брать первый активный язык из справочника;
* если данные ещё загружаются, поле disabled/loading;
* если сохранённое значение не входит в активный список, показывать его как
  disabled option, чтобы существующая запись не выглядела сломанной.

### Segment-Based Variant Editor

Привести frontend model к текущему API:

```ts
export type AnnouncementTemplateSegmentType =
  | 'audio_asset'
  | 'dynamic_slot'
  | 'pause'
  | 'text'

export type AnnouncementTemplateSegmentInput = {
  sortOrder: number
  type: AnnouncementTemplateSegmentType
  audioAssetId?: string | null
  slot?: 'check_in_counters' | 'gate_code' | null
  durationMs?: number | null
  text?: string | null
}

export type AnnouncementVariantInput = {
  languageCode: string
  sortOrder: number
  segments: AnnouncementTemplateSegmentInput[]
  enabled: boolean
}
```

UI должен позволять:

* добавлять сегмент;
* менять порядок сегментов;
* выбирать тип сегмента;
* для `audio_asset` выбирать активный asset, желательно фильтруя по языку
  варианта;
* для `dynamic_slot` выбирать только допустимые слоты:
  * `check_in_counters` для регистрационных объявлений;
  * `gate_code` для `boarding_invitation`;
* для `pause` указывать `durationMs` в диапазоне `100..10000`;
* для `text` вводить непустой текст и показывать, что без TTS конфигурация
  будет не готова к dispatcher запуску.

## Out of Scope

* хранение языков в базе данных;
* admin UI для создания/редактирования языков;
* перевод интерфейса AeroFlow;
* TTS;
* импорт legacy language mapping;
* изменение формата `LanguageCode`;
* изменение API payload для существующих command endpoints;
* playback integration.

## Acceptance Criteria

Backend:

* `GET /v1/languages` возвращает список языков из конфигурации;
* некорректный язык в конфигурации приводит к явной ошибке приложения или
  теста;
* существующие endpoints продолжают принимать `languageCode` как строку;
* покрыты unit/application/API tests для списка языков.

Frontend:

* во всех формах настройки объявлений язык выбирается из select;
* ручной `TextInput` для `languageCode` удалён из editor flow;
* новый variant editor отправляет `segments`, а не `sourceType/audioAssetId/text`;
* audio asset select в сегменте показывает язык asset и фильтруется по языку
  варианта или явно помечает несовпадение;
* ошибки API 422/409 показываются рядом с формой;
* существующие tests обновлены под `segments` и language select.

## Verification

Backend:

```bash
composer test
```

или точечно:

```bash
php bin/phpunit
```

Frontend:

```bash
npm test
npm run lint
npm run build
```

Manual check:

1. открыть страницу карточек рейсов;
2. создать или открыть `FlightDefinition`;
3. открыть вкладку `Объявления`;
4. создать config;
5. добавить языковой вариант через select языка;
6. добавить сегменты `audio_asset`, `dynamic_slot`, `pause`;
7. сохранить и убедиться, что request payload содержит `segments`;
8. загрузить новый audio asset с языком из select.

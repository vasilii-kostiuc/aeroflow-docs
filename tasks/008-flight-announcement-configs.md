# Flight Announcement Configs

Status: Done  
Bounded context: Announcements, Flight Operations, Audio Catalog  
Service: aeroflow-core, aeroflow-web  
Change type: Domain, Application, Infrastructure, API, Frontend

## Goal

Добавить в админке настройку объявлений на странице `FlightDefinition`, чтобы
администратор мог подготовить, какие объявления доступны для рейса и из какого
содержимого они собираются.

Старая САО хранила эту информацию в структуре каталогов:

```text
вылеты/Анталия-2M3571/1_MDA/begin.wav
вылеты/Анталия-2M3571/2_ENG/begin.wav
вылеты/Анталия-2M3571/3_TUR/begin.wav
```

В AeroFlow эта логика должна стать управляемой моделью:

```text
FlightDefinition
    -> FlightAnnouncementConfig
        -> AnnouncementVariant
            -> AudioAsset или Text
```

Админка настраивает не реальные объявления, а заготовки для будущего запуска.
Реальный `Announcement` создаётся только в диспетчерском сценарии, когда
диспетчер выбирает рейс и нажимает действие.

## Architectural Classification

Изменение относится к `aeroflow-core` и `aeroflow-web`.

Затрагиваемые bounded contexts:

* `Flight Operations` — `FlightDefinition` является владельцем страницы
  настройки рейса;
* `Announcements` — конфигурация определяет, какие рейсовые объявления доступны;
* `Audio Catalog` — аудиофайлы могут быть источниками вариантов;
* `Audit` — будущая история изменений конфигурации.

Затрагиваемые агрегаты:

* `FlightDefinition` — только как родительская карточка рейса;
* `FlightAnnouncementConfig` — новая транзакционная граница для настройки типа
  объявления рейса;
* `AudioAsset` — связанный агрегат по идентификатору, без ORM-графа.

Доменное изменение:

* добавить модель конфигурации объявлений рейса;
* добавить модель языковых вариантов объявления;
* разрешить два типа источника содержимого: готовое аудио и текст.

Инфраструктурное изменение:

* добавить persistence, миграции, repository и HTTP API;
* добавить проверку существования `FlightDefinition` и `AudioAsset`;
* добавить frontend-вкладку на странице рейса.

`aeroflow-playback` и `aeroflow-agent` не затрагиваются. Playback не должен
получать `FlightDefinition`, `Gate`, `CheckInCounter` или настройки рейса.

## Scope

В `aeroflow-core`:

* создать модель `FlightAnnouncementConfig`;
* создать модель `AnnouncementVariant`;
* поддержать типы источника варианта:
  * `AudioAsset`;
  * `Text`;
* добавить persistence и миграции;
* добавить application commands и handlers;
* добавить HTTP API для чтения и изменения конфигураций;
* проверять инварианты модели;
* покрыть domain, application, persistence и API тестами.
* разрешить администратору загрузить WAV, MP3 или OGG как новый `AudioAsset`;
* хранить загруженные файлы вне публичного web-каталога.

В `aeroflow-web`:

* добавить вкладку `Объявления` на странице редактирования рейса;
* отобразить список типов рейсовых объявлений;
* дать возможность включать и выключать тип объявления;
* дать возможность добавлять, редактировать, отключать и удалять языковые
  варианты;
* дать возможность выбрать источник варианта: аудиофайл или текст;
* дать возможность загрузить новый аудиофайл непосредственно при настройке
  варианта;
* показать валидность конфигурации для диспетчерского запуска.

## Out of Scope

* dispatcher UI запуска объявлений;
* создание реального `Announcement` из конфигурации;
* подготовка итоговой `AudioSequence`;
* TTS-интеграция;
* генерация аудиофайлов из текста;
* импорт legacy-каталогов;
* дополнительные и экстренные объявления аэропорта;
* музыка и гонги;
* playback, RabbitMQ и `PlaybackJob`;
* `FlightSchedule` и `FlightOccurrence`;
* полноценный Audit UI.

## Domain Decisions

### Configuration is not Announcement

`FlightAnnouncementConfig` — это настройка, а не факт запуска объявления.

```text
Администратор:
FlightDefinition -> FlightAnnouncementConfig -> AnnouncementVariant

Диспетчер:
FlightDefinition + действие -> Announcement -> PlaybackJob
```

`Announcement` остаётся бизнес-сообщением, которое должно быть озвучено.
Конфигурация отвечает только на вопрос:

```text
Какие варианты объявления доступны для этого рейса?
```

### Supported announcement types

На первом этапе конфигурация поддерживает рейсовые типы:

```text
CheckInOpening
CheckInContinuation
CheckInClosing
BoardingInvitation
Arrival
```

Доступность типа зависит от направления рейса:

* для вылета доступны `CheckInOpening`, `CheckInContinuation`,
  `CheckInClosing`, `BoardingInvitation`;
* для прибытия доступен `Arrival`;
* типы, не подходящие направлению рейса, не создаются или считаются
  недоступными.

### Content source

Один языковой вариант объявления может быть подготовлен из:

* готового аудиофайла — `AudioAsset`;
* текста — `Text`.

Текст нужен не только для будущего TTS. Он также позволяет хранить утверждённую
формулировку объявления, даже если аудио ещё не записано.

Независимо от источника, в playback позже должна передаваться уже подготовленная
аудиопоследовательность. Playback и Agent не должны получать текст.

## Domain Model

### FlightAnnouncementConfig

`FlightAnnouncementConfig` представляет настройку одного типа объявления для
одной карточки рейса.

Состояние:

* `id`;
* `flightDefinitionId`;
* `announcementType`;
* `enabled`;
* `repeatRule`, только для типов с повтором;
* `createdAt`;
* `updatedAt`.

Инварианты:

* `flightDefinitionId` обязателен и должен указывать на существующий
  `FlightDefinition`;
* одна карточка рейса не может иметь две активные конфигурации одного
  `announcementType`;
* `announcementType` должен быть совместим с направлением рейса;
* отключённая конфигурация не используется диспетчерским интерфейсом;
* `CheckInContinuation` может иметь правило повтора;
* остальные типы не должны получать правило повтора на этом этапе;
* конфигурацию можно отключить без удаления истории и вариантов.

### AnnouncementVariant

`AnnouncementVariant` представляет один языковой вариант объявления внутри
`FlightAnnouncementConfig`.

Состояние:

* `id`;
* `flightAnnouncementConfigId`;
* `languageCode`;
* `sortOrder`;
* `sourceType`;
* `audioAssetId`;
* `text`;
* `enabled`;
* `createdAt`;
* `updatedAt`.

`sourceType`:

```text
AudioAsset
Text
```

Инварианты:

* язык обязателен и задаётся через `LanguageCode`;
* `sortOrder` определяет порядок будущего воспроизведения языков;
* в одной конфигурации не может быть двух активных вариантов одного языка;
* активный вариант должен иметь источник;
* при `sourceType = AudioAsset` должен быть указан активный `AudioAsset`;
* при `sourceType = AudioAsset` поле `text` не используется;
* при `sourceType = Text` текст обязателен;
* при `sourceType = Text` поле `audioAssetId` не используется;
* отключённый вариант не используется при создании `Announcement`;
* удаление варианта должно быть мягким или запрещаться после появления истории
  использования. Для первого этапа допустимо физическое удаление, если вариант
  ещё не использовался.

### RepeatRule

Для `CheckInContinuation` нужна настройка повтора.

Первый вариант:

```text
repeatEveryMinutes: int
```

Инварианты:

* значение больше нуля;
* значение находится в разумном диапазоне, например от 1 до 120 минут;
* правило повтора не означает автоматического запуска само по себе: оно только
  описывает, как будущий сценарий продолжения регистрации должен планироваться.

## API

Admin endpoints:

```text
GET    /api/v1/admin/flight-definitions/{flightDefinitionId}/announcement-configs
POST   /api/v1/admin/flight-definitions/{flightDefinitionId}/announcement-configs
PATCH  /api/v1/admin/flight-definitions/{flightDefinitionId}/announcement-configs/{configId}
POST   /api/v1/admin/flight-definitions/{flightDefinitionId}/announcement-configs/{configId}/variants
PATCH  /api/v1/admin/flight-definitions/{flightDefinitionId}/announcement-configs/{configId}/variants/{variantId}
DELETE /api/v1/admin/flight-definitions/{flightDefinitionId}/announcement-configs/{configId}/variants/{variantId}
GET    /api/v1/admin/audio-assets
POST   /api/v1/admin/audio-assets
```

Загрузка `AudioAsset` выполняется как `multipart/form-data`:

```text
file: WAV, MP3 или OGG, до 50 МБ
languageCode: BCP 47 language code
```

Create config payload:

```json
{
  "announcementType": "check_in_opening",
  "enabled": true,
  "repeatEveryMinutes": null
}
```

Update config payload:

```json
{
  "enabled": true,
  "repeatEveryMinutes": 6
}
```

Create audio variant payload:

```json
{
  "languageCode": "ro-MD",
  "sortOrder": 1,
  "sourceType": "audio_asset",
  "audioAssetId": "uuid",
  "text": null,
  "enabled": true
}
```

Create text variant payload:

```json
{
  "languageCode": "en",
  "sortOrder": 2,
  "sourceType": "text",
  "audioAssetId": null,
  "text": "Check-in for flight 2M3571 to Antalya is now open.",
  "enabled": true
}
```

Read response should include validation status per config:

```json
{
  "id": "uuid",
  "announcementType": "check_in_opening",
  "enabled": true,
  "isValidForDispatcher": true,
  "validationErrors": [],
  "variants": []
}
```

## Frontend

На странице редактирования рейса добавить вкладку:

```text
Объявления
```

Экран должен показывать карточки типов:

```text
Начало регистрации
Продолжение регистрации
Окончание регистрации
Приглашение на посадку
Прибытие рейса
```

Для каждого типа:

* переключатель `Включено`;
* статус валидности;
* список языковых вариантов;
* кнопка добавления варианта;
* настройка повтора для продолжения регистрации.

Для варианта:

* язык;
* порядок;
* источник:
  * аудиофайл;
  * текст;
* выбор `AudioAsset`, если источник — аудиофайл;
* textarea, если источник — текст;
* включено / выключено.

Типы, не совместимые с направлением рейса, должны быть скрыты или показаны как
недоступные.

## Dispatcher Visibility Rules

Диспетчерский интерфейс в будущей задаче должен получать только действия, для
которых:

* `FlightDefinition` активен;
* `FlightAnnouncementConfig` включена;
* конфигурация совместима с направлением рейса;
* есть хотя бы один активный вариант;
* каждый активный вариант имеет корректный источник;
* обязательные операционные параметры можно выбрать в момент запуска.

Например, если у вылета нет валидной конфигурации `BoardingInvitation`, кнопка
приглашения на посадку для этого рейса не должна быть доступна.

## Domain Events

Внутренние domain events:

* `FlightAnnouncementConfigCreated`;
* `FlightAnnouncementConfigUpdated`;
* `FlightAnnouncementConfigEnabled`;
* `FlightAnnouncementConfigDisabled`;
* `AnnouncementVariantAdded`;
* `AnnouncementVariantUpdated`;
* `AnnouncementVariantEnabled`;
* `AnnouncementVariantDisabled`;
* `AnnouncementVariantRemoved`.

Integration messages не публикуются в этой задаче.

Audit может быть добавлен отдельной задачей, но события должны содержать
достаточно информации, чтобы позже построить audit log:

* пользователь;
* рейс;
* тип объявления;
* язык;
* тип источника;
* изменённые поля.

## Legacy Compatibility

Legacy import должен будет маппить структуру:

```text
C:\anons\звуки\вылеты\Анталия-2M3571\1_MDA\begin.wav
```

в модель:

```text
FlightDefinition: Анталия-2M3571
AnnouncementType: CheckInOpening
LanguageCode: ro-MD
SortOrder: 1
SourceType: AudioAsset
AudioAsset: begin.wav
```

Соответствие файлов типам:

| Legacy file | Announcement type |
| --- | --- |
| `begin.wav` | `CheckInOpening` |
| `continue.wav` | `CheckInContinuation` |
| `end.wav` | `CheckInClosing` |
| `dep.wav` | `BoardingInvitation` |
| `arr.wav` | `Arrival` |

Если в будущем появятся legacy-тексты, они должны импортироваться в тот же
механизм как `sourceType = Text`.

## Definition of Done

* На странице рейса в админке есть вкладка `Объявления`.
* Администратор может создать или изменить конфигурацию объявления рейса.
* Для каждого типа можно добавить несколько языковых вариантов.
* Вариант может ссылаться на `AudioAsset` или содержать текст.
* Администратор может загрузить новый аудиофайл и сразу выбрать созданный
  `AudioAsset`.
* Нельзя сохранить активный вариант без источника.
* Нельзя добавить два активных варианта одного языка в одну конфигурацию.
* Нельзя включить тип объявления, несовместимый с направлением рейса.
* Для `CheckInContinuation` можно задать интервал повтора.
* API возвращает признак валидности конфигурации для будущего диспетчерского
  интерфейса.
* `Announcement` не создаётся при изменении настроек в админке.
* `PlaybackJob` не создаётся при изменении настроек в админке.
* `aeroflow-playback` и `aeroflow-agent` не изменяются.
* Domain, application, persistence, API и frontend tests покрывают основной
  сценарий.
* Полный QA соответствующих репозиториев проходит.

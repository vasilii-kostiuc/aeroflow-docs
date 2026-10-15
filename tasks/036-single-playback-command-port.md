# Один порт для команд из core в playback

Status: Done
Bounded context: Announcements (Core — издатель команд в playback)
Service: aeroflow-core
Change type: Application, Infrastructure (рефакторинг, без изменения поведения)
Depends on: 035 (transactional outbox)

## Goal

Announcements отправляет команды в playback через два порта:

* `PlaybackRequestPublisherInterface` (задачи 013–020): `publish`,
  `publishCancel`, `publishStop`, `publishStopRepeat`;
* `EmergencyModePublisherInterface` (задача 026): `publishActivate`,
  `publishDeactivate`.

Разделение сделано по транспорту: экстренные команды идут своей очередью
`emergency_commands` (ADR 002, уровень 2), и отдельный порт должен был «сделать это
решение видимым в application layer». Но очередь — деталь инфраструктуры:
задача 035 заменила доставку целиком, не тронув ни одного обработчика. Срочность
экстренной команды выражена её типом, а не интерфейсом.

Цена разделения:

* граница одна (Announcements → playback, один издатель, один получатель, один
  владелец контракта), а портов два — и два почти одинаковых outbox-адаптера,
  различающихся только именем очереди;
* изоляции разделение не даёт: каждый потребитель зовёт ровно один метод, так что
  по ISP пришлось бы дойти до порта на команду;
* каждый тест держит свой фейк, реализующий все методы порта (пять фейков в
  `tests/Application/Announcements`);
* новая команда в playback (будущие `Pause`/`Resume`/`SetVolume`, следующие
  emergency-команды) требует нового метода в порту.

## Decisions

* **Один порт на границу** —
  `Announcements\Application\Playback\PlaybackCommandPublisherInterface` с одним
  методом:

  ```php
  interface PlaybackCommandPublisherInterface
  {
      public function publish(PlaybackCommand $command): void;
  }
  ```

* **`PlaybackCommand`** — маркерный интерфейс в
  `Announcements\Application\Playback`, его реализуют все шесть DTO контракта:
  `RequestAnnouncementPlayback`, `CancelAnnouncementPlayback`,
  `StopAnnouncementPlayback`, `StopAnnouncementRepeat`, `ActivateEmergencyMode`,
  `DeactivateEmergencyMode`. Поля и формат DTO не меняются.
* **Очередь выбирает адаптер по классу сообщения**, как раньше routing Messenger:
  один `OutboxPlaybackCommandPublisher` вместо `OutboxPlaybackRequestPublisher` и
  `OutboxEmergencyModePublisher`. `ActivateEmergencyMode`/`DeactivateEmergencyMode` →
  `emergency_commands`, остальные → `announcement_playback`. Команда без
  известной очереди — `LogicException` (ошибка конфигурации, не тихая отправка не
  туда).
* `PlaybackRequestPublisherInterface` и `EmergencyModePublisherInterface` удаляются.
* Обработчики меняют только тип зависимости и вызов: `publishCancel(...)`,
  `publishActivate(...)` и т. п. → `publish(...)`; их логика и порядок не
  меняются. Затронуты
  `PublishAnnouncementPlaybackRequest`, `PublishAnnouncementPlaybackCancel`,
  `PublishAnnouncementPlaybackStopRepeat`, `PublishEmergencyModeActivation`,
  `StopPlaybackHandler`, `DeactivateEmergencyHandler`.
* В тестах — один общий записывающий фейк (`tests/.../Support`) вместо пяти
  локальных.

## Out of Scope

* изменение контрактов и формата сообщений на проводе;
* playback и его порты (`PlaybackEventPublisherInterface`,
  `AgentCommandPublisherInterface`);
* правка docblock-комментариев обработчиков, оставшихся со времён post-commit
  publisher (отдельно).

## Acceptance Criteria

* в core один порт отправки команд в playback с одним методом `publish`;
* все шесть команд попадают в outbox в ту же очередь, что и до задачи, с тем же
  телом;
* экстренные команды по-прежнему идут очередью `emergency_commands`;
* поведение обработчиков не изменилось; тесты application-уровня используют один
  фейк.

## Verification

```bash
# aeroflow-core
php bin/phpunit
```

`OutboxPublishingTest` проверяет очередь и тело для каждой из шести команд.

## Documentation Updates

* `domain-model.md`: упоминания `EmergencyModePublisherInterface` и
  `PlaybackRequestPublisherInterface` (запуск экстренного объявления,
  «Согласованность и транзакции») → `PlaybackCommandPublisherInterface`; в
  «Открытых решениях» — запись о решении (порт на границу, очередь — деталь
  адаптера).

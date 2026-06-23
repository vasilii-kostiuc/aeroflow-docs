# Web User Authentication

Status: Done  
Bounded context: User Access  
Service: aeroflow-web  
Change type: Presentation, Integration, Client Infrastructure

## Goal

Пользователь входит в `aeroflow-web` с существующей учётной записью
`aeroflow-core`, получает клиентскую сессию, работает с защищёнными страницами и
может завершить сессию.

Первый вертикальный результат:

```text
login form
  -> POST /api/v1/login
  -> protected application
  -> authenticated API request
  -> refresh on expired access token
  -> logout
```

## Architectural Classification

Изменение относится к bounded context `User Access`, но не добавляет доменную
модель во frontend.

* изменение является presentation/integration;
* агрегат `User` и жизненный цикл `RefreshToken` остаются в `aeroflow-core`;
* frontend хранит только состояние клиентской сессии и API DTO;
* domain events во frontend не публикуются;
* `UserLoggedIn` и `UserLoggedOut` публикуются существующими use cases core.

Затрагиваемые backend-модели:

* `User` — только через login response DTO;
* `RefreshToken` — только через непрозрачное значение токена.

## Scope

* реализовать форму входа по email и паролю;
* реализовать единый HTTP client для `aeroflow-core`;
* типизировать login, refresh и logout API contracts;
* хранить и восстанавливать клиентскую сессию после перезагрузки страницы;
* прикладывать access token к защищённым API-запросам;
* выполнять одну контролируемую попытку refresh при ответе `401`;
* исключить параллельные refresh-запросы;
* повторять исходный запрос после успешного refresh;
* очищать сессию и переводить пользователя на `/login` после неуспешного refresh;
* защитить маршруты приложения;
* перенаправлять уже авторизованного пользователя с `/login` в приложение;
* реализовать logout из основного layout;
* показывать текущий email пользователя;
* сохранить исходный URL и вернуться на него после успешного входа;
* покрыть критический поток тестами;
* привести frontend-документацию в соответствие с тем, что User Access является
  первым реализуемым web-срезом.

## Out of Scope

* регистрация пользователя через web;
* восстановление и смена пароля;
* управление пользователями;
* назначение и редактирование ролей;
* отдельные интерфейсы администратора и диспетчера;
* детальная role-based навигация;
* изменение backend authentication contract;
* SSO, OAuth и MFA;
* cross-tab синхронизация logout;
* реализация Flight Definitions.

## Existing Core API Contract

Base prefix:

```text
/api/v1
```

Endpoints:

```text
POST /login
POST /token/refresh
POST /logout
```

Login request:

```json
{
  "email": "dispatcher@example.com",
  "password": "password123"
}
```

Login response data:

```json
{
  "accessToken": "jwt",
  "refreshToken": "opaque-token",
  "tokenType": "Bearer",
  "expiresIn": 900,
  "user": {
    "id": "uuid",
    "email": "dispatcher@example.com",
    "roles": ["ROLE_USER"]
  }
}
```

Refresh request:

```json
{
  "refreshToken": "opaque-token"
}
```

Refresh response data:

```json
{
  "accessToken": "new-jwt",
  "refreshToken": "new-opaque-token",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

Logout request:

```json
{
  "refreshToken": "opaque-token"
}
```

Успешный logout возвращает `204 No Content`.

Все успешные JSON-ответы, кроме `204`, обёрнуты в существующий `ApiResponse`.

## Session Storage Decision

Текущий core возвращает access token и refresh token в JSON и принимает refresh
token в request body.

Для первого web-среза:

* token pair и user DTO сохраняются в `localStorage`;
* Zustand store является runtime-представлением сессии;
* access token не извлекается из JWT как источник пользовательских данных;
* токены не логируются и не попадают в URL;
* logout удаляет все сохранённые данные сессии.

Это временный компромисс: JavaScript-доступный refresh token повышает последствия
XSS. Целевой production-контракт — refresh token в `HttpOnly Secure SameSite`
cookie. Переход на cookie требует отдельного согласованного изменения core и web.

## Frontend Use Cases

### Login

1. Проверить обязательность и формат email, обязательность пароля.
2. Отправить credentials в `POST /api/v1/login`.
3. Сохранить token pair и user DTO.
4. Перейти на сохранённый protected URL или на `/`.
5. При `401` показать общую ошибку неверных credentials, не раскрывая, какое
   поле неверно.
6. При network/server error оставить форму заполненной и предложить повторить.

### Restore Session

1. Синхронно прочитать сохранённую сессию до решения protected route.
2. Если данных нет или структура повреждена, считать пользователя
   неавторизованным и очистить storage.
3. Если сессия существует, разрешить открытие protected layout.
4. Действительность access token окончательно подтверждается backend при запросе.

Отдельный endpoint `current user` в core отсутствует, поэтому первый срез
восстанавливает user DTO из login response. Добавление `/me` рассматривается
отдельной задачей.

### Authenticated Request

1. Добавить `Authorization: Bearer <accessToken>`.
2. При первом `401` запустить или дождаться единственного общего refresh promise.
3. После успешной ротации атомарно заменить оба токена.
4. Один раз повторить исходный запрос с новым access token.
5. Не выполнять refresh для login, refresh и logout requests.
6. Не создавать бесконечный retry loop.

### Logout

1. Отправить текущий refresh token в `POST /api/v1/logout`.
2. Независимо от результата запроса очистить локальную сессию.
3. Очистить server-state cache, чтобы данные прошлого пользователя не были
   доступны следующему.
4. Перейти на `/login`.
5. При network/server error не восстанавливать локальную сессию автоматически.

## Routing

Public route:

```text
/login
```

Protected routes:

```text
/
/flight-definitions
```

Правила:

* anonymous user на protected route перенаправляется на `/login`;
* исходный path сохраняется в router state;
* authenticated user на `/login` перенаправляется на `/`;
* неизвестный protected URL после входа остаётся сценарием `not found`;
* проверка ролей не входит в этот task.

## UI States

Login page:

* пустая форма;
* client validation;
* submitting с disabled submit;
* invalid credentials;
* server/network error.

Application layout:

* email текущего пользователя;
* действие «Выйти»;
* состояние logout in progress.

Во время первичной гидратации session store не должен кратковременно показываться
protected content анонимному пользователю.

## Error Model

HTTP client приводит ошибки к общему типу `ApiError`:

```ts
type ApiError = {
  status: number
  code?: string
  message: string
  violations?: Record<string, string[]>
}
```

Правила отображения:

* `401` login — «Неверный email или пароль»;
* `422` — server violations рядом с полями, если поле известно;
* `401` refresh — завершить локальную сессию;
* `403` — отдельный access denied state в будущей задаче;
* network и `5xx` — общая ошибка с возможностью повторить.

## Suggested Code Organization

```text
src/
  app/
    router/
      ui/ProtectedRoute.tsx
      ui/PublicOnlyRoute.tsx
  features/
    auth/
      api/authApi.ts
      model/authStore.ts
      model/types.ts
      model/sessionStorage.ts
      hooks/useLogin.ts
      hooks/useLogout.ts
      ui/LoginForm.tsx
      ui/CurrentUserMenu.tsx
  pages/
    login/LoginPage.tsx
  shared/
    api/apiClient.ts
    api/types.ts
```

Конкретные имена могут быть уточнены при реализации, но зависимости сохраняют
направление:

```text
app/pages -> features -> shared
```

`shared/api` не импортирует auth feature. Доступ к token pair передаётся HTTP
client через небольшой session adapter/callback contract, чтобы не создавать
обратную зависимость.

## Test Scenarios

### Unit

* session storage сохраняет, восстанавливает и очищает валидную сессию;
* повреждённая session data отклоняется и удаляется;
* auth store атомарно заменяет token pair после refresh;
* logout очищает runtime и persisted state.

### Component

* login form валидирует обязательные поля и email;
* submit блокируется на время запроса;
* invalid credentials показываются без раскрытия конкретного неверного поля;
* server violations привязываются к форме;
* email пользователя и logout доступны в layout.

### Integration

* успешный login сохраняет сессию и открывает protected route;
* anonymous user перенаправляется на login;
* после reload сохранённая сессия восстанавливается;
* protected request получает Bearer token;
* один `401` приводит к refresh и повтору исходного запроса;
* несколько одновременных `401` создают один refresh request;
* неуспешный refresh очищает сессию и возвращает на login;
* logout вызывает API, очищает Query cache и session storage;
* неуспешный logout всё равно завершает локальную сессию.

### End-to-End

* пользователь входит и видит защищённый экран;
* пользователь обновляет страницу и остаётся в приложении;
* пользователь выходит и больше не может открыть protected route.

## Tooling Required

Для тестов добавить:

* Vitest;
* React Testing Library;
* Mock Service Worker;
* Playwright — допускается отдельным setup task, если окружение CI ещё не готово.

Добавить scripts:

```text
test
test:run
```

Минимальные проверки:

```text
npm run lint
npm run typecheck
npm run test:run
npm run build
```

## Definition of Done

* login page работает с реальным core API;
* anonymous user не видит protected pages;
* access token добавляется ко всем защищённым API-запросам;
* refresh token ротируется без параллельных refresh-запросов;
* неуспешный refresh завершает сессию;
* logout отзывает refresh token, очищает local session и Query cache;
* токены не логируются и не попадают в URL;
* критические сценарии покрыты тестами;
* lint, typecheck, tests и build проходят;
* `aeroflow-web/architecture.md` больше не утверждает, что Flight Operations —
  первый реализуемый frontend-срез;
* временный риск хранения refresh token в `localStorage` остаётся явно
  задокументирован.

## Decisions

* оба токена временно хранятся в `localStorage` до появления cookie-контракта;
* Playwright setup выносится в отдельную задачу;
* access denied page будет добавлен вместе с role-based routing;
* локальная разработка использует Vite proxy `/api` к `aeroflow-core`.

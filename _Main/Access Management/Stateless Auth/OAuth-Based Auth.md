[[Stateless Auth]]

RFC 6749 - протокол реализации системы авторизации, также стандартизовано бесконечность OAuth-flow для интеграции различных систем посредством данной технологии

Oauth 2.0 решает проблему с тем, что ввод длинной последовательности случайных символов в поле формы делает пользовательский опыт плохим, ровно как и использование API-ключей для доступа к системам

OAuth 1.0

**Автоматизация обмена ключами** - одна из основных проблем, решаемых с помощью OAuth. Она предоставляет клиенту стандартный способ получения ключа с сервера путем выполнения пользователем простого набора действий. Все, что нужно сделать пользователю, это ввести свои учетные данные. За кулисами клиент и сервер обмениваются информацией, чтобы получить от клиента действительный ключ.




**OAuth 2.0** — протокол делегированной авторизации, который на практике часто используют и как опору для аутентификации через токены доступа. Пользователь проходит логин у авторизационного сервера, тот выдает приложению access token (часто в формате JWT), и дальше API доверяет этому токену, не видя пароля пользователя, что позволяет реализовать вход через сторонние провайдеры, тонкую грануляцию прав (scopes) и сценарии как user‑to‑service, так и service‑to‑service, ценой усложненной архитектуры и необходимости отдельного auth‑сервера.

![[Pasted image 20260206212528.png]]


### 3. OAuth 2.0

**Суть:** стандарт авторизации, позволяющий пользователям давать третьим сервисам доступ к своим данным без передачи паролей. Использует токены доступа, которые периодически обновляются.

**Ключевые особенности:**

- поддержка различных типов грантов (код авторизации, PKCE, учётные данные клиента и др.);
- механизм областей (scopes) для ограничения доступа;
- интеграция с JWT для передачи данных.

**Плюсы:**

- гибкость для интеграции со сторонними сервисами;
- высокий уровень безопасности.

**Минусы:**

- сложность реализации;
- необходимость настройки прокси‑механизмов.

**Когда использовать:** авторизация через соцсети, Google, API сторонних сервисов.






---

**OAuth 2.0** - это протокол авторизации, позволяющий сторонним приложениям (клиентам) получать ограниченный доступ к защищённым ресурсам пользователя на сервере без передачи им логина и пароля. Вместо этого используются временные токены доступа, выдаваемые сервером авторизации.

В аутентификации / авторизации по протоколу участвуют: **пользователь**, **клиент**, **сервер** и **OAuth-сервис**.

---

![[Pasted image 20260207190023.png|296]]

1. Разработчик регистрируется в гугле и создает нового клиента, прописывает адреса своего сайта: гугл дает **client ID** и **client secret**. При этом разработчик на своем сервере генерирует гиперссылку для редиректа пользователей, прописывает, доступ к чему именно должен давать нам сервис: например, доступ к гугл диску

```javascript
// .env file
// GOOGLE_CLIENT_ID=your-google-client-id.googleusercontent.com
// GOOGLE_CLIENT_SECRET=your-google-client-secret
// GOOGLE_REDIRECT_URI=http://localhost:3000/auth/google/callback

const GOOGLE_OAUTH_URL = 'https://accounts.google.com/o/oauth2/v2/auth';

function generateGoogleAuthUrl(state = null) {
  const params = new URLSearchParams({
    client_id: process.env.GOOGLE_CLIENT_ID,
    redirect_uri: process.env.GOOGLE_REDIRECT_URI,
    scope: 'openid email profile', // Запрашиваемые scopes
    response_type: 'code',         // Authorization Code Flow
    access_type: 'offline',        // Для получения refresh token
    prompt: 'consent',             // Показывать экран согласия
    state: crypto.randomBytes(32).toString('hex'); // CSRF‑токен
    // CSRF‑токен должен генерироваться и храниться в БД для защиты
  });

  // Добавляем state для CSRF защиты (опционально)
  if (state) params.append('state', state);
  return `${GOOGLE_OAUTH_URL}?${params.toString()}`;
}
```

2. Клиент предоставляет по client.com/login пользователю кнопку для аутентификации по гуглу, пользователь кликает.
3. На кнопке стоит редирект на страницу гугла accounts.google.com/v3/signin, (если **prompt: 'consent'**) пользователь видит страницу с выбором гугл-аккаунта, простановкой галочек для выдачи доступа к ресурсам, вместе с запросом отправляются куки сервиса: `SID`, `HSID`, `SSID`, `APISID`, гугл считывает эти куки, показывает список ваших аккаунтов для выбора
4. Сервис гугла генерирует код пользователя, редиректит нас на сайт, с которого мы пришли, но добавляет код в виде client.com/login/google?code=23846278364t276346239867
5. **Клиент** делает `params.get('code')` и шлет его на **бек**
6. **На сервере** прописан эндпоинт для возвращения кодов вида localhost:8000/auth/google/callback. Сервер забирает код (авторизация успешна) и шлет POST в oauth2.googleapis.com/token и снова шлет данные на сервис google, но в этот раз с

```javascript
url='https://oauth2.googleapis.com/token'
// client_id и client_secret получены на первом этапе
body={
	'client_id':process.env.GOOGEL_CLIENT_ID,
	'client_secret': process.env.GOOGEL_CLIENT_SECRET,
	'grant_type': 'authorization_code,
	'redirect_url': 'http://localhost:3000/login/google',
	'code': code
}
```

9. Сервис гугла возвращает **access_token** (с ним можно ходить в гугл и взаимодействовать с ресурсами, короткоживущий токен), **refresh_token** (живет долго, хранится на бекенде), **scope** (наши возможности), **id_token** (в нем хранится запрошенная нами информация)
10. Сервер расшифровывает **id_token** `jwt.decode(id_token)`, достает все необходимые данные, передает на фронтенд, отпраляет в гугл запросы вида

```javascript
url='https://googleapis.com/drive/files'
headers={
	'Authorization: Bearer ${access_token}'
}
```

1. бекенд создает и отправляет пользователю сессию/токен для авторизации (обычно JWT, который бек генерит сам) с гуглом работает только бек
2. если пользователь был залогинен уже другим способом на сайте, тогда можно вообще ничего не возвращать нового, только сохранить данные аутентификации новым способом. Либо авторизовать пользователя по-новому
3. если мы получали доступ к ресурсам, то бекенд может ничег оне возвращать




<mark style="background:#ff4d4f"><font color="#c00000">исправление</font></mark>

В описанном сценарии OAuth 2.0 Authorization Code Flow с Google **opaque токен не используется на клиентском бэкенде**. Он появляется только при взаимодействии с защищёнными API Google (например, Drive), где ресурсный сервер Google проверяет `access_token` через introspection.​

## Этап использования opaque токена

Opaque токен (неструктурированная случайная строка, не JWT) — это `access_token`, возвращаемый Google в `/token` (шаг 5 описания).  
В вашем flow он используется **после обмена code на токены**: бэкенд отправляет его в заголовке `Authorization: Bearer ${access_token}` к API вроде `https://www.googleapis.com/drive/v3/files`.  
Ресурсный сервер Google (не ваш бэк) интроспектирует его на своём introspection endpoint для проверки валидности, scope и статуса (active/revoked/expired) по RFC 7662





**Шаг 1**: Пользователь на `client.com/login` кликает "Войти через Google" → редирект на `accounts.google.com/o/oauth2/v2/auth`.

**Шаг 2**: Google показывает выбор аккаунта + consent screen (scopes: email, profile, Drive). Пользователь соглашается.

**Шаг 3**: Google редиректит: `http://localhost:3000/auth/google/callback?code=4/0AX4Xf...&state=abc123`

**Шаг 4**: Frontend парсит `code`, отправляет на бэкенд. **Проверяем `state`** из сессии/БД (CSRF защита).

**Шаг 5**: Бэкенд обменивает code на токены:

javascript

`const tokenResponse = await fetch('https://oauth2.googleapis.com/token', {   method: 'POST',  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },  body: new URLSearchParams({    client_id: process.env.GOOGLE_CLIENT_ID,    client_secret: process.env.GOOGLE_CLIENT_SECRET,    code: code,    grant_type: 'authorization_code',    redirect_uri: process.env.GOOGLE_REDIRECT_URI  // Точно callback URI!  }) });`

**Google возвращает**:

- `access_token` (**opaque токен** — случайная строка, используется на следующем шаге)
    
- `refresh_token` (долгоживущий, храним на бэкенде)
    
- `id_token` (JWT с данными пользователя)
    

**Шаг 6**: **Opaque токен (`access_token`) используется здесь** — бэкенд делает запросы к Google API:

javascript

``// ✅ Opaque access_token используется для доступа к Drive fetch('https://www.googleapis.com/drive/v3/files', {   headers: { 'Authorization': `Bearer ${access_token}` } });``

Google Drive сервер интроспектирует opaque токен через внутренний introspection endpoint .

**Шаг 7**: `jwt.decode(id_token)` → получаем `sub`, `email`, `name`. Создаём/находим пользователя в БД.

**Шаг 8**: Генерируем **собственный JWT** для сессии пользователя, возвращаем фронтенду:

javascript

`// Наш внутренний JWT (не связан с Google) const sessionToken = jwt.sign({ userId, provider: 'google' }, process.env.JWT_SECRET);`

## Ключевые исправления

1. **`state`** сохраняется в сессии для CSRF
    
2. **`redirect_uri`** в `/token` — точно callback из .env
    
3. **Opaque токен** = `access_token` из шага 5, используется в заголовке `Bearer` для API[](https://developers.google.com/identity/protocols/oauth2/web-server)​
    
4. Google **не использует** ваши куки для выбора аккаунта (это браузерная сессия Google)
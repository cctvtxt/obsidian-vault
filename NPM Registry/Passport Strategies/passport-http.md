[[passport]]

**passport-http** — это модуль, предоставляющий две встроенные в протокол HTTP стратегии аутентификации: **HTTP Basic** и **HTTP Digest**. Они позволяют защищать эндпоинты без необходимости управлять сессиями.

Схема **HTTP Basic** является простейшей: она передает логин и пароль в заголовке `Authorization`, закодировав их в `Base64`. Важно понимать, что `Base64` — это не шифрование, поэтому данная схема безопасна только при использовании `HTTPS`. Пример ниже демонстрирует ее настройку.

```javascript
const passport = require('passport');
const BasicStrategy = require('passport-http').BasicStrategy;
const User = require('./models/user'); // Ваша модель пользователя

passport.use(new BasicStrategy(
  function(username, password, done) {
    User.findOne({ username: username }, function (err, user) {
      if (err) { return done(err); }
      if (!user || !user.isValidPassword(password)) { return done(null, false); }
      return done(null, user);
    });
  }
));

// Защита роута
app.get('/private', 
  passport.authenticate('basic', { session: false }),
  (req, res) => res.json({ message: "Secret content", user: req.user })
);
```

Схема **HTTP Digest** является более безопасной, так как не передает пароль в открытом виде, а использует challenge-response механизм с хешированием. Однако ее настройка сложнее, и в современных веб-приложениях она практически не встречается, уступив место более гибким методам.

Обе эти стратегии имеют низкую популярность в современных публичных приложениях. Схема `HTTP Basic` может быть полезна для защиты очень простых внутренних API, служебных эндпоинтов или вебхуков, где не требуется сложная система безопасности. `HTTP Digest` же считается устаревшим подходом. Для большинства сценариев `passport-jwt` или `passport-local` с сессиями являются значительно более предпочтительными решениями. Реализовать Basic-аутентификацию на "голом" Node.js очень просто, поэтому использование `passport-http` оправдано в основном для сохранения единообразия кода в проекте, уже использующем Passport.

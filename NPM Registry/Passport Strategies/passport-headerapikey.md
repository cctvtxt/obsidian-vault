[[passport]]

**passport-headerapikey** — это простая стратегия для защиты API-эндпоинтов, требующая наличия API-ключа в HTTP-заголовке. Этот метод отлично подходит для аутентификации сервер-сервер или для предоставления доступа к вашему API сторонним приложениям. Стратегия извлекает API-ключ из указанного HTTP-заголовка (например, `X-API-Key`), после чего передает его в верификационную функцию, где вы можете проверить его и найти связанного с ним пользователя или клиента в вашей базе данных. Это stateless-метод, не требующий сессий.

Пример ниже показывает, как защитить API-роут, требуя валидный ключ в заголовке `X-API-Key`.

```javascript
const express = require('express');
const passport = require('passport');
const HeaderAPIKeyStrategy = require('passport-headerapikey').HeaderAPIKeyStrategy;

const ApiKey = require('./models/apiKey'); // Ваша модель для ключей

const app = express();
app.use(passport.initialize());

passport.use(new HeaderAPIKeyStrategy(
  { header: 'X-API-Key' },
  false, // Не передавать req в callback
  function(apikey, done) {
    ApiKey.findOne({ key: apikey }, function(err, keyRecord) {
      if (err) { return done(err); }
      if (!keyRecord) { return done(null, false); }
      return done(null, keyRecord);
    });
  }
));

app.get('/api/private',
  passport.authenticate('headerapikey', { session: false }),
  function(req, res) {
    res.json({
      message: 'API access granted!',
      client: req.user.clientId 
    });
  }
);
```

Сам по себе паттерн аутентификации по API-ключу в заголовке очень популярен, хотя для его реализации в Passport существует несколько конкурирующих npm-пакетов. `passport-headerapikey` является одним из простых и надежных вариантов. Он идеально подходит для аутентификации программных клиентов, а не живых пользователей, например, при предоставлении доступа к публичному API, для внутренней аутентификации между микросервисами или для простых клиентов вроде IoT-устройств. Написать собственное Express middleware для проверки заголовка — очень простая задача, поэтому использование этой стратегии целесообразно в первую очередь для сохранения единообразия кода, если в проекте уже используется Passport. Важно помнить, что API-ключи должны рассматриваться как пароли: на сервере необходимо хранить только их хеши. Также, в отличие от OAuth, API-ключ аутентифицирует приложение в целом, а не конкретного пользователя.

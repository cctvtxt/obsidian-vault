[[passport]]

**passport-jwt** — это стратегия для аутентификации эндпоинтов с использованием JSON Web Token (JWT), позволяющая защитить API путем проверки валидного токена в заголовках запроса. Этот подход является stateless, то есть не требует хранения сессий на сервере. Стратегия извлекает JWT из запроса, как правило из заголовка `Authorization: Bearer <token>`, после чего проверяет его подпись и срок действия с помощью предоставленного секрета. Если токен оказывается валидным, его полезная нагрузка (payload) передается в callback-функцию для поиска и возврата пользователя.

Ниже представлен пример настройки `passport-jwt` для защиты роутов в приложении Express. Он включает в себя как настройку самой стратегии, так и пример роута для логина, где создается токен, и защищенного роута, доступ к которому требует валидного токена.

```javascript
const express = require('express');
const passport = require('passport');
const JwtStrategy = require('passport-jwt').Strategy;
const ExtractJwt = require('passport-jwt').ExtractJwt;
const jwt = require('jsonwebtoken'); // Для создания токенов

// Предполагается, что у вас есть модель User
const User = require('./models/user'); 

const app = express();
app.use(passport.initialize());

const jwtOptions = {
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  secretOrKey: 'your_jwt_secret' 
};

passport.use(new JwtStrategy(jwtOptions, function(jwt_payload, done) {
  User.findById(jwt_payload.id, function(err, user) {
    if (err) { return done(err, false); }
    if (user) { return done(null, user); } 
    else { return done(null, false); }
  });
}));

app.post('/login', (req, res) => {
  const username = req.body.username;
  const password = req.body.password;
  
  User.findOne({ username }, (err, user) => {
    if (err || !user) {
      return res.status(401).send('Authentication failed.');
    }
    const payload = { id: user.id, username: user.username };
    const token = jwt.sign(payload, jwtOptions.secretOrKey, { expiresIn: '1h' });
    res.json({ message: 'ok', token: token });
  });
});

app.get('/profile', passport.authenticate('jwt', { session: false }), (req, res) => {
  res.send({
    message: 'Success! You are authenticated.',
    user: req.user
  });
});
```

Эта стратегия обладает очень высокой популярностью и является стандартным способом защиты stateless API в экосистеме Node.js. Она идеально подходит для бэкенда, обслуживающего Single Page Applications (SPA) на React, Vue или Angular, а также для мобильных приложений и для аутентификации запросов между микросервисами. Самостоятельная реализация подобной логики на "голом" Node.js потребовала бы написания middleware для извлечения и верификации токена, в то время как `passport-jwt` инкапсулирует эту логику, делая код чище. Ключевой сложностью при работе с JWT является отзыв токена, так как сервер не может просто уничтожить сессию. Эта проблема обычно решается использованием токенов с коротким временем жизни (5-15 минут), ведением черных списков (blacklisting) отозванных токенов в Redis, или построением логики безопасности вокруг механизма Refresh-токенов.

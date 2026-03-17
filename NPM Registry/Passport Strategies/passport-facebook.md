[[passport]]

**passport-facebook** — это стратегия для аутентификации пользователей с помощью их учетных записей Facebook, реализующая протокол OAuth 2.0. Она перенаправляет пользователя на Facebook для аутентификации, и после получения согласия, Facebook возвращает пользователя на сайт с кодом авторизации. Этот код затем обменивается на `access token` и профиль пользователя. Для работы требуется создать приложение на портале Facebook for Developers, чтобы получить `App ID` (clientID) и `App Secret` (clientSecret).

Пример ниже демонстрирует настройку стратегии в приложении Express.

```javascript
const express = require('express');
const passport = require('passport');
const FacebookStrategy = require('passport-facebook').Strategy;
const session = require('express-session');

const User = require('./models/user'); // Ваша модель пользователя

const app = express();

app.use(session({ secret: 'your secret key', resave: false, saveUninitialized: true }));
app.use(passport.initialize());
app.use(passport.session());

passport.serializeUser((user, done) => done(null, user.id));
passport.deserializeUser((id, done) => {
  User.findById(id, (err, user) => done(err, user));
});

passport.use(new FacebookStrategy({
    clientID: process.env.FACEBOOK_APP_ID,
    clientSecret: process.env.FACEBOOK_APP_SECRET,
    callbackURL: "http://localhost:3000/auth/facebook/callback",
    profileFields: ['id', 'displayName', 'emails'] // Явно запрашиваем нужные поля
  },
  function(accessToken, refreshToken, profile, done) {
    User.findOrCreate({ facebookId: profile.id }, function (err, user) {
      return done(err, user);
    });
  }
));

app.get('/auth/facebook', passport.authenticate('facebook', { scope: ['email'] }));

app.get('/auth/facebook/callback',
  passport.authenticate('facebook', { failureRedirect: '/login' }),
  (req, res) => res.redirect('/')
);
```

"Войти через Facebook" — очень популярная функция для приложений, ориентированных на массового потребителя (B2C), таких как e-commerce, новостные сайты и мобильные игры. Основной альтернативой является использование identity-агрегаторов (Auth0, Okta), которые упрощают интеграцию с несколькими социальными сетями. Важно учитывать, что процесс создания и верификации приложения на портале Facebook for Developers может быть сложным и требовать подтверждения компании. Со временем Facebook значительно ограничил объем данных, которые можно получить о пользователе, поэтому необходимо явно запрашивать нужные поля через опцию `profileFields`.

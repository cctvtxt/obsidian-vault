[[passport]]

**passport-google-oauth20** — это стратегия для аутентификации пользователей с помощью их учетных записей Google. Она реализует протокол OAuth 2.0, позволяя пользователям входить на ваш сайт без необходимости создавать новый пароль. Работа стратегии заключается в перенаправлении пользователя на сайт Google для аутентификации. После того как пользователь даст согласие, Google перенаправляет его обратно на ваш сайт с кодом авторизации, который затем обменивается на `access token` и профиль пользователя. Для работы требуется `clientID` и `clientSecret`, которые можно получить в Google Cloud Console.

Пример ниже показывает полную настройку "Войти через Google" в приложении Express, включая настройку сессий, стратегии и необходимых роутов.

```javascript
const express = require('express');
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;
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

passport.use(new GoogleStrategy({
    clientID:     process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: "http://localhost:3000/auth/google/callback"
  },
  function(accessToken, refreshToken, profile, done) {
    User.findOrCreate({ googleId: profile.id }, function (err, user) {
      return done(err, user);
    });
  }
));

// Роут для инициации аутентификации
app.get('/auth/google',
  passport.authenticate('google', { scope: ['profile', 'email'] })
);

// Роут для callback'а
app.get('/auth/google/callback', 
  passport.authenticate('google', { failureRedirect: '/login' }),
  (req, res) => res.redirect('/')
);
```

Эта стратегия является крайне популярной и ожидаемой формой "социального логина" на большинстве современных веб-сервисов. Она идеально подходит для любых публичных приложений, таких как SaaS, e-commerce или социальные платформы, где важно максимально упростить процесс регистрации и входа. Пытаться реализовать протокол OAuth 2.0 вручную — крайне сложная задача, поэтому использование готовых библиотек является единственным разумным подходом. Основной альтернативой является использование комплексных сервисов-агрегаторов (Auth0, Okta), которые позволяют централизованно управлять входом через десятки провайдеров. Важно помнить, что использование этой стратегии несет операционную нагрузку: требуется создать и настроить проект в Google Cloud Console, а также безопасно хранить `clientID` и `clientSecret`. Сам по себе процесс аутентификации с редиректами подразумевает наличие состояния, поэтому стратегия требует использования сессий, например, через `express-session`.

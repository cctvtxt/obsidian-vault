[[OAuth-Based Auth]]

**passport-github2** — это стратегия для аутентификации пользователей с помощью их учетных записей GitHub, использующая протокол OAuth 2.0. Она работает по стандартной схеме: перенаправляет пользователя на GitHub для входа и получения разрешений, после чего GitHub возвращает пользователя на ваш сайт с кодом авторизации. Ваше приложение обменивает этот код на `access token` и данные профиля. Для настройки необходимо создать OAuth App в настройках вашего профиля или организации на GitHub, чтобы получить `clientID` и `clientSecret`.

Следующий пример показывает настройку "Войти через GitHub" в приложении Express.

```javascript
const express = require('express');
const passport = require('passport');
const GitHubStrategy = require('passport-github2').Strategy;
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

passport.use(new GitHubStrategy({
    clientID:     process.env.GITHUB_CLIENT_ID,
    clientSecret: process.env.GITHUB_CLIENT_SECRET,
    callbackURL: "http://localhost:3000/auth/github/callback"
  },
  function(accessToken, refreshToken, profile, done) {
    User.findOrCreate({ githubId: profile.id }, function (err, user) {
      return done(err, user);
    });
  }
));

app.get('/auth/github',
  passport.authenticate('github', { scope: [ 'user:email' ] })
);

app.get('/auth/github/callback', 
  passport.authenticate('github', { failureRedirect: '/login' }),
  (req, res) => res.redirect('/')
);
```

Эта стратегия очень популярна, особенно для сервисов, ориентированных на разработчиков. Она идеально подходит для инструментов (CI/CD, хостинги), образовательных платформ по программированию и технических форумов, так как предоставление входа через GitHub является хорошим тоном для технической аудитории. Как и в случае с другими OAuth-провайдерами, ручная реализация протокола нецелесообразна, а основной альтернативой выступают платформы-агрегаторы вроде Auth0 или Okta. При настройке важно внимательно подходить к запрашиваемым областям доступа (`scopes`), так как `user:email` необходим для получения почты пользователя, а запрос слишком широких прав может отпугнуть пользователей. Для управления сессиями после аутентификации требуется `express-session`.

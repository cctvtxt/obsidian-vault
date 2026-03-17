[[passport]]

**passport-twitter** — это стратегия для аутентификации пользователей с помощью их учетных записей Twitter. Ключевой особенностью является использование протокола OAuth 1.0a, в отличие от большинства современных стратегий, работающих на OAuth 2.0. Процесс включает редирект на сайт Twitter, но механизм получения токенов и подписи запросов отличается. `passport-twitter` инкапсулирует эту сложность. Для работы требуется создать приложение на портале Twitter Developer, чтобы получить `consumerKey` и `consumerSecret`.

Пример ниже демонстрирует настройку стратегии в приложении Express.

```javascript
const express = require('express');
const passport = require('passport');
const TwitterStrategy = require('passport-twitter').Strategy;
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

passport.use(new TwitterStrategy({
    consumerKey: process.env.TWITTER_CONSUMER_KEY,
    consumerSecret: process.env.TWITTER_CONSUMER_SECRET,
    callbackURL: "http://127.0.0.1:3000/auth/twitter/callback"
  },
  function(token, tokenSecret, profile, done) {
    User.findOrCreate({ twitterId: profile.id }, function (err, user) {
      return done(err, user);
    });
  }
));

app.get('/auth/twitter', passport.authenticate('twitter'));

app.get('/auth/twitter/callback', 
  passport.authenticate('twitter', { failureRedirect: '/login' }),
  (req, res) => res.redirect('/')
);
```

Популярность этой стратегии средняя. Она менее распространена, чем вход через Google или Facebook, но остается актуальной для приложений, тесно связанных с медиа, новостями и аналитикой социальных сетей. Основной альтернативой являются агрегаторы аутентификации, такие как Auth0. Важно отметить, что новый API Twitter v2 поддерживает более современный протокол OAuth 2.0, для которого может потребоваться другая библиотека. Использование `passport-twitter` подразумевает работу с OAuth 1.0a, который не использует `Bearer` токены и требует более сложной подписи запросов, что абстрагируется библиотекой. Перед интеграцией необходимо внимательно изучить актуальную политику и ограничения на портале для разработчиков Twitter.

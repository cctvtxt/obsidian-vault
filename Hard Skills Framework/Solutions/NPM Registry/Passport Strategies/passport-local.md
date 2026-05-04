[[Basic Auth]]

**passport-local** — это стратегия для аутентификации пользователя по логину (username) и паролю. Является одной из самых базовых и часто используемых стратегий, лежащей в основе классической сессионной аутентификации. Стратегия `passport-local` получает учетные данные из тела `POST` запроса, после чего ее задачей становится проверка этих данных путем обращения к базе данных. В случае успеха, для пользователя устанавливается сессия, для чего требуется предварительная настройка `passport.serializeUser()` и `passport.deserializeUser()`.

Пример ниже показывает полную настройку `passport-local` в приложении Express, включая конфигурацию сессий, саму стратегию и роуты для логина и доступа к защищенному профилю.

```javascript
const express = require('express');
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;
const session = require('express-session');

// Предполагается, что у вас есть модель User с методом для проверки пароля
const User = require('./models/user'); 

const app = express();

// Настройка сессий
app.use(session({
  secret: 'your secret key',
  resave: false,
  saveUninitialized: false
}));
app.use(passport.initialize());
app.use(passport.session());

// Конфигурация LocalStrategy
passport.use(new LocalStrategy(
  function(username, password, done) {
    User.findOne({ username: username }, function (err, user) {
      if (err) { return done(err); }
      if (!user) { return done(null, false, { message: 'Incorrect username.' }); }
      if (!user.isValidPassword(password)) { return done(null, false, { message: 'Incorrect password.' }); }
      return done(null, user);
    });
  }
));

// Сериализация и десериализация пользователя
passport.serializeUser(function(user, done) {
  done(null, user.id);
});
passport.deserializeUser(function(id, done) {
  User.findById(id, function(err, user) {
    done(err, user);
  });
});

// Роуты
app.post('/login', 
  passport.authenticate('local', { 
    successRedirect: '/profile',
    failureRedirect: '/login',
    failureFlash: true
  })
);

app.get('/profile', (req, res) => {
  if (req.isAuthenticated()) {
    res.send(`<h1>Hello ${req.user.username}</h1>`);
  } else {
    res.redirect('/login');
  }
});
```

Данная стратегия обладает чрезвычайно высокой популярностью и является стандартом де-факто для реализации классической аутентификации в приложениях Node.js. Она идеально подходит для любого веб-приложения, которому требуется собственная база пользователей с логином и паролем, будь то монолитное приложение на Express, традиционный веб-сайт или бэкенд для SPA, использующий сессии. Хотя реализовать подобную логику на "голом" Node.js возможно, создав middleware для проверки `req.body.username` и `req.body.password`, `passport-local` стандартизирует этот процесс и легко интегрируется с другими стратегиями Passport. Поэтому самостоятельная реализация нецелесообразна. Важно помнить, что сама по себе стратегия не занимается хешированием паролей и управлением сессиями. Для полноценной работы ее необходимо использовать в связке с библиотекой для хеширования, например `bcrypt`, и middleware для управления сессиями, таким как `express-session`.

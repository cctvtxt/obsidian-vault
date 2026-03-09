[[MVC]]

passport.js

```javascript
const express = require("express")
const app = express()
const helmet = require("helmet")

const ninetyDaysInSeconds = 90 * 24 * 60 * 60
const directives = {
  defaultSrc: ["'self'"],
  scriptSrc: ["'self'", "trusted-cdn.com"],
}

app.use(
  helmet.noSniff(),
  helmet.hidePoweredBy(),
  helmet.frameguard({ action: "deny" }),
  helmet.xssFilter(),
  helmet.ieNoOpen(),
  helmet.hsts({ maxAge: ninetyDaysInSeconds, force: true }),
  helmet.dnsPrefetchControl(),
  helmet.noCache(),
  helmet.contentSecurityPolicy({ directives }),
)

// 1. Mitigate the Risk of Cross Site Scripting (XSS) Attacks with helmet.xssFilter()

// Cross-site scripting (XSS) is a frequent type of attack where malicious scripts are injected into vulnerable pages, with the purpose of stealing sensitive data like session cookies, or passwords.

// This includes data coming from forms, GET query urls, and even from POST bodies. Sanitizing means that you should find and encode the characters that may be dangerous e.g. <, >.

// The X-XSS-Protection HTTP header is a basic protection. The browser detects a potential injected script using a heuristic filter. If the header is enabled, the browser changes the script code, neutralizing it. It still has limited support.

// 2. Avoid Inferring the Response MIME Type with helmet.noSniff()

// Browsers can use content or MIME sniffing to override response Content-Type headers to guess and process the data using an implicit content type. While this can be convenient in some scenarios, it can also lead to some dangerous attacks.

// This middleware sets the X-Content-Type-Options header to nosniff, instructing the browser to not bypass the provided Content-Type.

// 3. Ask Browsers to Access Your Site via HTTPS Only with helmet.hsts()

// Note: Configuring HTTPS on a custom website requires the acquisition of a domain, and a SSL/TLS Certificate.

// 4. Disable DNS Prefetching with helmet.dnsPrefetchControl()

// 5. Set a Content Security Policy with helmet.contentSecurityPolicy()

// This will protect your app from XSS vulnerabilities, undesired tracking, malicious frames, and much more. CSP works by defining an allowed list of content sources which are trusted.
```

userController

```javascript
import dotenv from "dotenv"
dotenv.config()
const SECRET_KEY = process.env.SECRET_KEY

import UserModel from "../models/User.js"
import jwt from "jsonwebtoken"
import bcrypt from "bcrypt"

export const register = async (req, res) => {
  try {
    const password = req.body.password
    const salt = await bcrypt.genSalt(10)
    const hash = await bcrypt.hash(password, salt)

    const doc = new UserModel({
      email: req.body.email,
      fullName: req.body.fullName,
      passwordHash: hash,
      avatarUrl: req.body.avatarUrl,
    })

    const user = await doc.save()

    const token = jwt.sign(
      {
        _id: user._id,
      },
      SECRET_KEY,
      { expiresIn: `24h` },
    )

    const { passwordHash, ...userData } = user._doc

    res.json({
      ...userData,
      token,
    })
  } catch (e) {
    console.log(e)
    res.status(500).json({
      message: `REGISTRATION FAILED`,
    })
  }
}

export const login = async (req, res) => {
  try {
    const user = await UserModel.findOne({ email: req.body.email })

    if (!user) {
      return res.status(404).json({
        message: `UNSUCCESSFUL`,
      })
    }

    const isValidPass = await bcrypt.compare(
      req.body.password,
      user._doc.passwordHash,
    )

    if (!isValidPass) {
      return res.status(400).json({
        message: `WRONG LOGIN OR PASSWORD`,
      })
    }

    const token = jwt.sign(
      {
        _id: user._id,
      },
      SECRET_KEY,
      { expiresIn: `24h` },
    )

    const { passwordHash, ...userData } = user._doc

    res.json({
      ...userData,
      token,
    })
  } catch (e) {
    console.log(e)
    res.status(500).json({
      message: `AUTHORIZATION FAILED`,
    })
  }
}

export const getMe = async (req, res) => {
  try {
    const user = await UserModel.findById(req.userId)

    if (!user) {
      return res.status(404).json({
        message: `USER DOES NOT EXIST`,
      })
    } else {
      const { passwordHash, ...userData } = user._doc

      return res.json(userData)
    }
  } catch (e) {
    console.log(e)
    res.status(500).json({
      message: `NO ACCESS`,
    })
  }
}
```

```javascript
import express from "express"
import mongoose from "mongoose"
import multer from "multer"
import dotenv from "dotenv"

import { UserController, PostController } from "./controllers/export.js"
import {
  registerValidation,
  loginValidation,
  postingValidation,
} from "./validations.js"
import { handleValidationErrors, checkAuth } from "./utils/export.js"

dotenv.config()
const PORT = process.env.PORT
const DB_CONN = process.env.DB_CONN

mongoose
  .connect(DB_CONN)
  .then(() => console.log("DB connected"))
  .catch((e) => console.log("DB error", e))

const app = express()

const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, "uploads")
  },
  filename: (req, file, cb) => {
    cb(null, file.originalname)
  },
})

const upload = multer({ storage })

app.use(express.json())
app.use("/uploads", express.static("uploads"))

app.post("/upload", upload.single("image"), (req, res) => {
  res.json({
    url: `/uploads/${req.file.originalname}`,
  })
})

app.post(
  "/auth/register",
  registerValidation,
  handleValidationErrors,
  UserController.register,
)
app.post(
  "/auth/login",
  loginValidation,
  handleValidationErrors,
  UserController.login,
)
app.get("/auth/me", checkAuth, UserController.getMe)

app.post(
  "/posts",
  checkAuth,
  postingValidation,
  handleValidationErrors,
  PostController.create,
)
app.get("/posts/:id", PostController.getOne)
app.get("/posts", PostController.getAll)
app.delete("/posts/:id", checkAuth, PostController.remove)
app.patch(
  "/posts/:id",
  checkAuth,
  postingValidation,
  handleValidationErrors,
  PostController.update,
)

app.listen(PORT, (err) => {
  if (err) {
    return console.log(err)
  } else {
    console.log(`SERVER started`)
  }
})
```

server.js

```javascript
"use strict"

const express = require("express")
const bodyParser = require("body-parser")
const cors = require("cors")
require("./db.js")
require("dotenv").config()

const apiRoutes = require("./routes/api.js")
const fccTestingRoutes = require("./routes/fcctesting.js")
const runner = require("./test-runner")

const app = express()
//const nocache = require('nocache')
//app.use(nocache)
app.use("/public", express.static(process.cwd() + "/public"))

app.use(cors({ origin: "*" })) //USED FOR FCC TESTING PURPOSES ONLY!

app.use(bodyParser.json())
app.use(bodyParser.urlencoded({ extended: true }))

//Index page (static HTML)
app.route("/").get(function (req, res) {
  res.sendFile(process.cwd() + "/views/index.html")
})

//For FCC testing purposes
fccTestingRoutes(app)

//Routing for API
apiRoutes(app)

//404 Not Found Middleware
app.use(function (req, res, next) {
  res.status(404).type("text").send("Not Found")
})

//Start our server and tests!
const listener = app.listen(process.env.PORT || 3000, function () {
  console.log("Your app is listening on port " + listener.address().port)
  if (process.env.NODE_ENV === "test") {
    console.log("Running Tests...")
    setTimeout(function () {
      try {
        runner.run()
      } catch (e) {
        console.log("Tests are not valid:")
        console.error(e)
      }
    }, 1500)
  }
})

module.exports = app //for unit/functional testing
```

# Express.js Security Best Practices

Express.js is a fast, unopinionated, minimalist web framework for Node.js. It's widely used to build web applications, and as such, ensuring that applications built with Express.js are secure is paramount. Here, we’ll walk through ten best practices to help you strengthen the security of your Express.js applications.

1. Use Helmet

Helmet helps secure your Express apps by setting various HTTP headers. It's not a silver bullet, but it can help protect against some well-known web vulnerabilities by setting headers like `X-Content-Type-Options`, `X-DNS-Prefetch-Control`, and others.

```
const helmet = require('helmet');
const app = require('express')();

app.use(helmet());
```

## [](https://dev.to/tristankalos/expressjs-security-best-practices-1ja0#2-keep-dependencies-updated)2. Keep Dependencies Updated

Vulnerabilities in software are discovered regularly. Using outdated packages can leave you susceptible to these issues. Use tools like `npm audit` or `Snyk` to identify and update vulnerable dependencies.

```
npm audit fix
```

## [](https://dev.to/tristankalos/expressjs-security-best-practices-1ja0#3-implement-rate-limiting)3. Implement Rate Limiting

Rate limiting helps to prevent brute-force attacks by limiting the number of requests users can make within a certain timeframe. For Express.js, you can use the `express-rate-limit` middleware.

```
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
});

app.use(limiter);
```

## [](https://dev.to/tristankalos/expressjs-security-best-practices-1ja0#4-sanitize-input)4. Sanitize Input

Input sanitization helps prevent injection attacks. It's important to never trust user input and always to sanitize and validate it before processing.

```
const express = require('express');
const app = express();
const { body, validationResult } = require('express-validator');

app.post('/user',
  body('username').isAlphanumeric(),
  body('email').isEmail(),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    // Proceed with handling request
});
```

## [](https://dev.to/tristankalos/expressjs-security-best-practices-1ja0#5-use-https)5. Use HTTPS

HTTP traffic is unencrypted, making it susceptible to interception and alteration. Use HTTPS to encrypt data in transit. With Express.js, you can use Node's `https` module in conjunction with fs to read SSL/TLS certificates.

```
const https = require('https');
const fs = require('fs');
const app = require('express')();

const options = {
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
};

https.createServer(options, app).listen(443);
```

## [](https://dev.to/tristankalos/expressjs-security-best-practices-1ja0#6-handle-errors-properly)6. Handle Errors Properly

Express.js has default error handling, but it's recommended to implement a custom error handling middleware to catch and properly manage errors.

```
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).send('Something broke!');
});
```

## [](https://dev.to/tristankalos/expressjs-security-best-practices-1ja0#7-use-cookies-securely)7. Use Cookies Securely

If you’re using cookies for session management, use secure + httpOnly flags to prevent client-side scripts from accessing the cookie.

```
const session = require('express-session');

app.use(session({
  secret: 'your-secret',
  cookie: {
    secure: true,
    httpOnly: true,
  },
  // other configuration
}));
```

## [](https://dev.to/tristankalos/expressjs-security-best-practices-1ja0#8-avoid-revealing-stack-traces)8. Avoid Revealing Stack Traces

Express.js, by default, will expose a stack trace to the client on an error. Turn off stack traces in production to avoid revealing too much.

```
if (app.get('env') === 'production') {
  app.use((err, req, res, next) => {
    res.status(500).send('Server Error');
  });
}
```

## [](https://dev.to/tristankalos/expressjs-security-best-practices-1ja0#9-secure-file-uploads)9. Secure File Uploads

If your application deals with file uploads, ensure you check the file type, limit the size, and scan for malware.

```
const multer = require('multer');
const upload = multer({
  dest: 'uploads/',
  limits: { fileSize: 1000000 },
  fileFilter: (req, file, callback) => {
    if (!file.originalname.match(/\.(jpg|jpeg|png|gif)$/)) {
      return callback(new Error('Only image files are allowed!'), false);
    }
    callback(null, true);
  }
});
```

## [](https://dev.to/tristankalos/expressjs-security-best-practices-1ja0#10-authenticate-amp-authorize)10. Authenticate & Authorize

Always implement strong authentication and authorization checks. Tools like Passport.js can help you manage these securely.

```
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;

passport.use(new LocalStrategy((username, password, done) => {
  // Implement your verification logic
}));
```

When you follow these best practices, you're not only protecting your application but also safeguarding users' data and trust in your application. Remember, security is an ongoing effort and **requires regular code reviews, monitoring, and updates** to stay ahead of potential threats 😉

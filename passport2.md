# Express auth!

### Learning Objectives

1. Define authentication in the context of a web app.
2. Explain what Passport and Passport strategies are and how they fit into the Express framework.
3. Install Passport and set up a local authentication strategy.
4. Add authentication to Movies, so that users must be logged in in order to add or edit Movies.


## Add a user

# User model

We need to set up a user model to store user account info. Create a new migration which will create the users table. We will be using passport so we need to have a `username` and `password_digest` column.

```sql
-- db/migrations/migration-1503764143787.sql

CREATE TABLE IF NOT EXISTS users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(255) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_digest TEXT NOT NULL
);
```

Make sure to run the migration.

No we have to create the user model.

```javascript
// models/user.js

const db = require('../db/config');

const User = {};

User.findByUserName = userName => {
  return db.oneOrNone(`
    SELECT * FROM users
    WHERE username = $1
  `, [userName]);
};

User.create = user => {
  return db.one(`
    INSERT INTO users
    (username, email, password_digest)
    VALUES ($1, $2, $3)
    RETURNING *
  `, [user.username, user.email, user.password_digest]);
};

module.exports = User;
```

As you can see, our user model only has a `create` and `findByUserName` function. This is because we will not be looking at user profiles or displaying a list of users. We will only be finding a user record when someone tries to sign in. If a user has not yet visited our app, we will allow them to create a user record.

```
git add .
git commit -m "Add user model"
git push
```


##  `Passport` 
is authentication middleware for Node. It is designed to serve a singular purpose: authenticate requests. When writing modules, encapsulation is a virtue, so Passport delegates all other functionality to the application. This separation of concerns keeps code clean and maintainable, and makes Passport extremely easy to integrate into an application. 

### Sessions

- The session is an integral part of a web application.
- It allows data to be passed throughout the application through cookies that are stored on the browser and matched up to a server-side store.
- Usually sessions are used to hold information about the logged in status of users as well as other data that needs to be accessed throughout the app.
- We will be working with [express-session](https://github.com/expressjs/session) to enable sessions within our quotes app.

### Password Encryption

- When storing passwords in your database you **never** want to store plain text passwords. Ever.
- There are a variety of encryption methods available including SHA1, SHA2, and Blowfish.
- Check out this [video on password security](https://www.youtube.com/watch?v=7U-RbOKanYs)

### Using `bcrypt`

- `bcryptjs` is an NPM module that helps us create password hashes to save to our database.
- Let's check out [the documentation](https://www.npmjs.com/package/bcrypt) to learn how to implement this module.
- We will implement this together with [passport](https://www.passportjs.org/) to create an authentication strategy for our Express application.

# Implementing auth with passport

- Passport - Passport is authentication middleware for Node. It is designed to serve a singular purpose: authenticate requests. When writing modules, encapsulation is a virtue, so Passport delegates all other functionality to the application. This separation of concerns keeps code clean and maintainable, and makes Passport extremely easy to integrate into an application. -
  [Passport documentation](http://passportjs.org/docs/overview)
- Passport Strategy - Passport recognizes that each application has unique authentication requirements. Authentication mechanisms, known as strategies, are packaged as individual modules. Applications can choose which strategies to employ, without creating unnecessary dependencies. For example, there are separate strategies for GitHub logins, Facebook logins, etc.

#### Install the dependencies we will be using.

# Setting up passport

```
npm install --save passport passport-local express-session cookie-parser bcryptjs dotenv
```

-  `passport`: express middleware to handle authentication
-  `passport-local`: passport strategy to set up the username-password login flow
-  `bcryptjs` is an NPM module that helps us create password hashes to save to our database.
-  `express-session`: to store our sessions on the express server. 
-  `cookie-parser`: to parse cookies
-  `bcryptjs`: the blowfish encryption package to encrypt and decrypt our passwords. (**NOTE**: There's also a package `bcrypt`. We want `bcryptjs`.)
-  `dotenv`


## Now let's tell our app how to use them.

```javascript
// app.js
const cookieParser = require('cookie-parser');
const session = require('express-session');
const passport = require('passport');

require('dotenv').config();


app.use(cookieParser()); // read cookies (needed for auth)
app.use(bodyParser()); // get information from html forms

app.use(session({
  secret: process.env.SESSION_KEY, // session secret
  resave: false,
  saveUninitialized: true,
}));

app.use(passport.initialize()); // <-- Registers the Passport middleware.
app.use(passport.session()); // persistent login sessions

const authRouter = require('./routes/auth-routes');
app.use('/auth', authRouter);

const authHelpers = require('./services/auth/auth-helpers');
app.use(authHelpers.loginRequired)

// all other routes go below here
```
We are telling express to use the `cookie-parser`. This is similar to `body-parser` but it parses request cookies. Passport stores user auth info in to cookies.

We are also telling express to use `express-session` which will allow use to bounce user auth info back and forth every request, so the user doesn't have to reauthenticate every time they visit a new url on our app.

After we tell the app to use passport and passport sessions, we then require an auth router. This will have all of the routes for signing in and creating (registering) a user.

After that, we are requireing a middleware that we will write called `authHelpers.loginRequired`. We are telling our app to use this function before we tell it to use all of our other routes. This is because we want our app to require a user to be signed in for all routes except the user login and registration pages.

> Note: We haven't written this `auth-helpers` middlware or the `auth-routes` yet. But it's still ok to write this in our app.js file because it will give us a sence of what our next tasks sould be.

Lastly, for this `app.js` to be configures properly, we need a `.env` file with a SESSION_KEY.

```bash
# .env

SESSION_KEY=whatever_key_you_want
```

**Make sure to add this to your `.gitignore!`**


https://stackoverflow.com/questions/18565512/importance-of-session-secret-key-in-express-web-framework

## Why's the session key a secret and why do we need it?
It's used to encrypt the session cookie so that you can be reasonably (but not 100%) sure the cookie isn't a fake one, and the connection should be treated as part of the larger session with express.
This is why you don't put the string in your source code. You make it an environment variable and read it in as process.env("SESSION_SECRET") or you use an .env file with https://npmjs.org/package/habitat, and make sure those files never touch your repository (svn/git exclusion/ignores) so that your secret data stays secret.
the secret is immutable while your node app runs. It's much better to just come up with a long, funny sentence than a UUID, which is generally much shorter than "I didn't think I needed a secret, but the voices in my head told me Express needed one".


## Now that our app.js is in order, we can configure passport.

```
mkdir -p services/auth
touch services/auth/passport.js services/auth/local.js services/auth/auth-helpers.js
```

## Passport

```javascript
// services/auth/passport.js

const passport = require('passport');
const User = require('../../models/user');

module.exports = () => {
  passport.serializeUser((user, done) => {
    done(null, user.username);
  });

  passport.deserializeUser((username, done) => {
    User.findByUserName(username)
      .then(user => {
        done(null, user);
      }).catch(err => {
        done(err, null);
      });
  });
};
```

#### what does serializeUser do?

#### what does deserializeUser do?


## "Local Strategy"
What, you might ask, is a strategy? Perhaps it seems like a strange word to encounter in the context of programming? Perhaps, but in the context it actually fits well.

One of the key features of Passport as an authenication framework is that it is modular and extensible, meaning that it provides a loose framework for programmers to define their own pathways of authentication. It is opinionated about the series of steps that are followed to perform an authentication, but it remains neutral about the specific way that an application authenticates a user.

Why is this a good thing? Well, let's say that in addition to a default username/password login we want to make it possible for users to login through their facebook or google accounts. Each of these methods would represent a unique "Strategy".

```javascript
// services/auth/local.js

const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;

const init = require('./passport');
const User = require('../../models/user');
const authHelpers = require('./auth-helpers');

const options = {};

init();

passport.use(
  new LocalStrategy(options, (username, password, done) => {
    User.findByUserName(username)
      .then(user => {
        if (!user) {
          return done(null, false);
        }
        if (!authHelpers.comparePass(password, user.password_digest)) {
          return done(null, false);
        } else {
          return done(null, user);
        }
      }).catch(err => {
        console.log(err);
        return done(err);
      });
  })
);

module.exports = passport;
```
## Authentication middleware

Now that passport has been told how to find a user and check a password for a user, we can create some authentication middleware.

```javascript
// services/auth/auth-helpers.js

const bcrypt = require('bcryptjs');

function comparePass(userPassword, databasePassword) {
  return bcrypt.compareSync(userPassword, databasePassword);
}

function loginRedirect(req, res, next) {
  if (req.user) return res.redirect('/');
  return next();
}

function loginRequired(req, res, next) {
  if (!req.user) return res.redirect('/auth/login');
  return next();
}

module.exports = {
  comparePass,
  loginRedirect,
  loginRequired,
}
```

The `loginRequired` and `loginRedirect` functions are the actual middleware functions that we are going to insert into our apps function chain to control the flow of a request as it comes in. Remember the Legos! We are going to squeeze these functions somewhere before our route handlers so we can stop the chain if the user is not signed in.

## Auth routes
Now that we have out middleware functions set up, we can create our auth routes.

```
touch routes/auth-routes.js controllers/users-controller.js
```

```javascript
// routes/auth-routes.js

const express = require('express');
const authRouter = express.Router();
const passport = require('../services/auth/local');
const authHelpers = require('../services/auth/auth-helpers');
const usersController = require('../controllers/users-controller');

authRouter.get('/login', authHelpers.loginRedirect, (req, res) => {
  res.render('auth/login');
});

authRouter.get('/register', authHelpers.loginRedirect, (req, res) => {
  res.render('auth/register');
});

authRouter.post('/register', usersController.create);

authRouter.post('/login', passport.authenticate('local', {
    successRedirect: '/movies',
    failureRedirect: '/auth/login',
    failureFlash: true,
  })
);

authRouter.get('/logout', (req, res) => {
  req.logout();
  res.redirect('/');
});

module.exports = authRouter;
```

```javascript
// controllers/users-controller.js

const bcrypt = require('bcryptjs');
const User = require('../models/user.js');

const usersController = {};

usersController.create = (req, res) => {
  const salt = bcrypt.genSaltSync();
  const hash = bcrypt.hashSync(req.body.password, salt);
  User.create({
    username: req.body.username,
    email: req.body.email,
    password_digest: hash,
  }).then(user => {
    req.login(user, (err) => {
      if (err) return next(err);
      res.redirect('/user');
    });
  }).catch(err => {
    console.log(err);
    res.status(500).json({error: err});
  });
}

module.exports = usersController;
```

And now we just need a log in and registration page!

```
mkdir views/auth
touch views/auth/login.ejs views/auth/register.ejs
```

```html
<%# views/auth/login.ejs %>

<form method="POST" action="/auth/login">
  <input name="username" type="text" placeholder="username" required />
  <input name="password" type="password" placeholder="password" required />
  <input type="submit" value="Log in!" />
</form>
```

```html
<%# views/auth/register.ejs %>

<form method="POST" action="/auth/register">
  <input name="username" type="text" placeholder="username" required />
  <input name="email" type="email" placeholder="email" required />
  <input name="password" type="password" placeholder="password" required />
  <input type="submit" value="Register!" />
</form>
```

Great job! Now a user to register and sign into our app!

```
git add .
git commit -m "Add user registration and signin"
git push
```



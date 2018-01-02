# Express auth!

### Learning Objectives

1. Define authentication in the context of a web app.
2. Explain what Passport and Passport strategies are.
3. Install Passport and set up a local authentication strategy.
4. Add authentication to Movies, so that users must be logged in in order to add or edit Movies.

### What's authentication and why would I need it?

Authentication is for identifying users and provide different access rights and content depending on their id. In most cases the application provides a login form with certain credentials to verify a user.


## First we need to create a User model

The first thing we'll do is set up a user model to store user account info. We'll create a new migration to create the users table. We will be using passport so we need to have a `username` and `password_digest` column, and we'll add `email` as well.

Create a migration file with the timestamp using `Date.now()`.

```
touch db/migrations/migration-1503681175400.sql
```
Add in our user info
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

```sql
psql -d movies_auth_dev -f db/migrations/migration-01022018.sql
```


Now we have to create the user model.

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

Our user model only has a `create` and `findByUserName` function. We will only be finding a user record when someone tries to sign in. If a user has not yet visited our app, we will allow them to create a user record.



#  `Passport` 
Passport is authentication middleware for Node. It is designed to serve a singular purpose: authenticate requests. 

### Cookies and Sessions

[helpful stack overflow explanation](https://stackoverflow.com/questions/11142882/how-do-cookies-and-sessions-work)


Cookies and sessions are both ways to preserve the application's state between different requests the browser makes. It's thanks to them that, for instance, you don't need to log in every time you request a page on Facebook.

- Sessions are a way of temporarily persisting data between requests. This is commonly used to 'remember' that a user is logged in.
- It allows data to be passed throughout the application through cookies that are stored on the browser and matched up to a server-side store.
- Usually sessions are used to hold information about the logged in status of users as well as other data that needs to be accessed throughout the app.
- We will be working with [express-session](https://github.com/expressjs/session) to enable sessions within our quotes app.

### Password Encryption

- When storing passwords in your database you **NEVER** want to store plain text passwords.
- There are a variety of encryption methods available including SHA1, SHA2, and Blowfish. 

### Using `bcrypt`

- `bcryptjs` is an NPM module that helps us create password hashes to save to our database.
- We will implement this together with [passport](https://www.passportjs.org/) to create an authentication strategy for our Express application.

#### Hashing and salting
Cryptographic hash functions take a piece of information and return a string, representing this information. Hash values cannot easily be “unhashed” or decrypted and that’s why they are a perfect fit for passwords.

Salt values are random data that is included with the input for the hash function.


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
-  `bcryptjs`: encryption package to encrypt and decrypt our passwords. (**NOTE**: There's also a package `bcrypt`. We want `bcryptjs`.)
-  `dotenv`: a zero-dependency module that loads environment variables from a .env file into process.env.


## Now let's tell our app how to use these dependencies.

```javascript
// server.js
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
Passport stores user auth info into cookies & `cookie-parser` makes sure it's the same cookie every single time, information about user. To tie stuff from movies and directors to our users.

`express-session`  will allow us to bounce user auth info back and forth every request, so the user doesn't have to reauthenticate every time they visit a new url on our app.

`auth router` after we tell the app to use `passport` and `passport sessions`, we then require an auth router. This will have all of the routes for signing in and creating (registering) a user.

`authHelpers.loginRequired` tells  our app to use this function before we tell it to use all of our other routes. This is because we want our app to require a user to be signed in for all routes except the user login and registration pages.

> Note: We haven't written this `auth-helpers` middleware or the `auth-routes` yet. But it's still ok to write this in our app.js file because it will give us a sense of what our next tasks should be.

#### And finally
For this `server.js` to be configured properly, we need a `.env` file with a SESSION_KEY.

```bash
# .env

SESSION_KEY=whatever_key_you_want
```

**Make sure to add this to your `.gitignore!`**


## Why's the session key a secret and why do we need it?
[stack overflow- importance of secret session key](https://stackoverflow.com/questions/18565512/importance-of-session-secret-key-in-express-web-framework)

Used to encrypt the session cookie so that you can be reasonably sure the cookie isn't a fake one, and the connection should be treated as part of the larger session with express.



## Now that our server.js is in order, we can configure passport.
We'll create a services/auth folder and add 3 files to it: `passport.js`, `local.js`, `auth-helpers.js`.


```
mkdir -p services/auth
touch services/auth/passport.js services/auth/local.js services/auth/auth-helpers.js
```

## 1. Passport

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

#### what do serializeUser & deserializeUser do?
[stack overflow- Serialize & deserialize](https://stackoverflow.com/questions/27637609/understanding-passport-serialize-deserialize)



`Serialize`- the functions tell Passport.js how to get information from a user object to store in a session

`Deserialize`- how to take that information and turn it back into a user object 


## 2 "Local Strategy"
What, you might ask, is a strategy? Perhaps it seems like a strange word to encounter in the context of coding? 

- Passport recognizes that each application has unique authentication requirements. Authentication mechanisms, known as strategies, are packaged as individual modules. Applications can choose which strategies to employ, without creating unnecessary dependencies. For example, there are separate strategies for GitHub logins, Facebook logins, etc.

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
## 3. Authentication middleware

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

The `loginRequired` and `loginRedirect` functions are the actual middleware functions that we are going to insert into our apps function chain to control the flow of a request as it comes in. We are going to squeeze these functions somewhere before our route handlers so we can stop the chain if the user is not signed in.

## Auth routes
Now that we have our middleware functions set up, we can create our auth routes.

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
and to our controller

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
      res.redirect('/movies');
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
  
<a href="/auth/register">Register Here</a>  
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



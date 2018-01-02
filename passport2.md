## passport 2

```
touch db/migrations/migration-1514862925942.sql
```

we'll migrate this 

```
psql -d movies_auth_dev -f db/migrations/migration-1514862925942.sql
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

# Setting up passport

Install the dependencies we will be using.

```
npm install --save passport passport-local express-session cookies-parser bcryptjs dotenv
```

Now let's tell our app how to use them.

```javascript
// app.js

require('dotenv').config();

const cookieParser = require('cookie-parser');
app.use(cookieParser());

const session = require('express-session');
app.use(session({
  secret: process.env.SESSION_KEY,
  resave: false,
  saveUninitialized: true,
}));

const passport = require('passport');
app.use(passport.initialize());
app.use(passport.session());

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


# Express auth!

### Learning Objectives

1. Define authentication in the context of a web app.
2. Explain what Passport and Passport strategies are and how they fit into the Express framework.
3. Install Passport and set up a local authentication strategy.
4. Add authentication to Movies, so that users must be logged in in order to add or edit Movies.


##  `Passport` 
is authentication middleware for Node. It is designed to serve a singular purpose: authenticate requests. When writing modules, encapsulation is a virtue, so Passport delegates all other functionality to the application. This separation of concerns keeps code clean and maintainable, and makes Passport extremely easy to integrate into an application. 

Install the dependencies we will be using.

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

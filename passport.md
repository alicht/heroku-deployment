# Passport


The de facto authentication solution in the Express.js world is Passport, which offers a host of strategies for authentication.

Passport is simply authentication middleware, and does not handle any of the other parts of authentication for you: that means the Node.js developer is likely to roll out 
-  their own API token mechanism
-  password reset token mechanisms
-  user authentication routes and endpoints
-  and views in the templating language of their choice

https://hackernoon.com/your-node-js-authentication-tutorial-is-wrong-f1a3bf831a46


# What passport can do

 Storing and recalling credentials is pretty standard fare for identity management, and the traditional way to do this is in your own database or application. Passport, being middleware that simply says “this user is cool” or “this user is not cool”, requires the passport-local module for handling password storage in your own database.
 
 password storage
 
 password reset
 
 https://scotch.io/tutorials/easy-node-authentication-setup-and-local


# What passport CAN'T  do



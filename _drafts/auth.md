---
layout: post
title: Session-Based Authentication
---
###Introduction###
**Authentication** is one of those things that comes up in almost every web application you will build. Also it is one of those things that is extremely important in order to make a **trustworthy** and **secure** application. 

After having the "authentication" discussion again in one of my recent projects I decided to write a blog post series on exactly this topic. In this series I will go over the two most widely used ways of handling authentication. Namely: **session-based authentication** and **token-based authentication**. I hope to give a clear overview of both of those and how they compare and scale.

This blog post will be as the title suggests on the former: session-based authentication. I will go over a simple server implementation and point out common pitfalls and important specifics on the way. I will also go into a little bit more detail on some of the things that I find interesting and important. The code for this post will also be on Github so you can follow along and play around with the code while reading. In the Github repo I will also add a very simple front-end so you can see the code doing it's job, namely handling the authentication. You can get the code [here]().

###Sessions###
As most of you probably hear and read very often HTTP is a stateless protocol. What that means is that any request is completely independent of any other request. There is no state saved across different requests. **Sessions** are a common solution for this problem so that the server can save data for the same user over a number of requests. The way this works is through cookies, which store small amounts of data on the client site for a specific client and website. 

For authentication purposes the server hands out a sessionId, which is signed with a secret key that is kept secure on the server and sends the sessionId back to the client where it is stored in a cookie. Now with every request to the server the client keeps sending the cookie with the request. The server can decrypt the sessionId and check if the user is indeed already authenticated and otherwise redirect him to an authentication page. On the server you try to protect all the routes that contain information you do not want unauthenticated users to be able to access. If someone tries to access those routes you would for example simply want to redirect them to a signin/signup page. Session-based authentication can be a easy and efficient way to do exactly that and protect your site. Combined with SSL it is also very secure.

###Skeleton###
Let's quickly go through the directory structure of the application. For the purposes of this blogpost I have kept all of the server logic and routing, besides database setup, in the `server.js` file. As your application starts to grow you would not want to do that, but modularize the code into different files with as little dependencies on each other as possible.

{% highlight vctreestatus %}

.
|-- .env
|-- .gitignore
|-- db.js
|-- index.html
|-- package.json
|-- server.js
|-- signin.html
`-- signup.html

{% endhighlight %}

We won't look into the html files, but you can check them out on [Github]() and they are on purpose limited to the bare minimum, so they do not distract from the authentication logic. The `.env` file stores all the environment variables that you want to keep sacred. In this application it contains database passwords and username as well as the session secret. It is also included in the `.gitignore` file. The `db.js` file handles the database setup and the `server.js` file contains all the routing and logic for the application.

Enough talking. Let's jump into the code.


###Setup###
First things first. There are a few modules that you need to require in your server.js to make your job easier. You could obviously do all this yourself as well, but this is one of the powers of Node that you have all these great Open Source modules and frameworks that you can use in your code.

{% highlight javascript %}
// a lightweight middleware and routing framework
var express = require("express");

// middleware to populate req.body with the parsed body
// can handle different kinds of data from JSON to URL-encoded
var bodyParser = require("body-parser");
var session = require("express-session");
// cryptographic algorithm module to handle our password encryption
var bcrypt = require("bcrypt-nodejs");
{% endhighlight %}

Especially interesting is the *bcrypt-nodejs* module. It provides a few functions which implement the bcrypt cryptographic algorithm and even handles the salt(an important security extra to guard against [rainbow table attacks](https://en.wikipedia.org/wiki/Rainbow_table)). Bcrypt continues to be one of the best algorithms to hash passwords. One of the reasons is that it is by nature slow and can be made even slower by increasing the number of iterations. The reason that is good is because it makes a brute-force attack much harder. Since it is slow an attacker will also need much longer to test each key.

{% highlight javascript %}
var app = express();
app.use(bodyParser.urlencoded({
  extended: false
}));
// require our database that we set up in a different file
var db = require("./db.js");
{% endhighlight %}

For this specific example we also won't need much set up. The *body-parser* middleware is set up to handle `urlencoded data. HTML forms send their data in application/x-www-form-urlencoded`, which can be parsed by the bodyParser middleware. `req.body` will be populated with the data from the form so we can read out username and password.

Also we need to set up our mongoDB database that we will use to store our username and password as well as our sessions. This code snippet handles that.

{% highlight javascript %}
// again grab the things we need
// mongoose is a mongoDB object modeling package for Node that makes 
// the interaction with mongoDB very easy
var mongoose = require("mongoose");
var habitat = require("habitat");
var env = habitat.load(__dirname + "/.env");

var dbuser = env.get("db_user");
var dbpassword = env.get("db_password");
// connection url for our mongolab database
var dbUri = "mongodb://"+dbuser+":"+dbpassword+"@ds045614.mongolab.com:45614/authentication-tut";

var db = {};

mongoose.connect(dbUri);
db.connection = mongoose.connection;
// schema for our user data
db.userSchema = new mongoose.Schema({
  "username": String, 
  "password": String
});
db.User = mongoose.model("User", db.userSchema);
// export the important database things
module.exports = db;
{% endhighlight %}

Do not worry about *habitat* for now, I will quickly explain what it does later. Just know that it helps access private variables that you do not want to show in your source code. For this application we use mongolab for hosting. It is extremely simple to use and great for playing around. It get's extremely expensive quickly though and you are better off hosting your own on AWS or something similiar if you store more data. The file exports the database connection and user model so we can use them in other files.

###Initialization of *express-session*###

Below you can see the initialization of sessions for our application. Express middleware handles nearly all of the session processing for us. The only thing we have to do is pass in a few options that specify how sessions should be handled.

{% highlight javascript %}
// our session data store
var MongoStore = require('connect-mongo')(session);
var habitat = require("habitat");
var env = habitat.load(__dirname + "/.env");

app.use(session({
  // reuse the same mongo connection from earlier
  store: new MongoStore({ mongooseConnection: db.connection }),
  secret: env.get("session_secret"),
  resave: false,
  saveUninitialized: true
  // in production environment use cookie: { secure: true } 
  // in order to require https connection for cookie
}));
{% endhighlight %}

I am not going to go into too much detail here, but I would love to point out a few very important points that we have to take into account when using *express-session*. The first one is **store**. By default all the session data will be stored in RAM. As you can probably guess that can start to get problematic relatively quickly. For example all the session data will be gone if the server would be to crash. Even more importantly though if your application scales you probably won't only be using one single process, server, virtual machine anymore and you also do not want to fill your RAM with the session-data from thousands of users. A decent and easy option to handle that problem is to hook up your session storage with a database. There are a lot of session storage database wrappers out there for everyones favourite database so you can get a database hooked up easily. In this instance I have choosen to use mongo-connect, just simply because we had the mongo database already running for storing our users. Now we can just reuse that same connection we have just established and tada your session storage is ready to roll out. One downside of using this storage mechanisms is that it will be much slower than the in-memory storage version, since you have to make a request to a database over the network every single time. If you are just developing for yourself think about maybe setting up a database on your local machine, which will speed up the process.

The other thing I would love to point out is the secret. The secret is used to encrypt and decrypt the session cookie, to make sure the cookie-id you are receiving is indeed one you also handed out. One thing you generally should not do is put your secret right into your source code. If someone has access to your sourcecode, he or she will also have access to your secret and will be able to fake cookies. A handy way to store your secret is for example using habitat, which is a library for managing environment variables. You can for example store your environment variables in an .env file which you add to your .gitignore and have habitat handle the reading out of that data for you. That's it for the session setup. 

Moving on to the routes.

###Routes###

We will need to handle 6 different routes for our application. Only one of them needs protection for now. But depending on if your application interacts with the server a lot you would probably have a lot more protected routes. 

{% highlight javascript %}
app.get("/", isLoggedIn, function(req, res) {
  res.sendFile(__dirname + "/index.html");
});

app.get("/signin", function(req, res) {
  res.sendFile(__dirname + "/signin.html");
});

app.get("/signup", function(req, res) {
  res.sendFile(__dirname + "/signup.html");
});
{% endhighlight %}

In the previous code snippet we serve all the static files. Those are the `index.html` file, which in this case just shows you a message congratulating you that you successfully logged in. As you can see we have a middleware function that is called every time you try to access the `/` route or `/index.html` route. `/signup` and `/signin` just serve the forms for the authentication process. Those are also the routes you get redirected to if you are not authenticated.

###User Handling###

#####Signup#####
Let's get to the bread and butter of authentication. I am not going to explain too much as it is pretty much self-explanatory, but read the code carefully.

{% highlight javascript %}
app.post("/api/signup", function(req, res) {
  // grab username and password from parsed body object
  var username = req.body.username;
  var password = req.body.password;
  
  // search for user with specified username
  db.User.findOne({
    username: username
  }, function(err, user) {
    if (user) {
      res.redirect("/signup");
    } else {
      // bcrypt hash function is called with data to be encrypted
      // the salt if you want to specify one yourself 
      // a progress callback that periodically gets called during hashing and a handling callback
      bcrypt.hash(password, null, null, function(err, hash) {
      	// create the user with the hashed password
        var user = new db.User({
          username: username,
          password: hash
        });
        
        //save the user to the database and create a session
        user.save(function(err, user) {
          if (err) throw err;
          req.session.regenerate(function(err) {
            if (err) throw err;
            // an hour in milliseconds
            req.session.maxAge = 3600000;
            req.session.user = user;
            res.redirect("/");
          });
        });
      });
    }
  });
});
{% endhighlight %}

First we check if the username is already taken. If so we redirect the user back to the `/signup` page for another try. A good practice is to show the user a message what went wrong (e.g. "Username already exists"). I keep this out of this tutorial for simplicity reasons but it is not hard to do and there are a few modules like *express-flash* for example out there to do this. 

If the username is not taken we hash the password using the *bcrypt-nodejs* module I introduced earlier. It provides this easy to use function `hash`, which asynchronously creates a unique salt hashes the salt-password agglomeration and calls the callback with the hashed password.

We store the user to the database and create a session for the user and make it expire in one hour. Setting an expiration date is good practice regarding security, but you obviously do not want to annoy your users. One hour always seems to be a good compromise. Rather go for less here though. 

Now we know the user is authenticated and can redirect him to our secured route.

#####Signin#####

{% highlight javascript %}
app.post("/api/signin", function(req, res) {
  var username = req.body.username;
  var password = req.body.password;

  db.User.findOne({
    username: username
  }, function(err, user) {
    if (!user) {
      res.redirect("/signin");
    } else {
      bcrypt.compare(password, user.password, function(err, isMatch) {
        if (isMatch) {
          req.session.regenerate(function() {
            req.session.user = user;
            res.redirect("/");
          });
        }
      });
    }
  });
});
{% endhighlight %}

Signin is pretty much the same as `/signup` only that we do it the other way around checking the password for the already registered user and depending on the result redirect the user to either the `/signin` page or our secured home route.

Last but not least we will want to create the `isLoggedIn` function that protects our home route. Also we will need to start the server.

{% highlight javascript %}
function isLoggedIn(req, res, next) {
  if (req.session && req.session.user) {
    next();
  } else {
    res.redirect("/signin");
  }
}


app.listen(port, function() {
  console.log("Server listening on Port " + port);
});
{% endhighlight %}

The *express-session* middleware populates req.session with the user for every request if the received cookie contains a sessionId that is valid and not timed out yet. That means the only thing we need to do is check req.session.user. If it is populated we can be sure that the user is indeed already authenticated and allow access. Otherwise again we redirect to `/signin`.

###Wrapping up###
As you saw the *express-session* module does take a lot of work off our back so we do not have to worry about reading, writing and decrypting sessionId and data. I hope there was something interesting in this blogpost for everyone. If you have questions hit me up on <a href="mailto:arthur.mathies@googlemail.com">arthur.mathies@googlemail.com</a>.

Also some final words of advice. All this is only secure if you send your data over an encrypted connection. Always use an SSL or TLS encrypted connection when you send passwords or other sensitive data. That means at least securing your signup or signin forms and their requests to the server.

Thanks for reading my first blog post and talk to you soon in one of my hopefully many blog posts to come. 


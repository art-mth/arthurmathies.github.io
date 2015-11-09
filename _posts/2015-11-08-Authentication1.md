---
layout: post
title: Session-Based Authentication
---
###Introduction###
**Authentication** is one of those topics that comes up in almost every web application you will build. There are many ways to handle authentication and the one you chose will depend on the required balance between user **security** and **convenience**.

After having the "authentication" discussion again in one of my recent projects, I decided to write a blog post series on this noteworthy topic. In this series, I will go over the two most widely used ways of handling authentication, **session-based authentication** and **token-based authentication**. I hope to give a clear overview of both of these and how they compare and scale.

This blog post will be, as the title suggests, on the former- session-based authentication. I will go over a simple server implementation and point out common pitfalls and important specifics on the way. I will also go into a little bit more detail on some of the things that I find interesting and important. The code for this post will also be available on Github so you can follow along and play around with the code while reading. In the Github repo, I will also add a very simple front-end so you can see the authentication in action. You can view and download the code [here](https://github.com/arthurmathies/session-based-authentication-tut).

###Sessions###
As most of you probably hear and read very often, HTTP is a stateless protocol. What that means is that one request is completely independent of another request; there is no state saved across different requests. **Sessions** are a common solution for this problem, so that the server can save data for the same user over a number of requests. The way this works is through cookies, which store small amounts of data on the client site for a specific client and website. 

For authentication purposes, the server hands out a Session ID, which is signed with a secret key and  kept secure on the server. It then sends the Session ID back to the client, where it is stored in a cookie. Now with every request to the server, the client sends the cookie along with the request. From here,the server can decrypt the Session ID, check if the user is indeed already authenticated, and if not redirect him to an authentication page. On the server, you must always try to protect all routes that contain information you do not want unauthenticated users to be able to access. If someone tries to access those routes you can, for example, simply redirect them to a signin/signup page. Session-based authentication can be a easy and efficient way to do all the former and protect your site from unauthenticated users. One thing to point out though, is that any authentication method will fail if you do not encrypt your connection properly. This is a topic for another blog post though, for now just keep that in mind.

###Skeleton###
Let's quickly go through the directory structure of the application. For the purposes of this blogpost, I have kept all of the server logic and routing, besides database setup, in the `server.js` file. As your application starts to grow, you would not want to do that, but modularize the code into different files with as little dependencies on each other as possible.

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

For the purposes of this post, we won't look into the html files, but feel free to check them out on [Github](https://github.com/arthurmathies/session-based-authentication-tut). They are  purposely limited to the bare minimum, so they do not distract from the authentication logic. 

The `.env` file stores all the environment variables that you want to keep sacred. In this application, it contains database usernames and passwords as well as the session secret. This file should always be added to your .gitignore file so you do not push it to your remote accidentally. 

The `db.js` file handles the database connection and setup of the needed schemas and models.

Finally the `server.js` file contains all the routing and logic for the application.

Enough talking. Let's jump into the code.


###Setup###
First things first-there are a few modules that you need to require in your `server.js` to make your job easier. You could obviously do all of this yourself, but one of the powers of Node is that you have all these great Open Source modules and frameworks available to use in your code.

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

Especially interesting is the *bcrypt-nodejs* module. It provides a few functions which implement the bcrypt cryptographic algorithm and even handles the salt (an important security extra to guard against [rainbow table attacks](https://en.wikipedia.org/wiki/Rainbow_table)). Bcrypt continues to be one of the best algorithms to hash passwords. One of the reasons is that it is, by nature, slow and can be made even slower by increasing the number of iterations. The reason slow is good is because it makes a brute-force attack much harder.

{% highlight javascript %}
var app = express();
app.use(bodyParser.urlencoded({
  extended: false
}));
// require our database that we set up in a different file
var db = require("./db.js");
{% endhighlight %}

For this specific example, we won't need much set up. The *body-parser* middleware is set up to handle urlencoded data. HTML forms send their data in `application/x-www-form-urlencoded`, which can be parsed by the bodyParser middleware, and  `req.body` will be populated with the data from the form so we can read out the username and password.

We also need to set up our mongoDB database that will be used to store the username and password as well as our sessions. This code snippet handles that.

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
// put in your own database URI here and store
// username and password in the .env file.
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

Do not worry about *habitat* for now, I will explain what it does later. Just know that it helps access private variables that you do not want to show in your source code. For this application we use MongoLab for hosting. It is extremely simple to use and great for playing around. The downside is that MongoLab gets extremely expensive quickly and with more data you are better off hosting your own on AWS or something similar. The file exports the database connection and user model so we can use them throughout our application.

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

I am not going to go into too much detail here, but I would love to point out a few very important elements that we have to take into account when using *express-session*. The first one is store. By default all the session data will be stored in RAM. As you can probably guess, that can start to get problematic relatively quickly. For example, all the session data will be gone if the server would be to crash. Even more importantly though, if your application scales you probably won't be using only one single process, server, virtual machine anymore and you also do not want to fill your RAM with the session-data from thousands of users. A decent and easy option to handle that problem is to hook up your session storage with a database. There are a lot of session storage database wrappers out there for that easily connect with your  favourite database. In this instance, I have chosen to use mongo-connect, just simply because we have the mongo database already running for storing our users. Now we can reuse that same connection we have already established and tada- your session storage is ready to roll out. One downside of using this storage mechanism is that it will be much slower than the in-memory storage version, since you have to make a request to a database over the network every single time. If you are just developing for personal use you can consider setting up a database on your local machine, which will speed up the process.

The other thing I would love to point out is the secret. The secret is used to encrypt and decrypt the session cookie, to make sure the Session ID you are receiving is indeed one you also handed out. One thing you generally should not do is put your secret right into your source code. If someone has access to your source code, he or she will also have access to your secret and will be able to fake cookies. A handy way to store your secret is using habitat, which is a library for managing environment variables. You can store your environment variables in an .env file which you add to your .gitignore and have habitat handle the reading out of that data for you. That's it for the session setup. Moving on to the routes.

###Routes###

We will need to handle 6 different routes for our application. For now, only one of them needs protection but depending on if your application interacts with the server a lot you would probably have a lot more protected routes. 

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
Let's get to the bread and butter of authentication. I am not going to explain the following too much as it is pretty self-explanatory, but be sure to read the code carefully.

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

First we check if the username is already taken. If so, we redirect the user back to the `/signup` page for another try. A good practice is to show the user a message that indicates what went wrong (e.g. "Username already exists"). I kept that out of this tutorial for simplicity reasons, but it is not hard to do and there are a few modules, such as *express-flash*, out there to do this. 

If the username is not taken, we hash the password using the *bcrypt-nodejs* module I introduced earlier. It provides an easy to use function, `hash`, which asynchronously creates a unique salt, hashes the salt-password agglomeration, and calls the callback function with the hashed password.

We store the user in the database, create a session for the user, and make it expire in one hour. Setting an expiration date is good practice for security purposes, but you obviously do not want to annoy your users. One hour always seems to be a good compromise.

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

`/signin` is pretty much the same as `/signup` only that we do it the other way around- checking the password for the already registered user and, depending on the result, redirect the user to the appropriate page.

Last but not least, we will want to create the `isLoggedIn` function that protects our home route. 
We also need to start the server.

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

The *express-session* middleware populates req.session with the user for every request if the received cookie contains a Session ID that is valid and not timed out yet. That means the only thing we need to do is check req.session.user. If it is populated we can be sure that the user is indeed already authenticated and allow access. Otherwise again we redirect to `/signin`.

###Wrapping up###
As you saw, the *express-session* module takes a lot of work off our back so we do not have to worry about reading, writing and decrypting Session ID’s and data. 


Some final words of advice- all of this is only secure if you send your data over an encrypted connection. Always use an SSL or TLS encrypted connection when you send passwords or other sensitive data. That means at least securing your signup or signin forms and their requests to the server.

I hope there was something interesting in this blogpost for everyone. If you have questions hit me up on <a href="mailto:arthur.mathies@googlemail.com">arthur.mathies@googlemail.com</a>.


Thanks for reading my first blog post and talk to you soon in one of my many blog posts to come. 

Arthur



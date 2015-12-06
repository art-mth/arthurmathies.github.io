---
layout: post
title: Token-Based Authentication
---
###Introduction
**JSON Web Tokens** have been the hype for quite some time now and maybe fairly so. **Token-based authentication** can have considerable merits over session-based authentication in the world of **API's**, **mobile phones** and **connected gadgets** we live in. But also for our web applications they can be a great way to handle authentication. 

This will be the second post in my series about authentication in web applications and before we get started I would highly advise everyone who has not read the previous post on session-based authentication to do so as I will not go over all the details I pointed out already in that blogpost. Please check it out [here](http://arthurmathies.com/2015/11/08/Authentication1/).

###JSON Web Tokens
JWT is an open standard for passing around self-contained, securely signed JSON data. In it's simplest terms you can think of JWT's as Base64Url encoded strings which contain all the data you want them to contain as well as a signature which is signed with a private key in order for anyone who has access to the private key to verify the validity of the tokens. JWT's are based on the idea of message authentication codes (MAC's), which are pieces of information providing authenticity as well as integrity assurance of the message. In JWT's the signature is created by taking the header: the first part of the token (contains general information about the token), the payload: the second part of the token (contains all the information you pass in) both Base64Url encoded and combining them with the private key/secret and running them through a hashing algorithm. In that way anyone who has the private key can repeat that exact procedure and therefore verify the token.

###The Benefits of Tokens
Token authentication is stateless. In the last blog post we went into much detail about how we could best store the session data on the server side without sacrificing scalability and speed. If you remember we decided to hook up a database just for our session data in order to make it possible for more than one server/process to handle the same users. Since we had to make another request over the network to retrieve the information about the user the process of routing and requesting information from the server became much slower. With Tokens we do not have this problem anymore because all of the data we need from the user is part of the token that we handed out when the user signed in and which is sent along with the clients requests to the server. The only thing we need to do is verify the token using our private key which is accessible on all of our servers, decode the information and voila we have everything we need. 

The fact that JWT's are self-contained and based on JSON also makes them usable across many different devices and platforms. Even if you would be to build an application for the smartphone or smartwatch where cookie handling is much harder (there is no specified standard for cookies) you could pass around the JWT's and store them in whatever way makes sense on the specific device.

Last but not least tokens make it extremely easy to set access rights and give privileges to other applications; You can just encode JSON information in the payload of the token. Tokens can be passed along and access can easily be extended to various different applications and devices as needed. Most of those things can also be achieved using sessions, but the self-contained nature of the tokens makes it much easier to pass them around and scale your applications.

###The Application
Again I set up an application, which puts JSON Web Tokens into action. I used AngularJS for the front-end and tried to make the application structure scalable by modularising even this simple application. It should give you a good guideline on how you could structure such an application and should help you get up and running with token-based authentication. Feel free to check out the code on [Github](https://github.com/arthurmathies/token-based-authentication-tut). There are again exactly the same 3 views as in the previous tutorial with sessions: signup, signin and home.

###Skeleton

{% highlight vctreestatus %}

|-- .env
|-- .gitignore
|-- LICENSE.md
|-- README.md
|-- client
|   |-- app.js
|   |-- auth
|   |   `-- authFactory.js
|   |-- home
|   |   |-- home.html
|   |   |-- home.js
|   |   `-- homeFactory.js
|   |-- index.html
|   |-- signin
|   |   |-- signin.html
|   |   |-- signin.js
|   |   `-- signinFactory.js
|   `-- signup
|       |-- signup.html
|       |-- signup.js
|       `-- signupFactory.js
|-- database
|   |-- database.js
|   |-- db.sqlite
|   `-- model
|       `-- user.js
|-- package.json
`-- server
    |-- auth
    |   |-- authCtrl.js
    |   `-- authRouter.js
    |-- config
    |   |-- middlewareConfig.js
    |   `-- routeConfig.js
    |-- data
    |   |-- dataCtrl.js
    |   `-- dataRouter.js
    `-- server.js
    
{% endhighlight %}

As you can see the application structure is much more complicated than the one in the last blog post. As I mentioned that is because I wanted to make the structure as close to how you would structure a bigger application as possible. Some people do not like putting the html with the corresponding javascript/angular files but I have found this structuring by views to be the best way to keep being productive and stay sane as the application grows bigger. Otherwise you will spend more of your time searching for files than actually writing code. This just makes it much easier to built a feature in one of the views as all the needed files or at least most of them are already there. Also I modularized the server in order to make the it more scalable. Normally all my backend logic happens in the Controllers and the rest is mostly routing and supporting middleware. You will understand it better when you see it so let's get started!

###Server

####Setup
For this application I decided to use SQLite as the database to switch it up a little. This will not be the focus of this post, but it should not be too hard to follow since we are using Sequelize (a SQL ORM) which makes the syntax very similiar to mongoose, the MongoDB object modeling tool from the last blog post.

First let's look at the setup.

{% highlight javascript %}
/*

server.js
express/node server 

*/

// using .dotenv instead of habitat this time to load environment variables
// never put environment variables in your source code
require('dotenv').config({
  path: __dirname + '/../.env'
});

// lightweight middleware and routing framework for Node
var express = require('express');

var port = process.env.PORT || 3000;
var app = express();

// setup and configure database
var db = require(__dirname + '/../database/database.js');

// configure middleware
require(__dirname + '/config/middlewareConfig.js')(app, express);
// configure routes
require(__dirname + '/config/routeConfig.js')(app, express, db);

app.listen(port, function() {
  console.log('Server is listening on port: ' + port);
});
{% endhighlight %}

Nothing fancy here. In `server.js` we required the baseline dependencies(`express` and `dotenv`) and start the server. Look at the way we inject the app and our database into the `routeConfig.js` or `middlewareConfig.js`. This form of dependency injection makes it really easy to keep our coupling between different parts of the application to a minimum.

####Configuration

I like to use a distinct file to require all the helper middleware that my application requires. The `middlewareConfig.js` file in this case only sets up the `bodyParser` module to parse request bodies to JSON, but as your application grows you will have much more middleware to configure which you can do in this file.

{% highlight javascript %}

/*

middlewareConfig.js
configures middleware for our app

 */

// middleware to populate req.body with the parsed body
// can handle different kinds of data from json to URL-encoded
var bodyParser = require('body-parser');

module.exports = function(app, express) {

  // configure app to use bodyParser middleware for JSON data
  app.use(bodyParser.json());

};

{% endhighlight %}

#####routeConfig.js
{% highlight javascript %}

/*

routeConfig.js
configure routes for our app
 
 */

module.exports = function(app, express, db) {
  // serve all static files from the client folder
  app.use(express.static(__dirname + '/../../client'));

  // handling all authentication (signup, signin, route protection)
  var authRouter = express.Router();
  require(__dirname + '/../auth/authRouter.js')(authRouter, db);
  app.use('/api/auth', authRouter);

  // authCtrl contains function 'authenticate' to check user access rights
  var authCtrl = require(__dirname + '/../auth/authCtrl.js')(db);

  // handling all data requests for this application
  var dataRouter = express.Router();
  // protect all /api/data routes
  dataRouter.use(authCtrl.authenticate);
  require(__dirname + '/../data/dataRouter.js')(dataRouter);
  app.use('/api/data', dataRouter);
};

{% endhighlight %}

All our static files are served by `express.static` a neat helper that express supplies out of the box. The rest of our routes act as an API backend for our application. We can protect all specific routes we want to by specifying the `authCtrl.authenticate` function as middleware. Here we only protect our `/api/data` routes that serve the data for our home view. 

#####authCtrl.authenticate
{% highlight javascript %}

authenticate: function(req, res, next) {
  // pull out token
  var token = req.body.token || req.query.token || req.headers['x-access-token'];

  if (token) {
    // decode and verify token
    jwt.verify(token, process.env.TOKEN_SECRET, function(err, decoded) {
      if (err) {
        // failed decoding token --> error or token not handed out by server
        res.status(403).json({
          success: false
        });
      } else {
        // token successfully decoded
        // set user on req for further middleware to use
        req.user = decoded;
        console.log('Successfully authenticated token, access granted for user: ' + req.user.username);
        // move on to next middleware
        next();
      }
    });
  } else {
    // no token supplied
    res.status(403).send({
      success: false
    });
  }
}
    
{% endhighlight %}
   
The function acts as our authentication module on the server and checks if the token provided is indeed one we also handed out by verifying the token signature with the `TOKEN_SECRET` we store securely in our .env file. If the token is one that is signed with the secret, therefore one we handed out, it sets the decoded token object which is the payload we specified when we created the token on the request body for further middleware to be able to access the user information. You will see the process of creating the token in the next code snippet.

####Signup
{% highlight javascript %}

/*

authCtrl.js
configuring routes for authRouter

*/

// promise library to avoid callback hell
var Promise = require('bluebird');
// module for securely hashing passwords
var bcrypt = require('bcrypt-nodejs');
// module implementing authO jwt's
var jwt = require('jsonwebtoken');

module.exports = function(db) {
  return {
    signup: function(req, res) {
      // pull out user data
      var username = req.body.username;
      var password = req.body.password;
      // query for user with username(unique)
      db.User.findOne({
          where: {
            username: username
          }
        })
        .then(function(user) {
          console.log('user with username: ' + username + ' exists: ' + !!user);
          if (user) {
            // username is already taken
            res.json({
              success: false
            });
          } else {
            var hashing = Promise.promisify(bcrypt.hash);

            // hash password and save user to the database
            hashing(password, null, null)
              .then(function(hash) {
                // create user in db
                db.User.create({
                    username: username,
                    password: hash
                  })
                  .then(function(user) {
                    console.log('user with username: ' + username + ' got created: ' + !!user);
                    // create token with username and id
                    var signUser = {
                      id: user.id,
                      username: user.username
                    };
                    jwt.sign(signUser, process.env.TOKEN_SECRET, {
                      expiresIn: '1h'
                    }, function(token) {
                      res.json({
                        success: true,
                        token: token
                      });
                    });
                  });
              });
          }
        });
    },
    //signin and authenticate
 };
};

{% endhighlight %}

When the user signs up we create a token, set an expiration date on it and send the token to the client. The `jsonwebtoken` module, which is developed against `draft-ietf-oauth-json-web-token-08` is great and provides all the functionality we need in order to handle the verification and creation of tokens.

I will not go over the signin as well as it is very similiar to the signup, but you can definitely check it out on the [Github](https://github.com/arthurmathies/token-based-authentication-tut) repo.

###Client
I will not go over all the angular views, but I want to go over the code in the `app.js` file and one of the factories to show you the way we interact with the server from the client side.

####app.js
{% highlight javascript %}

angular.module('App', [
  'ui.router',
  'App.home',
  'App.homeFactory',
  'App.signin',
  'App.signinFactory',
  'App.signup',
  'App.signupFactory',
  'App.authFactory'
])

.config(['$stateProvider', '$urlRouterProvider', '$httpProvider', function($stateProvider, $urlRouterProvider, $httpProvider) {

  $urlRouterProvider
    .otherwise('/');

  $stateProvider
    .state('signin', {
      url: '/signin',
      templateUrl: 'signin/signin.html',
      controller: 'signinCtrl',
      controllerAs: 'signinCtrl',
      data: {
        authenticate: false
      }
    })
    .state('signup', {
      url: '/signup',
      templateUrl: 'signup/signup.html',
      controller: 'signupCtrl',
      controllerAs: 'signupCtrl',
      data: {
        authenticate: false
      }
    })
    .state('home', {
      url: '/',
      templateUrl: 'home/home.html',
      controller: 'homeCtrl',
      controllerAs: 'homeCtrl',
      data: {
        authenticate: true
      }
    });

  // intercepts outgoing http request and sets token header
  $httpProvider.interceptors.push(['$window', function($window) {
    return {
      request: function(config) {
        var jwt = $window.localStorage.getItem('authtoken');
        if (jwt) {
          config.headers['x-access-token'] = jwt;
        }
        return config;
      }
    };
  }]);
}])

// on route changes the user authentication status is checked and the user is redirected accordingly
.run(['$rootScope', '$state', 'authFactory', function($rootScope, $state, authFactory) {
  $rootScope.$on('$stateChangeStart', function(event, toState) {
    if (toState.data.authenticate && !authFactory.isAuthenticated()) {
      $state.go('signin');
      event.preventDefault();
    }
  });
}]);

{% endhighlight %}

In `app.js` we initialize the application, configure it and handle the routes. In `.config` we create an interceptor on our `$httpProvider` module in order to include the token we got from the server after signing in  with every request that we make to the server by default. Also important is the run function. The run function is executed after all of the services have been configured and the injector has been created. It is where we do all global configurations for services before we start loading the controllers and directives. In this case we setup a listener on `$stateChangeStart` which pretty much means listens for a change in `$state`. This is where we check if the user is authenticated when he tries to change state. For example if the user is not authenticated he should not be able to get into a `$state` which is not signup or signin.

In case you were wondering: ui-router is a routing framework for Angular and is not part of Angular by default. It is much more widely used than the standard `ng-route` that ships with Angular. The reason is that it is possible to have nested and parallel views which is not possible with `ng-route`. That is something that you will almost always want in any non trivial application. Instead of routes with `ui-route` you have states. You can then include views in your html by using `ui-view` and route using `ui-sref`. If you want to know more about `ui-view` definitely check out their Github page.

####Signup
Now quickly onto the signup view and logic. 

####signup.html
{% highlight html %}

<div class="signup">
  <h1>Sign Up</h1>
  <form ng-submit="signupCtrl.signup()">
    <input type="text" name="username" placeholder="username..." autocomplete="off" ng-model="signupCtrl.user.username">
    <input type="password" name="password" placeholder="password..." autocomplete="off" ng-model="signupCtrl.user.password">
    <button type="submit">Sign Up</button>
  </form>
  <span>Already a User?<a ui-sref="signin">Sign In</a></span>
</div>

{% endhighlight %}

Basic html form with two input fields username and password. As you can see the `signup()` function is bound to the `ng-submit` event and the input fields are bound to the models on the controller.

####signup.js
{% highlight javascript %}

angular.module('App.signup', [])

.controller('signupCtrl', ['signupFactory', '$window', '$state', function(signupFactory, $window, $state) {
  var self = this;

  self.user = {};

  self.signup = function() {
    signupFactory.signup(self.user)
      .then(function(data) {
        if (data.success) {
          $window.localStorage.setItem('authtoken', data.token);
          $state.go('home');
        } else {
          self.user = {};
        }
      });
  };
}]);

{% endhighlight %}

####signupFactory.js
{% highlight javascript %}

angular.module('App.signupFactory', [])

.factory('signupFactory', ['$http', function($http) {
  var signup = function(user) {
    return $http({
        method: 'POST',
        url: '/api/auth/signup',
        data: user
      })
      .then(function(resp) {
        return resp.data;
      });
  };

  return {
    signup: signup
  };
}]);

{% endhighlight %}

As you can see there is not too much logic for this view or actually any other for this basic application and most of the more advanced stuff, namely the `$httpInterceptor` and the `$routeChangeStart` event are handled in the `app.js`. I encourage everyone to check out the tutorial code as a whole on [Github](https://github.com/arthurmathies/token-based-authentication-tut) and try to play around with it on your own machine. At least for me that always seems to be the best way to learn it.

###Wrapping Up
As we saw using tokens or in specific JSON Web Tokens can be beneficial in some circumstances. I would not write off session-based authentication completely and it is still extremely easy and secure to set up session-based authentication especially if we are not working on a Single-Page Application. Whenever you need great scalability or platform independence though tokens seem to be the way to go at least for now.

In this tutorial we stored the token in local storage. This can sometimes lead to security problems since local storage is available from the javascript code which can make it the point of attack for XSS(Cross Site Scripting). Another possibility is to store the token in a cookie. That diminishes some of the benefits but is a more secure option since we can set httpOnly flags on the cookies to make them not accessible through JavaScript.

I hope I have shared some valuable infos on the topic of authentication in web applications and hope I have motivated some of you to build more secure authentication for your applications. 

If you have any comments or questions please send me a message at <a href="mailto:arthur.mathies@googlemail.com">arthur.mathies@googlemail.com</a>.

Thanks for reading and until next time.

Arthur
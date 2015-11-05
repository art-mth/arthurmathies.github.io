#Authentication

Authentication is one of those things that comes up in almost every web application you will build. Also it is one of those things that is extremely important in order to make a trustworthy and secure application. 

After having the "authentication" discussion again in one of my recent projects I decided to write a blog post series on exactly this topic. In this series I will go over the two  most widely used ways of handling authentication. Namely: session-based authentication and token-based authentication. I hope to give a clear overview of both of those and how they compare and scale.

Quick introduction to doing authentication using sessions


Below you can see the initialization of sessions for our application. Express middleware handles nearly all of the session processing for us. The only thing we have to do is pass in a few options that specify how sessions should be handled.

{% highlight javascript %}
var MongoStore = require('connect-mongo')(session);
var habitat = require("habitat");
var env = habitat.load(__dirname + "/.env");

app.use(session({
  store: new MongoStore({ mongooseConnection: db.connection }),
  secret: env.get("session_secret"),
  resave: false,
  saveUninitialized: true
}));
{% endhighlight %}

I am not going to go into too much detail here, but I would love to point out a few very important points that we have to take into account when using express sessions. The first one is "store". By default all the session data will be stored in memory. As you can probably guess that can start to get problematic relatively quickly. For example all the session data will be gone if the server would be to crash. Even more importantly though if your application scales you probably won't only be using one single server anymore and you also do not want to fill your RAM with the session-data from thousands of users. A decent and easy option to handle that problem is to hook up your session storage with a database. There are a lot of session storage database wrappers out there for everyones favourite database so you can get a database hooked up easily. In this instance I have choosen to use mongo-connect, just simply because we had the mongo database already running for storing our users. Now we can just reuse that same connection we have just established and tada your session storage is ready to roll out. One downside of using this storage mechanisms is that it will be much slower than the In-Memory storage version, since you have to make a request to a database over the network every single time. If you are just developing for yourself think about maybe setting up a database on your local machine, which will speed up the process.

The other thing I would love to point out is the secret. The secret is used to encrypt and decrypt the session cookie, to make sure the cookie-id you are receiving is indeed one you also handed out. One thing you generally should not do is put your secret right into your source code. If someone has access to your sourcecode, he or she will also have access to your secret and will be able to fake cookies. A handy way to store your secret is for example using habitat, which is a library for managing environment variables. You can for example store your environment variables in an .env file which you add to your .gitignore and have habitat handle the reading out of that data for you. That's it for the session setup. Let's move on to...

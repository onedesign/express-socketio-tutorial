# Express Generator and Socket.io

Recently we went about adding socket.io to a site scaffolding created with [Express application generator](https://github.com/expressjs/generator). Socket.io's documentation is pretty good, but doesn't "just work" with the Express generator setup.

While in the end it's a trivial fix, a quick google didn't provide much for answers so I figured I'd put together a simple guide on [Express](http://expressjs.com/) and [Socket.io](http://socket.io/) using the command line [Express generator](https://github.com/expressjs/generator).

Express has been my go to node framework for some time, it's replaced Sinatra as my un-opinionated web framework of choice. Express is extremely un-opiniontated out of the box so in my default setup I add some bells and whistles such as automagic controller setup, a standard app/* dir for controllers, routes, models, and views and a few more things to tweak it to my workflow.

I'm not gonna talk about those today though. For sake of keeping it clean, and leaving you with a boilerplate you can fit in your own world I'm gonna stick as close to out of the box as I can. So without further ado let's make some stuff.

## Prerequisites.

You're gonna need [nodejs](https://nodejs.org) installed. If you haven't already done that head on over to [the nodejs website](https://nodejs.org) and install for your system.

This tutorial assumes basic knowledge of node and the command line, but I'm gonna try and be as verbose as possible so non techie types can give it a go.

Also, I have us test the example using curl. It just feels cooler, more like someone else is triggering the socket. If you don't have curl installed you can just visit the url in a separate tab in the browser.

## Generating your express project

Cool, now that that's out of the way we're gonna set up the basics.

```
$ sudo npm install express-generator -g
```

O.K. I'm assuming you're familiar with NPM at this point. But just in case you aren't NPM is node's default package manger. The command above does a couple of things. it uses **npm** to install **express-generator** it's using **sudo** since the **-g** option is telling it to install "globally".

Now let's use express generator to set up our app.

```
$ express myApp
```

This sets up a barebones express application in the folder myApp. Let's go there now.

```
$ cd myApp
```

Now let's add socket.io to our app. Note the **--save** flag to save it to our package.json file as a dependancy.

```
$ npm install socket.io --save
```

Now that we've added socket.io to our package.json let's install all the default dependencies too.

```
$ npm install
```

And for kicks let's launch our application.

```
$ node app.js
```

If all went well you should have an express app running at http://localhost:3000 check that out in your browser. 

Let's kill the server so we can set up socket.io (If you don't know how to do this try hitting Ctrl+c at the command prompt).

## Adding the websocket server

Now for the fun part. open app.js in your editor of choice. Lines 1-11 should look like this:

```
var express = require('express');
var path = require('path');
var favicon = require('serve-favicon');
var logger = require('morgan');
var cookieParser = require('cookie-parser');
var bodyParser = require('body-parser');

var routes = require('./routes/index');
var users = require('./routes/users');

var app = express();
```

Pretty straightforward stuff here. Require all our necessary modules. Add a route for index and users and create the express app.

Let's go ahead and add socket.io to the app.

We're going to add the following lines below our app variable:

```
var server = require('http').Server(app);
var io = require('socket.io')(server);
```

The first line creates our apps http server. This is normally done in bin/www but we're going to move it here. The second sets up our websockets server to run in the same app. When you're lines 1-13 should look like so.

```
var express = require('express');
var path = require('path');
var favicon = require('serve-favicon');
var logger = require('morgan');
var cookieParser = require('cookie-parser');
var bodyParser = require('body-parser');

var routes = require('./routes/index');
var users = require('./routes/users');

var app = express();
var server = require('http').Server(app);
var io = require('socket.io')(server);
```

Cool, now we have to pass our server into bin/www so let's move on down to the bottom of the file. It should look like this:

```
app.use(function(err, req, res, next) {
  res.status(err.status || 500);
  res.render('error', {
    message: err.message,
    error: {}
  });
});


module.exports = app;
```

See that last line where we're exporting the app? well we want to export our server there as well since we're declaring it on line #12 instead of bin/www. That will make your last line look like this:

```
module.exports = {app: app, server: server};
```

Next, open bin/www in your editor.

Lines 1-22 should look like this:

```
#!/usr/bin/env node

/**
 * Module dependencies.
 */

var app = require('../app');
var debug = require('debug')('www:server');
var http = require('http');

/**
 * Get port from environment and store in Express.
 */

var port = normalizePort(process.env.PORT || '3000');
app.set('port', port);

/**
 * Create HTTP server.
 */

var server = http.createServer(app);
```

Notice two things are wrong here. First we're no longer just returning our express app but an object containing app and server. So let's change line #7 from:

```
var app = require('../app');
```

To:

```
var app = require('../app').app;
```

Second, we're declaring a second server on line 22:

```
var server = http.createServer(app);
```

Let's change that to require the instance we created in app.js that now contains our socket.io server as well.

```
var server = require('../app').server;
```

That should do it. Lines 1-22 should now look like this:


```
#!/usr/bin/env node

/**
 * Module dependencies.
 */

var app = require('../app').app;
var debug = require('debug')('www:server');
var http = require('http');

/**
 * Get port from environment and store in Express.
 */

var port = normalizePort(process.env.PORT || '3000');
app.set('port', port);

/**
 * Create HTTP server.
 */

var server = require('../app').server;
```

Cool, let's fire up our server and make sure everything still works. If it does you should see the same express message you saw at http://localhost:3000 previously.

Now that we have our pieces in place, let's wire up a simple socket.

We'll start by passing our socket to our response in middleware. Open app.js back up and start a new line on line 21 and add the following. This simply adds socket.io to res in our event loop.

```
app.use(function(req, res, next){
  res.io = io;
  next();
});
```

Still nothing to see though. Let's add something fun. Save that and open up routes/users.js. It should look like this:

```
var express = require('express');
var router = express.Router();

/* GET users listing. */
router.get('/', function(req, res, next) {
  res.send('respond with a resource.');
});

module.exports = router;
```

Remember how we added socket.io to our response object a minute ago? We'll now we can use it to respond to routed information via a websocket. Keen.

```
var express = require('express');
var router = express.Router();

/* GET users listing. */
router.get('/', function(req, res, next) {
  res.io.emit("socketToMe", "users");
  res.send('respond with a resource.');
});

module.exports = router;
```

Now let's add a socket listener to our layout. Here's what it should look like now:


```
extends layout

block content
  h1= title
  p Welcome to #{title}
``` 

Let's add the following JS:

```
extends layout

block content
  h1= title
  p Welcome to #{title}
  script(src="/socket.io/socket.io.js")
  script.
    var socket = io('//localhost:3004');
    socket.on('socketToMe', function (data) {
      console.log(data);
    });
``` 

Great. I think that's it, let's restart the server and see if this works.

First open up your homepage at http://localhost:3000. Open up dev tools and watch your JS console.

Now pop open another terminal and type:

```
$ curl http://localhost:3000/users
```

If all went well you should see your dev tools console output "users".

That's it, you're ready to start building cool socket enabled stuff. Pretty easy right?





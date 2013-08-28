# Node.js workshop

## Welcome

Please get seated and make sure you have installed Node.js from http://nodejs.org on your computer.

Program:

- Node fundamentals
- Break
- Writing a server
- The Node.js ecosystem
- MAGE introduction
- Q&A


## Chrome and console

* Released in 2008 by Google
* The open source V8 JavaScript engine (JIT compiled)
* Let's open Chrome and the console


## Node.js

* Released in 2009 by Ryan Dahl
* JavaScript on the server
* The same V8 JavaScript engine that powers Chrome
* Strong points: networking, language of the web, modules
* Let's open an interactive node shell


## Functions as values

	function sum(a, b) {
		return a + b;
	}

	sum = function (a, b) {
		return a + b;
	};
	
	console.log(sum);


## Closures

JavaScript supports closures, and makes things like this possible:

	var sum = 0;

	[1, 2, 3].forEach(function (num) {
		sum += num;
	});

	console.log(sum);  // outputs 6

The function given to forEach we call a "callback". It's called for each element during the execution of forEach. When the callback has been called 3 times, forEach returns and we print the sum to our console.

Our function has access to variables outside its own scope. We call the space in which the sum variable is available to inner functions a "closure".


## Sleeping in JavaScript

This does not exist:

	console.log('Hello');
	sleep(1);
	console.log('world!);

> Q: Does anybody know why?


## Multithreading vs. Event Loop

### Multithreading

	sleep(1);

	result = db.query('SELECT * FROM table');

	data = fs.readFile('./index.html');

	content = http.get('http://www.google.com');

My execution thread is blocked, waiting for the result, while we could be executing other code! The solution: create threads.

In a web server where I have 1000 concurrent requests, that gives me:

- 1000 threads in parallel
- 1000 times thread create/release overhead
- 1000 call stacks in my memory
- (race conditions, shared memory danger, locks, deadlocks)

### Event loop

Another solution is the reactor pattern, or "event loop".

	console.log('Hello');

	setTimeout(function () {
		console.log('world!);
	}, 2000);

Our setTimeout example does not create a thread. It registers a function to be executed when 2000 milliseconds have passed. In the mean time, we can execute other code. JavaScript (and therefore Node.js) only runs 1 function at a time.

So this works too:

	setTimeout(function () {
		console.log('world!);
	}, 2000);

	console.log('Hello');

This single threaded approach keeps our memory footprint incredibly low.
A good example: Apache webserver (pre-fork or multithreaded) vs. Nginx (eventy) as illustrated here:

![apache vs nginx!](./nginx-apache-memory.png)

Node.js performs all I/O through an event loop:

* Timers
* TCP, UDP, DNS
* File system


## Callbacks

So the function we passed to setTimeout was a callback. Because the implementation of setTimeout schedules a time based event, it calls our function when that event occurs. In the mean time, we can continue executing code. The way we get our result from setTimeout, we call asynchronous, because it happens after our function has finished executing.

Besides callbacks, Node.js offers one more model for dealing with asynchronous responses: event emitters.


## Events

In the browser:

	document.body.addEventListener('click', function (event) {
		console.log('clicked!');
	});

*click*

In Node.js:

	process.on('exit', function () {
		console.log('Goodbye, friends!');
	});


## Streams

For dealing with large quantities of data (strings or bytes), Node.js also gives us a built-in API called Streams. One typical example of a Readable Stream:

	var query = db.query('SELECT * FROM myTable');

	query.on('row', function (row) {
		console.log('Found a row:', row);
	});

This can be incredibly useful when dealing with lots of data, while still allowing us to use very little memory.


## So... single threaded? But I have 64 cores!

Actually, you have many more cores than that. Your application or game server will often be running on many servers simultaneously. This means we have to write our application to scale across the network.

If we can do that, we can naturally scale across processes as well. Not one process per user request like PHP, but one persistently running daemon per CPU core.

Node.js has a built-in module for this called "cluster" which makes this really easy.


## A simple HTTP server

The HTTP server exposes both Readable and Writeable streams.

	var http = require('http');

	var server = http.createServer(function (req, res) {
		res.writeHead(200, { 'content-type': 'text/plain' });
		res.end('Hello world!\n');
	});

	server.listen(12345);

```sh
curl -i http://localhost:12345
ulimit -n 10000
siege -n 10 http://localhost:12345
siege -n 500 http://localhost:12345
```

## A streaming HTTP server

First step:

	var http = require('http');

	var server = http.createServer(function (req, res) {
		res.writeHead(200, { 'content-type': 'text/plain' });
		res.write('Hello\n');

		setTimeout(function () {
			res.end('World!\n');
		}, 2000);
	});

	server.listen(12345);

```sh
curl -i http://localhost:12345
siege -n 10 http://localhost:12345
```

Next step:

	var http = require('http');

	var server = http.createServer(function (req, res) {
		res.writeHead(200, { 'content-type': 'text/plain' });
		res.write('Hello\n');

		setInterval(function () {
			res.write('World!\n');
		}, 1000);
	});

	server.listen(12345);


## A simple TCP chat server

chatserver.js

	var net = require('net');

	var sockets = [];

	function broadcast(sender, msg) {
		sockets.forEach(function (socket) {
			if (socket !== sender) {
				socket.write(sender.remoteAddress + '> ' + msg + '\n');
			}
		});
	}


	var server = net.createServer(function (socket) {
		sockets.push(socket);

		broadcast(socket, 'I entered the room\n');

		socket.on('data', function (data) {
			broadcast(socket, data);
		});
		
		socket.on('end', function () {
			var index = sockets.indexOf(socket);
			if (index !== -1) {
				sockets.splice(index, 1);
			}
		});
	});

	server.listen(12345);

```sh
nc myip:12345
telnet myip 12345
```

## What else is in Node?

Have a look at [http://nodejs.org/docs/latest/api/](http://nodejs.org/docs/latest/api/)!

* File system
* TCP, UDP, DNS
* HTTP, HTTPS
* Child process
* Readline (keyboard input)
* ZLIB
* ... and not much more actually.


## NPM

Node Package Manager: http://npmjs.org

> Currently around 39000 open source packages

Packages include support for:

* MySQL, Memcached, Redis, MongoDB, CouchDB, etc
* WebSocket
* HTTP2 (spdy)
* OAuth
* ZeroMQ, RabbitMQ, etc


## JSHint

Let's install a package using NPM.

```sh
npm install -g jshint
```

There is a tool called "lint" that has been developed for many programming languages. It does static code analysis on your codebase. That's a fancy word for saying: it finds situations that a compiler will consider legal, but are still a common source of bugs.

JavaScript has a linter called "jslint" and a slightly more modernized version called "jshint" which we will use.

> demo


## Meet MAGE!

Meet MAGE, your Magical Asynchronous Game Engine!

Why?

- We believe social game production, deployment and operation should be easy and painless

How?

- Easy to start working on a new project
- Easy interaction with data through APIs and tools
- Built-in health monitoring
- Built-in performance measuring
- Solves the hard infrastructure problems automatically


### Setting up a MAGE environment

MAGE can be up and running in less than a minute.

```sh
nvm use 0.8.25
mkdir simplegame
cd simplegame
BOOTSTRAP=true npm install ../mage
```

### The dashboard

* Let's open up the documentation
* Logger
* Inspect our configuration
* Database access through Archivist
* Asset management


### Let's open up some code

1. Open up an editor and navigate to our www files.
2. Loader (PhoneGap), landing, main.
3. The main page is where we will write most of our game UI.

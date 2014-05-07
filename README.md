# workerpool

JavaScript is based upon a single event loop which executes one event at a time. All I/O operations are evented, asynchronous, and non-blocking, while the execution of non-I/O code itself is executed sequentially. Jeremy Epstein explains this clearly in the blog [Node.js itself is blocking, only its I/O is non-blocking](http://greenash.net.au/thoughts/2012/11/nodejs-itself-is-blocking-only-its-io-is-non-blocking/):

> In Node.js everything runs in parallel, except your code.
> What this means is that all I/O code that you write in Node.js is non-blocking,
> while (conversely) all non-I/O code that you write in Node.js is blocking.

This means that CPU heavy tasks will block other tasks from being executed. In case of a browser environment, the browser will not react to user events like a mouse click while executing a CPU intensive task (the browser "hangs"). In case of a node.js server, the server will not respond to any requests while executing a single, heavy request.

For front-end processes, this is not a desired situation.
CPU heavy tasks should be offloaded from the main event loop onto dedicated *workers*. We can use [Web Workers](http://www.html5rocks.com/en/tutorials/workers/basics/) when in a browser environment, and [child processes](http://nodejs.org/api/child_process.html) when using node.js. Effectively, this results in an architecture which achieves concurrency by means of isolated processes and message passing.

workerpool offers an easy way to use a pool of workers for both dynamically offloading computations, as well as managing a pool of dedicated workers. All logic to manage a pool of workers is hidden, whilst the workers can be accessed via a natural, promise based proxy, as if they are available locally.

workerpool runs on node.js, Chrome, Firefox, Opera, Safari, and IE10+.


## Features

- Simple to use
- Can be used in both browser and node.js environment
- Dynamically offload functions to a worker
- Workers are accessible via a proxy
- Automatically restores crashed workers


## Install

Install via npm:

    npm install workerpool


## Load

To load workerpool in a node.js application (both main application as well as workers):

```js
var workerpool = require('workerpool');
```

To load workerpool in the browser:

```html
<script src="workerpool.js"></script>
```

To load workerpool in a web worker in the browser:

```js
importScripts('workerpool.js');
```


## Use

### Offload functions dynamically

In the following example there is a function `add`, which is offloaded dynamically to a worker to be executed for a given set of arguments.

```js
var workerpool = require('workerpool');
var pool = workerpool.pool();

function add(a, b) {
  return a + b;
}

pool.run(add, [3, 4])
    .then(function (result) {
      console.log('result', result); // outputs 7

      pool.clear(); // clear all workers when done
    });
```

Note that both function and arguments must be static and stringifiable, as they need to be send to the worker in a serialized form. In case of large functions or function arguments, the overhead of sending the data to the worker can be significant.


### Dedicated workers

A dedicated worker can be created and used via a worker pool. The worker is written in a separate JavaScript file

**myWorker.js**
```js
var workerpool = require('workerpool');

// a deliberately inefficient implementation of the fibonacci sequence
function fibonacci(n) {
  if (n < 2) return n;
  return fibonacci(n - 2) + fibonacci(n - 1);
}

// create a worker and register public functions
workerpool.worker({
  fibonacci: fibonacci
});
```

This worker can be used by a worker pool:

**myApp.js**
```js
var workerpool = require('workerpool');

// create a worker pool using an external worker script
var pool = workerpool.pool(__dirname + '/myWorker.js');

// run functions on the worker via exec
pool.exec('fibonacci', [10])
    .then(function (result) {
      console.log('Result: ' + result); // outputs 55

      pool.clear(); // clear all workers when done
    });
```


## Examples

Examples are available in the examples directory:

[https://github.com/josdejong/workerpool/tree/master/examples](https://github.com/josdejong/workerpool/tree/master/examples)


## API

The API of workerpool consists of two parts: a function `workerpool.pool` to create a worker pool, and a function `workerpool.worker` to create a worker.

### workerpool

A workerpool can be created using the function `workerpool.pool`:

`workerpool.pool([script: string] [, options: Object]) : Pool`

When a `script` argument is provided, the provided script will be started as a dedicated worker.
When no `script` argument is provided, a default worker is started which can be used to offload functions dynamically via `Pool.run`.

The following options are available:
- `maxWorkers: number`. The default number of workers on node.js is the number of CPU's minus one. The default number of workers in a browser environment is 3.

A worker pool contains the following functions:

- `Pool.exec(method: string, params: Array | null) : Promise.<*, Error>`<br>
  Execute a function on a worker with given arguments.
- `Pool.run(fn: Function, args: Array | null) : Promise.<*, Error>`<br>
  Offload a function dynamically to a worker, execute it with given arguments, and return the result. The provided function `fn` is stringified and send to the worker, therefore this function must be static.
- `Pool.proxy() : Promise.<Object, Error>`<br>
  Create a proxy for the worker pool. The proxy contains a proxy for all methods available on the worker. All methods return promises resolving the methods result.
- `Pool.clear([force: boolean])`<br>
  Clear all workers from the pool. If parameter `force` is false (default), workers will finish the tasks they are working on before terminating themselves. When `force` is true, all workers are terminated immediately without finishing running tasks.

Example usage:

```js
var workerpool = require('workerpool');

function add(a, b) {
  return a + b;
}

var pool1 = workerpool.pool();

// offload a function to a worker
pool1.run(add, [2, 4])
    .then(function (result) {
      console.log(result); // will output 6
    });

// create a dedicated worker
var pool2 = workerpool.pool(__dirname + '/myWorker.js');

// supposed myWorker.js contains a function 'fibonacci'
pool2.exec('fibonacci', [10])
    .then(function (result) {
      console.log(result); // will output 55
    });

// create a proxy to myWorker.js
pool2.proxy()
    .then(function (myWorker) {
      myWorker.fibonacci(10)
          .then(function (result) {
            console.log(result); // will output 55
          });
    });

// create a pool with a specified maximum number of workers
var pool3 = workerpool.pool({maxWorkers: 7});
```


### worker

A worker is constructed as:

`workerpool.worker([methods: Object.<String, Function>])`

Argument `methods` is optional can can be an object with functions available in the worker. Registered functions will be available via the worker pool.

Example usage:

```js
// file myWorker.js
var workerpool = require('workerpool');

function add(a, b) {
  return a + b;
}

function multiply(a, b) {
  return a * b;
}

// create a worker and register functions
workerpool.worker({
  add: add,
  multiply: multiply
});
```


## Build

First clone the project from github:

    git clone git://github.com/josdejong/workerpool.git
    cd workerpool

Install the project dependencies:

    npm install

Then, the project can be build by executing the build script via npm:

    npm run build

This will build the library workerpool.js and workerpool.min.js from the source
files and put them in the folder dist.


## Test

To execute tests for the library, install the project dependencies once:

    npm install

Then, the tests can be executed:

    npm test

To test code coverage of the tests:

    npm run coverage

To see the coverage results, open the generated report in your browser:

    ./coverage/lcov-report/index.html


## License

Copyright (C) 2014 Jos de Jong <wjosdejong@gmail.com>

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

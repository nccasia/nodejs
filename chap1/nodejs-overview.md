Nodejs itself is multithreaded. The lower levels of Node.js are written in C++ (libuv which handles operating system abstractions and I/O), V8 (the javascript engine)

```javascript
const fs = require('fs');

fs.readFile('/etc/passwd', (1)
  (err, data) => { (4)
    if (err) throw err;
    console.log(data);
});

setImmediate( (2)
  () => { (3)
    console.log('This runs while file is being read');
});

```
(1) Node.js reads /etc/passwd. It's scheduled by libuv.\
(2) Node.js runs a callback in a new stack. It's scheduled by V8.\
(3) Once the previous stack ends, a new stack is created and print a message\
(4) Once the file is done reading, libuv passes the result to the V8 event loop.\

_Note: libuv thread pool size defaults to four, has a max of 1,024 can ovveridden by setting the UV_THREADPOOL_SIZE=\<threads>_ 

Node.js maintains a list of asynchronous tasks that still need to be completed. This list is used to keep the process running. When a stack completes and the event loops looks for more work to do. if there are no more operations left to keep the process alive, it will exit.\

Once an asynchronous task has been created, this is enough to keep a process alive, like in this example:

```javascript
setInterval(() => {
  console.log('Process will run forever');
}, 1000);
```
Another example for keep the process alive when a HTTP server is created, it also keeps the process running forever.
There is a commom pattern in the Node.js APIs where such objects can be configured to no longer keep the process alive. For example, if a listening HTTP server port is closed, then the process may choose to end. Additionally, many of these objects have a pair methods attached to them, ```.unref()``` and ```.ref()```, used to tell the object to no longer keep the process alive or not.

```javascript
const t1 = setTimeout(() => {}, 1000000); (1)
const t2 = setTimeout(() => {}, 2000000); (2)
// ....
t1.unref(); (3)
// ....
clearTimeout(t2) (4);
```
(1) There is now one asynchronous operation keeping Node.js alive. The process should end in 1000 seconds.
(2) There are now two such operations. The process should now end in 2,000 seconds
(3) The t1 timer has been unreferenced. Its callback can still run in 1,000 seconds, but it won't keep the process alive
(4) The t2 timer has been cleared and will never run. A side effect of this is that it no longer keeps the process alive. With no remaining asynchronous operations keeping the process alive, the next iteration of the event loop ends the process.

### The Node.js Event Loop
Both the javascript that runs in your browser and the JavaScript that runs in Node.js come with an implementation of an event loop.\
As the name implies, the event loop runs in a loop, manages a queue of events that are used to trigger callbacks and move the application along.\

**Event Loop phases**
The Event loop has serveral different phases to it. Some of these phases don't deal with application code directly;

Each one of these phases maintains a queue of callbacks that are to be executed. Callbacks are destined for different phases based on how they are used by the application.

- **Pool**: the pool phase executes I/O-related callbacks. This is phase that application code is most  likely to execute in. When your main application code starts running, it runs in this phase.
- **Check**: In this phase, callbacks that are triggered via ```setImmediate()``` are executed
- **Close**: This phase executes callbacks that are triggered via EventEmitter close event. For example, when a __net.Server TCP__ server close, it emits a close event that runs a callback in this phase.
- **Timer**: Callbacks scheduled using ```setTimeout()``` and ```setInterval()``` are executed in this phase.
- **Pending**: Special system events are run in this phase, like when a __net.Server TCP__ socket throws an ECONNREFUSED error.

Pool -> Check -> Close -> Timers -> Pending

To make things a litle more complicated, there are also two special microtask queues that can have callbacks added to them whilw a phase is running. The first microtask queue handles callbacks that have been registered using ```process.nextTick()```. The second microtask queue handles promises that reject or resolve. Callbacks in the microtask queues take priority over callbacks in the phase's normal queue, and callbacks in the next tick microtask queue run before callbacks in the promise microtask queue.

When the application starts running, the event loop is also started and the phases are handled one at a time. Node.js adds callbacks to different queues as appropriate while the application runs. When the event loop gets to a phase, it will run all the callbacks in that phase'queue. Once all the callbacks in a given phase are exhausted, the event loop then moves on to the next phase. If the application runs out of things to do but is waiting for I/O operations to complete, it'll hang out in the pool phase.\

```javascript
const fs = require('fs');

setImmediate(() => console.log(1));
Promise.resolve().then(() => console.log(2));
process.nextTick(() => console.log(3));
fs.readFile(__filename, () => {
  console.log(4);
  setTimeout(() => console.log(5));
  setImmediate(() => console.log(6));
  process.nextTick(() => console.log(7));
});
console.log(8);
```
The script starts off executing line by line in the poll phase. Next, the ```setImmediate()``` call is run, which adds the callback printing 1 to the check queue. Then, the promise resolves, adding callback 2 to the promise microtask queue. ```process.nextTick()``` runs next, adding callback 3 to the next tick microtask queue. Once that's done the ```fs.readFile()``` call tells the Node.js APIs to start reading a file, placing its callback in the poll queue once it's ready. Finally, log number 8 is called directly and is printed to the screen.\

That's it for current stack. Now the two microtask queues are consulted. The next tick microtask queue is always checked first, and callback 3 is called. Here callback 2 is executed. That finishes the two microtask queues, and the current poll phase is complete.

Now the event loop enters the check phase. This phase has callback 1 in it, which is then executed. Both the microtask queues are empty at this point, so the chekc phase ends. The close phase is checked next but is empty, the loop continues. The same happens with the timers phase and the pending phase, and the event loop continues back around to the poll phase.

Once it's back in the poll phase, the application doesn't have much else going on, so it basically waits until the file has finished being read. Once that happens the ```fs.readFile()``` callback is run.\
The number 4 is immediately printed since it's the first line in the callback. Next, the ```setTimeout()``` call is made and callback 5 is added to the timer queue. The ```setImmediate()``` call happen next, adding callback 6 to the check queue. Finally, the ```process.nextTick()``` call is made, adding callback 7 to the next tick microtask queue. The poll queue is now finished, and the microtask queues are again consulted. Callback 7 runs from the next tick queue, the promise queue is consulted and found empty, and the poll phase ends.\
Again, the event loop switches to the check phase where callback 6 is encountered, the microtask queues are determined to be empty and the phase ends. The close phase is checked again and found empty. Finally the timers phase is consulted wherein callback 5. Once that's done, the application doesn't have nay more work to do and it exist.\
So the result is: ``` 8, 3, 2, 1, 4, 7, 6, 5```

### Event Loop Tips

**Don't starve the event loop**. Running too much code in a single stack will stall the event loop and prevent other callbacks from firing. One way to fix this is to break CPU-heavy operations up across multiple stacks. For example, if you need to process 1,000 data records, you might consider breaking it up into 10 batches of 100 records, using ```setImmediate()``` at the end of each batch to continue processing the next batch. Depeding on the situation, it might make more sense to offload processing to a child process.\
You should never breakup such work using ```process.nextTick()```. Doing so will lead to a microtask queue that never empties - your application will be trapped in the same phase forever.

```javascript
const nt_recursive = () => process.nextTick(nt_recursive);
nt_recursive(); // setInterval will never run

const si_recursive = () => setImmediate(si_recursive);
si_recursive(); // setInterval will run

setInterval(() => console.log('hi'), 10);
``` 
\
In this example, the ```setInterval()``` represents some asynchorous work that the application performs, such as responding to incoming HTTP requests. Once the nt_recursive() function is run, the application ends up with a microtask queue that never empties and the asynchorous work never gets processed. but the alternative version ```si_recursive()``` does not have the same side effect. Making ```setImmediate()``` calls within a check phase adds callback to the next event loop interation's check phase queue,  not current phase's queue. \
\
Don't instroduce Zalgo. When exposing a method that that takes a callback, that callback should always be run asynchronous. For example, it's far too easy to write something like this:
```javascript
// Antipattern
function foo(count, callback) {
  if (count <= 0) {
    return callback(new TypeError('count > 0'));
  }
  myAsyncOperation(count, callback);
}
```
The callback is sometimes called synchronously, like when count is set to zero, and sometimes asynchronously, like when count is set to 1. Instead, ensure the callback is executed in a new stack, like in this example:
```javascript
function foo(count, callback) {
  if (count <= 0) {
    return process.nextTick(() => callback(new TypeError('count > 0')));
  }
  myAsyncOperation(count, callback);
}
```
\
in this case, either using ```setImmediate()``` or ```process.nextTick()``` is okay.

With this reworked example, the callback is always run asynchronously. Ensuring the callback is run consistently is important because of the following situation:
```javascript
let bar = false;
foo(3, () => {
  assert(bar);
});
bar = true;
```
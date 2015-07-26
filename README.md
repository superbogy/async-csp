# async-csp
Communicating sequential processes, or asynchronous buffered data pipes, designed to be used with async/await.

```js
import Channel, { timeout } from 'async-csp';

async function puts(channel) {
    for (let i = 0; i < 5; i++) {
        await timeout(1000);
        await channel.put(i);
    }
}

async function takes(channel) {
    for (let i = 0; i < 5; i++) {
        let val = await channel.take();
        console.log('val:', val);
    }
}

// create a new csp channel
let channel = new Channel();

// places a value onto the channel once every second
puts(channel);

// logs values from the channel as soon as they are available
takes(channel);

// console logs, once a second
//=> 0
//=> 1
//=> 2
//=> 3
//=> 4
```

## Installation

```
npm install async-csp
```

## Default Task

* Install node.js
* Clone the async-csp project
* Run `npm install`
* Run `gulp`
    * Executes tests
    * Cleans dist
    * Lints source
    * Builds source
    * Watches source and tests

## Usage

*Note: All of the examples below are assumed to be executed from an `async` context, so `await` is available. To read more about these methods, see [this proposal](https://github.com/lukehoban/ecmascript-asyncawait) for async/await in ES7.*

### Data Flow

A `Channel` is a buffered pipe which makes use of `Promises` to handle the flow of data.

To place a value on a `Channel`, you may use the method `Channel#put()`, and to take a value off of the channel, you may use the method `Channel#take()`.

```js
import Channel from 'async-csp';

let ch = new Channel();
await ch.put(1);
await ch.put(2);
await ch.put(3);
console.log(await ch.take()); //=> 1
console.log(await ch.take()); //=> 2
console.log(await ch.take()); //=> 3
```

### Buffering

When a `Channel` is full, `Channel#put()` will not resolve until space on the buffer becomes available. When a `Channel` is empty, `Channel#take()` will not resolve until a value is available.

To create a `Channel` with a buffer size, pass in a `Number` as the first argument to the constructor.

```js
import Channel, { timeout } from 'async-csp';

async function puts(ch) {
    await ch.put(1); //=> this will resolve immediately
    await ch.put(2); //=> this will also resolve immediately
    await ch.put(3); //=> this will block until another process takes a value from the Channel
}

async function takes(ch) {
    console.log(await ch.take()); //=> this will cause the blocked ch.put(3) to resolve, and will return 1
    console.log(await ch.take()); //=> this will resolve immediate, and return 2
    console.log(await ch.take()); //=> this will resolve immedlately, and return 3
    console.log(await ch.take()); //=> this will not resolve until another value is available on the channel
}

let channel = new Channel(2);
puts(channel);
await timeout(100); // this is a helper method which will resolve after the given milliseconds
takes(channel);
```

### Transforming

When constructing a `Channel`, you can pass in a callback to transform values as they are placed onto the buffer.

```js
let ch = new Channel(x => x * 2);

await ch.put(1);
await ch.put(2);
await ch.put(3);

console.log(await ch.take()); //=> 2
console.log(await ch.take()); //=> 4
console.log(await ch.take()); //=> 6
```

If values should be dropped from the `Channel`, simply return `undefined` from the transform callback.

```js
let ch = new Channel(x => {
    if (x > 2)
        return x;
});

await ch.put(1);
await ch.put(1);
await ch.put(3);
await ch.put(4);

console.log(await ch.take()); //=> 3
console.log(await ch.take()); //=> 4
```

If a transform should take a single value and expand it into multiple values, then the `push` parameter can be used with the transform callback.

Note that when using this method, any values must be sent through `push`. Any value returned from the transform callback will be ignored when more than one parameter is defined.

```js
let ch = new Channel((x, push) => {
    push(x);
    push(x + 1);
});

await ch.put(1);
await ch.put(3);

console.log(await ch.take()); //=> 1
console.log(await ch.take()); //=> 2
console.log(await ch.take()); //=> 3
console.log(await ch.take()); //=> 4
```

If the transform needs to work asynchronously, there are two ways to accomplish this.

The first is to use a third parameter with the transform callback, and execute it to signify that the transform has finished.

```js
let ch = new Channel((x, push, done) => {
    push(x);
    setTimeout(() => {
        push(x + 1);
        done();
    }, 100);
});

await ch.put(1);
await ch.put(3);

console.log(await ch.take()); //=> 1
console.log(await ch.take()); //=> 2
console.log(await ch.take()); //=> 3
console.log(await ch.take()); //=> 4
```

The second is to declare the transform as `async` or to return a `Promise`.

```js
let ch = new Channel(async(x, push) => {
    push(x);
    await timeout(100);
    push(x + 1);
});

await ch.put(1);
await ch.put(3);

console.log(await ch.take()); //=> 1
console.log(await ch.take()); //=> 2
console.log(await ch.take()); //=> 3
console.log(await ch.take()); //=> 4
```

### Channel#pipe

Similarly to `Streams`, `Channels` can be piped from one to another.

```js
let ch1 = new Channel();
let ch2 = new Channel();

ch1.pipe(ch2);

await ch1.put(1);
await ch1.put(2);
await ch1.put(3);

console.log(await ch2.take()); //=>  1
console.log(await ch2.take()); //=>  2
console.log(await ch2.take()); //=>  3
```

`Channels` can be piped to multiple destinations. In this case, all downstream `Channels` will receive every value from upstream.

```js
let ch1 = new Channel();
let ch2 = new Channel();
let ch3 = new Channel();

ch1.pipe(ch2, ch3);

await ch1.put(1);
await ch1.put(2);
await ch1.put(3);

console.log(await ch2.take()); //=>  1
console.log(await ch2.take()); //=>  2
console.log(await ch2.take()); //=>  3

console.log(await ch3.take()); //=>  1
console.log(await ch3.take()); //=>  2
console.log(await ch3.take()); //=>  3
```

### Channel#merge

Merging is a form of automated piping from multiple `Channels` into a single, new `Channel`.

```js
let ch1 = new Channel();
let ch2 = new Channel();
let ch3 = ch1.merge(ch2); // or Channel.merge(ch1, ch2)

await ch1.put(1);
await ch2.put(2);

console.log(await ch3.take()); //=> 1
console.log(await ch3.take()); //=> 2
```

### Channel#close

`Channels` have 3 states: open, closed, and ended. An open `Channel` can be written to, a closed `Channel` can no longer be written to, and an ended `Channel` is both closed and empty.

To signify that a `Channel` will no longer have data written, execute `close`. Data can still be taken from the channel, even when closed, but no data can be added after that point.

```js
let ch1 = new Channel();

await ch1.put(1);
await ch1.put(2);

ch1.close();

await ch1.put(3); // resolves immediately with value of Channel.DONE

console.log(await ch1.take()); //=> 1
console.log(await ch1.take()); //=> 2
console.log(await ch1.take()); //=> Channel.DONE
```

If `Channels` are piped together, and you want the entire pipeline to close, simply pass `true` as an argument to `Channel#close`.

```js
let ch1 = new Channel();
let ch2 = new Channel();
ch1.pipe(ch2);

await ch1.put(1);
await ch1.put(2);

ch1.close(true);

console.log(await ch2.take()); //=> 1
console.log(await ch2.take()); //=> 2
console.log(await ch2.take()); //=> Channel.DONE
```

### Channel#done

In order to wait for a channel to be ended (closed and empty), await the resolution of `done`.

```js
let ch = new Channel();

await ch.put(1);
await ch.put(2);
ch.close();

(async() => {
    await timeout(1000);
    await ch.take();
    await timeout(1000);
    await ch.take();
})();

await ch.done(); // will not resolve until the async IIFE takes both values from the channel
```

### Channel#consume

If you would like to execute a callback as soon as values become available on the `Channel`, you may add a callback using `Channel#consume`.

```js
let ch = new Channel();
ch.consume(x => {
    console.log(x);
});

await ch.put(1);
await ch.put(2);
await ch.put(3);
await ch.put(4);

// console logs
//=> 1
//=> 2
//=> 3
//=> 4
```

`Channel#consume` can also be handled asynchronously.

```js
let ch = new Channel();
ch.consume(async x => {
    await timeout(1000);
    console.log(x);
});

await ch.put(1);
await ch.put(2);
await ch.put(3);
await ch.put(4);

// console logs, once a second
//=> 1
//=> 2
//=> 3
//=> 4
```

### Channel#produce

Similar to `Channel#consume`, `Channel#produce` will put returned values onto the `Channel` as soon as space becomes available.

```js
let ch = new Channel();
let counter = 0;
ch.produce(() => ++counter);

console.log(await ch.take()); //=> 1
console.log(await ch.take()); //=> 2
console.log(await ch.take()); //=> 3
console.log(await ch.take()); //=> 4
```

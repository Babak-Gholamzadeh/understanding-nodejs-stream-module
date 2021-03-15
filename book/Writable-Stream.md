# Writable Stream

As we talked already about the stream module, we understood that this module has two main parts which are `readable` and `writable` streams, and two other parts that are created from the combination of these two main parts which are called `duplex` and `transform`.

Thus we need to first understand the main parts and also for warming up, we are going to start with the `writable` stream.

The main responsibility of the `writable` stream – as you can see in the image below – is to get chunks of data at a time and pass them to a destination – the destination could be anything i.e. a file on the disk.

<p align="center">
  <img alt="writable stream" src="/book/assets/figure-09_writable-stream.png" />
</p>

The `writable` stream module to perform its duties follows a simple workflow and is such that when a chunk of data is pushed to this module, it passes that to the destination.

So let’s build a `class` based structure (you are free to go with `function` or any approach that you feel it’s better for you) for our module and start to implement its workflow.

```javascript
// =============================
//   Inside writable.js module
// =============================
class Writable {
  constructor() {}
}
```

In order to implement that workflow, we need a method that we are going to name it `write` method which is in charge of getting a chunk of data from the outside and pass it to the destination.

Also for passing the chunk to the destination we will go with a callback pattern but in a different way.

We are not going to take the callback as a parameter of the `write` method. Because pushing data to the stream module and moving it to a destination is going to be the two sides of a wall and none of them are supposed to know about each other.

So we plan to allow the destination to define a method in the stream object with the name of `_write` as a convention that the stream be able to send the data through it.

```javascript
write(chunk) {
  if (typeof this._write !== 'function')
    throw new Error('_write() must to be implemented!');
  this._write(chunk);
}
```

So far so good.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/fb8ee7cc7d6729d0a7a9a57039f1e36a9a66cc4f)

The other thing that the `writable` stream should care about it is when we push data very fast to the destination and it cannot handle the pressure. Because what we have here is just a straight data transfer.

So we need a strategy to control this pressure.

To handle the pressure and preventing from data loss simultaneously, we have to store the data in memory.

Storing the data is simple and we just need to use an array with the name of `buffered` and push the chunks when needed.

But how do we figure out that the destination is not ready for receiving another chunk?

We can handle this situation by sending a callback from the stream module to the destination via the `_write` method that is implemented by the destination itself and assume that whenever this callback is called, the destination has consumed the previous chunk already and it is ready to receive another one.

Also, we need to know about the state of the destination – whether it is ready – at some points. So we store this state as a property of `this` object with the name of `writing` and we set its value to `true` when we pass a chunk to the destination and the callback is passed to the `_write` method is going to turn it back to `false` – its default value is `false` which means that the first time, the destination is open to receive data.

```javascript
constructor() {
  this.buffered = [];
  this.writing = false;
}
```

Because the states of the stream that we need to track are going to grow over time, it’s better to build a separate class for them from now on to avoid further messing.

```javascript
class WritableState {
  constructor() {
    this.buffered = [];
    this.writing = false;
  }
}
```

Now we just need to create an instance of this in the `constructor` of our `Writable` class.

```javascript
constructor() {
  this._writableState = new WritableState();
}
```

After doing that we ought to update the `write` method to cover our strategy.

```javascript
write(chunk) {
  const state = this._writableState;

  if (typeof this._write !== 'function')
    throw new Error('_write() must to be implemented!');
  
  if (state.writing) {
    state.buffered.push(chunk);
  } else {
    state.writing = true;
    this._write(chunk, this._onwrite.bind(this));
  }
}
```

> *Note: The first line of the method (that you might see a lot in the future methods) is just there to make the name shorter and nothing more.*

The `_onwrite` method is supposed to be the callback to update the `writing` state.

> *Note: We have to bind the `this` object to it because it needs to access the `this` object when it’s called.*

```javascript
_onwrite() {
  const state = this._writableState;
  state.writing = false;
}
```

That’s it.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/2953fa04dd565f1a2178c8c002b1555357377fb2)

But wait, what is supposed to happen to the buffered data? Because if we walk over the workflow we would notice that the buffered data will remain even after the `writing` state back to `false` again and we just pass a new chunk to the `_write` method. So we need to find a solution to this issue.

We should handle that in the `_onwrite` method because over there, is the only place that we would be noticed that the destination is ready to receive more.

So let’s upgrade the `_onwrite` method to flush the buffer before going for the next chunk.

In order to clear the buffer, we can do some recursive processes indirectly.

```javascript
_onwrite() {
  const state = this._writableState;
  state.writing = false;
  if (state.buffered.length) {
    this._clearBuffer();
  }
}
```

Here we check if there is any data in the buffer, just call the `_clearBuffer` method.

In the `_clearBuffer` method, we treat the buffer as a queue and fetch its first chunk by using the `shift` method.

```javascript
_clearBuffer() {
  const state = this._writableState;
  const chunk = state.buffered.shift();
  this._doWrite(chunk);
}
```

After fetching the first chunk in the buffer list, we call the `_doWrite` method and let it continue the process of sending chunks to the destination by calling the `_write` method again.

```javascript
_doWrite(chunk) {
  const state = this._writableState;
  state.writing = true;
  this._write(chunk, this._onwrite.bind(this));
}
```

As you realized, this recursive process of reading chunks from the buffer will continue until the buffer become empty.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/32e080f047f3dd7e5e5bb5e1a4c002c19b562a41)

> *Note: If you think having this number of helper methods that contains only one or two lines of code is pointless, don’t worry because they will grow up over time and be called in several places. And the other reason is that we intend to follow the same structure as the built-in stream module.*

So far our implementation can handle the primary operations of a `writable` stream. But there are still some issues that we ought to worry about them and one of them is the size of the buffer list.

One way to take care of the buffer size is to get a callback as a parameter of the `write` method and run it inside of the `_onwrite` method. With this approach, the outside world (who called the `write` method) will be noticed that the chunk that you sent recently is consumed and you can go for the next chunk.

To implement this approach, first, we need to define a property in the `WritableState` class and name it `writecb` - with the default value of `null.

```javascript
this.writecb = null;
```

Next, add another parameter to the `write` method next to the `chunk` parameter and put the callback parameter in the `writecb` when we want to call the `_write` method directly.

```javascript
write(chunk, callback) {
  const state = this._writableState;

  if (typeof this._write !== 'function')
    throw new Error('_write() must to be implemented!');
  
  if (state.writing) {
    state.buffered.push(chunk);
  } else {
    state.writing = true;
    state.writecb = callback;
    this._write(chunk, this._onwrite.bind(this));
  }
}
```

And of course, we need to change the `_onwrite` method to call the callback.

```javascript
_onwrite() {
  const state = this._writableState;

  const cb = state.writecb;

  state.writing = false;
  state.writecb = null;

  if (state.buffered.length) {
    this._clearBuffer();
  }

  if (typeof cb === 'function') {
    cb();
  }
}
```

Also, we shouldn’t forget the path of the running method to read the buffered data.

First, we need to update the way we buffer data, so update the `write` method again.

```javascript
write(chunk, callback) {
  const state = this._writableState;

  if (typeof this._write !== 'function')
    throw new Error('_write() must to be implemented!');
  
  if (state.writing) {
    state.buffered.push({ chunk, callback });
  } else {
    state.writing = true;
    state.writecb = callback;
    this._write(chunk, this._onwrite.bind(this));
  }
}
```

Plus, update the `_clearBuffer` and `_doWrite` method as well.

```javascript
_clearBuffer() {
  const state = this._writableState;
  const { chunk, callback } = state.buffered.shift();
  this._doWrite(chunk, callback);
}

_doWrite(chunk, callback) {
  const state = this._writableState;
  state.writing = true;
  state.writecb = callback;
  this._write(chunk, this._onwrite.bind(this));
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/65f8817ce302c23c317ba495a04f17e759521097)

This approach is good but not enough. Because it doesn’t give us enough power to control pushing data to a `writable` stream and using that for this purpose is kind of difficult.

There is also another approach for taking care of the buffer size is to define a threshold and warn the outside world that the buffer size exceeds the threshold.

We are going to define that threshold as a property with the name of the `highWaterMark` and the default value of `16K`. Also, give the user an option to define its value on creating a `writable` instance.

For the purpose of having a threshold, we should always know about the current length of the buffer size to be able to compare it to the threshold, which we can add another property named `length` to could track the current size of the buffer.

First, update the `constructor` of the `Writable` class to get the option.

```javascript
constructor(options = {}) {
  this._writableState = new WritableState(options);
}
```

Then update the `constructor` of the `WritableState` class to set the option and its default value, plus adding the `length` property.

```javascript
constructor({ highWaterMark = 16 * 1024 }) {
  this.highWaterMark = highWaterMark;
  this.length = 0;
  this.buffered = [];
  this.writing = false;
  this.writecb = null;
}
```

And to could warn the user that the current size of the buffer exceeds the `highWaterMark`, we need to update the `write` method and return a proper value based on the circumstance.

```javascript
 write(chunk, callback) {
  const state = this._writableState;

  if (typeof this._write !== 'function')
    throw new Error('_write() must to be implemented!');
  
  const len = chunk.length;
  state.length += len;
  const ret = state.length < state.highWaterMark;

  if (state.writing) {
    state.buffered.push({ chunk, callback });
  } else {
    state.writing = true;
    state.writecb = callback;
    this._write(chunk, this._onwrite.bind(this));
  }

  return ret;
}
```

Now when the current size of the buffer exceeds the `highWaterMark`, the `write` method would return `false`. But it will never return to `true` from that point because we need to subtract the length of the consumed chunk from the `length` property in the `_onwrite` method.

Due to access the chunk length in the `_onwrite` method, first, we must define another property named `writelen`. To update the `constructor` of the `WritableState` class again.

```javascript
constructor({ highWaterMark = 16 * 1024 }) {
  this.highWaterMark = highWaterMark;
  this.length = 0;
  this.buffered = [];
  this.writing = false;
  this.writecb = null;
  this.writelen = 0;
}
```

Next only update this part of the code of the `write` method:

```javascript
if (state.writing) {
  state.buffered.push({ chunk, callback });
} else {
  state.writing = true;
  state.writecb = callback;
  state.writelen = len;
  this._write(chunk, this._onwrite.bind(this));
}
```

And Also this part of the code of the `_doWrite` method:

```javascript
_doWrite(chunk, callback) {
  const state = this._writableState;
  state.writing = true;
  state.writecb = callback;
  state.writelen = chunk.length;
  this._write(chunk, this._onwrite.bind(this));
}
```

Eventually, update the `_onwrite` method.

```javascript
_onwrite() {
  const state = this._writableState;

  const cb = state.writecb;

  state.writing = false;
  state.writecb = null;
  state.length -= state.writelen;
  state.writelen = 0;

  if (state.buffered.length) {
    this._clearBuffer();
  }

  if (typeof cb === 'function') {
    cb();
  }
}
```

Okay, now the returned value of the `write` method works as expected. But imagine how the user is going to use this feature?

The user calls this method and if it returns a `false` value, it means that the buffer size exceeds the `highWaterMark`. But how is the user supposed to be noticed that it drains and now it’s safe to call the `write` method again?

So the feature is not complete yet and we need a way to notify the user.

This is where the event system comes into play. Simply we just emit an event by the name of `drain` and the user will be noticed by listening to this event.

To use the event system, our `Writable` class needs to extend from an event emitter class – which can be our implemented events module or the built-in one. So first `require` an events module and then let the `Writable` class extends from it.

```javascript
const EventEmitter = require('./events');
```

> *Note: Don’t forget to call the `super` method inside of the `constructor` of the `Writable` class to construct the `EventEmitter` class.*

```javascript
class Writable extends EventEmitter {
  constructor(options = {}) {
    super();
    this._writableState = new WritableState(options);
  }
}
```

That’s it. Now we can use event emitter methods to emit an event or listen to one. But to understand when we need to emit a `drain` event, first, we should know when need to drain the buffer.

For achieving this purpose, we define a flag named `needDrain` with a default value of `false` and whenever the `write` method is going to return `false` value - which means the buffer size exceeds the `highWaterMark` - we need to drain, then just switch this flag to `true`.

```javascript
write(chunk, callback) {
  // there are some codes before this part

  const len = chunk.length;
  state.length += len;
  const ret = state.length < state.highWaterMark;
  if (!ret)
    state.needDrain = true;

  if (state.writing) {
    state.buffered.push({ chunk, callback });

  // there are some codes after this part
}
```

And inside of the `_onwrite` method, we check if we have already needed to drain and currently the buffer is empty - which means it has already drained - then just emit a `drain` event.

```javascript
_onwrite() {
  // there are some codes before this part

  if (state.buffered.length) {
    this._clearBuffer();
  }
  if (state.needDrain && state.length === 0) {
    state.needDrain = false;
    this.emit('drain');
  }
  if (typeof cb === 'function') {
    cb();
  }
}
```

At this point, the feature is complete and whenever the `write` method returns a `false` value, the user should stop writing more and just listen to the `drain` event, then resume writing again once it drains.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/7cde2145556cd6ea1b4a6dee8c30fd75e42125b8)

As you might have noticed so far, this approach does not strictly prevent the exceeding of the buffer size too. This is how the stream works in Node.js and it only informs the user of what is happening. Ultimately, it is up to the user to take an action.

The `writable` stream also has some other methods to control the process of buffering data, which are `cork` and `uncork` methods.

As mentioned in the Node.js document, when the `cork` method is called, it forces all written data to be buffered. And by calling the `uncork` method the buffered data will be flushed.

But before starting to implement these methods, be aware of how many times you call the `cork` method, you have to call the `uncork` method at least the same number of times to be able to flush the buffer.

It means that we need to set another property named `corked`, and increase its value inside the `cork` method, and decrease it inside the `uncork` method until it reaches zero and then the buffer is flushed.

Thus, first, define the required property.

```javascript
this.corked = 0;
```

Next, define the `cork` method to increase it.

```javascript
cork() {
  this._writableState.corked++;
}
```

Also define the `uncork` method to decrease it and then call the `_clearBuffer` method to flush the buffer.

```javascript
uncork() {
  const state = this._writableState;
  if (state.corked) {
    state.corked--;
    if (!state.writing)
      this._clearBuffer();
  }
}
```

Now we have the methods so we need to make sure that if the stream is corked, just buffer the chunks of data right away.

So we should update the condition of buffering data in the `write` method:

```javascript
write(chunk, callback) {
  // there are some codes before this part

  if (state.writing || state.corked) {
    state.buffered.push({ chunk, callback });
  } else {
    state.writing = true;
    state.writecb = callback;
    state.writelen = len;
    this._write(chunk, this._onwrite.bind(this));
  }

  return ret;
}
```

And also add an `if` statement into the `_clearBuffer` method to ensure that the buffer would never be flushed if the stream is corked.

```javascript
_clearBuffer() {
  const state = this._writableState;
  if (state.corked)
    return;

  const { chunk, callback } = state.buffered.shift();
  this._doWrite(chunk, callback);
}
```

We are done with the `cork` and `uncork` methods and their effects.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/137845046597d5dd863e2c747852babdfcb85609)

The next part that we are going to talk, is about the type of data chunks.

Currently, the built-in stream module accepts three types of data include `string`, `Buffer`, and `Uint8Array`. There exists another flag named `objectMode` which is `false` by default and if you change it to `true` then your stream object can work with any JavaScript data type except `null` (the `null` value is defined for a special purpose).

> *Note: Also keep in my mind that when the value of `objectMode` is `true`, then the length of each chunk of data, regardless of its type, is always considered equal to **one**.*

Here we plan to work with two different types of `objectMode` and only implement `string` and `Buffer` types while the `objectMode` is equal to `false` - we just ignore the `Uint8Array` type for simplicity reason.

Also stream module provides another option for working with the `string` type that you can change the encoding of chunks of string – the default encoding is `utf8`.

First of all, before doing anything we need to define a bunch of properties. Thus update the `constructor` of the `WritableState` class to add the `objectMode` (the default value is `false`), `defaultEncoding` (the default value is `utf8`), and `decodeStrings` (the default value is `false` which means that we don’t need to change the encoding of string value unless this becomes `true`). Also, we should set them as an option that user can define their initial values on creating a `writable` instance.

```javascript
constructor({
  objectMode = false,
  decodeStrings = false,
  defaultEncoding = 'utf8',
  highWaterMark = 16 * 1024,
}) {
  this.objectMode = objectMode;
  this.decodeStrings = decodeStrings;
  this.defaultEncoding = defaultEncoding;
  this.highWaterMark = highWaterMark;
  this.length = 0;
  this.buffered = [];
  this.writing = false;
  this.writecb = null;
  this.writelen = 0;
  this.needDrain = false;
  this.corked = 0;
}
```

After defining the required properties, we ought to add some more stuff into our `write` method.

First, we define a parameter named `encoding` to get the encoding type, but it’s optional. So place it as the second parameter (between `chunk` and `callback`) like this:

```javascript
write(chunk, encoding, callback) {}
```

Then because we want to make sure that the `write` method can be called in different ways:

```javascript
write(chunk);                       // 1
write(chunk, encoding);             // 2
write(chunk, callback);             // 3
write(chunk, encoding, callback);   // 4
```

Hence we should take care of the value of the last two parameters.

First, we check whether the type of `encoding` value is `function`, if so it means we didn’t get an encoding type from the user and we must use the value of `defaultEncoding`. Otherwise, we need to check whether the `encoding` parameter has a truthy value, if so then check if the value neither is equal to `buffer` nor a valid encoding type, then throw an error. Also if it doesn’t have a truthy value, just put the `defaultEncoding` value to it.

If the explanation confused you, don’t worry. Look at the code, it’s much simpler to understand.

```javascript
if (typeof encoding === 'function') {
  callback = encoding;
  encoding = state.defaultEncoding;
} else {
  if (!encoding)
    encoding = state.defaultEncoding;
  else if (encoding !== 'buffer' && !Buffer.isEncoding(encoding))
    throw new Error('invalid encoding');
}
```

Now we have a safeguard for the `encoding` and `callback` parameters, let’s go to examine the `chunk` value.

First, we need to assure that the `chunk` value is not `null`. Because as we said before, the `null` value in the stream module is defined for a special purpose.

After that, we just need to check whether the `objectMode` is equal to `false`. Because if it is `true` then we don’t have to do anything with the value and just use it as it is.

But if the `objectMode` is `false` then we check if the type of value is not equal to `string` or `buffer` (as we said before here we ignore the `Uint8Array` type), then throw an error.

```javascript
if (chunk === null) {
  throw new Error('chunk cannot be null');
} else if (!state.objectMode) {
  if (typeof chunk === 'string') {
    if (state.decodeStrings !== false) {
      chunk = Buffer.from(chunk, encoding);
      encoding = 'buffer';
    }
  } else if (chunk instanceof Buffer) {
    encoding = 'buffer';
  } else {
    throw new Error('invalid types! only string and buffer are accepted');
  }
}
```

> *Note: Add these two parts of code at the beginning of the `write` method.*

After adding these parts and according to what we said before about the length of chunks when the `objectMode` is equal to `true`, we need to update the value of the `len` constant:

```javascript
const len = (state.objectMode ? 1 : chunk.length);
```

Also because we might need the `encoding` value when we read data from the buffer, so we should store it there, and in addition to that, we must pass this value to the `_write` method as well.

```javascript
if (state.writing || state.corked) {
  state.buffered.push({ chunk, encoding, callback });
} else {
  state.writing = true;
  state.writecb = callback;
  state.writelen = len;
  this._write(chunk, encoding, this._onwrite.bind(this));
}
```

we are done with updating the `write` method and we just need a few updates in the `_clearBuffer` and `_doWrite` methods to adapt together.

```javascript
_clearBuffer() {
  const state = this._writableState;
  if (state.corked)
    return;

  const { chunk, encoding, callback } = state.buffered.shift();
  this._doWrite(chunk, encoding, callback);
}

_doWrite(chunk, encoding, callback) {
  const state = this._writableState;
  state.writing = true;
  state.writecb = callback;
  state.writelen = (state.objectMode ? 1 : chunk.length);
  this._write(chunk, encoding, this._onwrite.bind(this));
}
```

In the end, we should just be nice and define a method to let the user set the `defaultEncoding` value after creating an instance.

```javascript
setDefaultEncoding(encoding) {
  if (!Buffer.isEncoding(encoding))
    throw new Error('invalid encoding');
  this._writableState.defaultEncoding = encoding;
  return this;
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/e15a7ea3298ac4093bf4e2f136cae4841b1ffcf8)

The next feature that our module needs to have is an approach that enables the user - who writes to the stream - to inform the destination that he is done with writing and the destination can do its cleaning.

In order to do that we need to implement a method named `end` which can also take the last chunk - but it’s arbitrary – and an `encoding` parameter. Then after writing the last chunk if there is any, we also check if the stream is corked then let’s uncork it to flush the buffer, and at the end, if there wouldn’t be any other problem, maybe we could finish it by emitting a `finish` event.

First of all, as always, we need to define some flags to be able to track the current state of the stream later.

```javascript
this.ending = false;    // the stream in the ending process
this.ended = false;     // the stream has already ended
this.finished = false;	// the event finish is emitted
```

After defining the flags, let’s implement the required methods to manipulate them.

```javascript
end(chunk, encoding) {
  const state = this._writableState;

  if (chunk !== null && chunk !== undefined)
    this.write(chunk, encoding);

  if (state.corked) {
    state.corked = 1;
    this.uncork();
  }

  state.ending = true;
  this._finishMaybe();
  state.ended = true;
}
```

The definition of the `end` method is pretty straightforward – as we discussed before.

In the middle of the function, we set the value of `corked` to 1 to ensure that the `uncork` method could flush the buffer on the first try.

At the end of the function, we just call the `_finishMaybe` method to let it finish the stream if it is ready to be finished.

```javascript
_finishMaybe() {
  const state = this._writableState;
  if (this._needFinish()) {
    state.finished = true;
    this.emit('finish');
  }
}
```

The `_finishMaybe` method doesn’t do anything special, it just checks the required conditions to see whether the stream needs to be finished. If so then it changes the `finished` flag to `true` and emits a `finish` event.

Also, these are the necessary conditions for a stream to be finished:

```javascript
_needFinish() {
  const state = this._writableState;
  return (
    state.ending &&
    state.length === 0 &&
    !state.finished &&
    !state.writing
  );
}
```

Now we need to use an `if` statement inside the `write` method to check if the stream has already been ended then avoid continuing by throwing an error.

```javascript
if (state.ending)
  throw new Error('cannot write after stream has already been ended');
```

Put this statement before the line for checking the type of the `_write` method.

The next update that we need is inside the `_onwrite` method.

First, add another condition on emitting the `drain` event.

```javascript
if (state.needDrain && state.length === 0 && !state.ending) {
  state.needDrain = false;
  this.emit('drain');
}
```

Then call the `_finishMaybe` at the end of the method.

```javascript
this._finishMaybe();
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/938364a2a1b052dd277ba94ea868803dfb14cd65)

The next thing after ending a stream that might happen is destroying the stream and then closing it.

Destroying a stream is nothing more than changing a bunch of flags. The required flags to be modified on destroying the stream include:

```javascript
this.destroyed = false;
this.emitClose = true;	// also make it as an option that user could set it
this.closed = false;
```

And we need to implement the `end` method which is so simple:

```javascript
destroy() {
  const state = this._writableState;
  state.destroyed = true;
  if (state.emitClose)
    this._close();
  return this;
}
```

Just change the `destroyed` flag and then run the `_close` method to emit the `close` event if the `emitClose` option is `true`.

```javascript
_close() {
  this._writableState.closed = true;
  process.nextTick(() => {
    this.emit('close');
  });
}
```

And inside the `_close` method after changing the `closed` flag to `true`, then emit the `close` event in the next tick – which means emit it after running the all synchronous codes.

There is also another flag named `autoDestroy` that lets the stream be destroyed after it is ended.

```javascript
this.autoDestroy = true;	// also make it as an option that user could set it 
```

Then inside the `_finishMaybe` method after emitting the `finish` event, just check if this flag is equal to `true` then call the `destroy` method.

```javascript
_finishMaybe() {
  const state = this._writableState;
  if (this._needFinish()) {
    state.finished = true;
    this.emit('finish');
    if (state.autoDestroy) {
      this.destroy();
    }
  }
}
```

To finish this feature, we only need to check the `destroyed` state in some places.

Let’s start with the `write` method and add this `if` statement after checking for the `ending` flag.

```javascript
if (state.destroyed)
  throw new Error('cannot write after stream has already been destroyed');
```

Then add another condition to the `_onwrite` method on emitting the `drain` event.

```javascript
if (state.needDrain && state.length === 0 && !state.ending && !state.destroyed) {
  state.needDrain = false;
  this.emit('drain');
}
```

Also, update the `if` statement in the `_clearBuffer` method.

```javascript
_clearBuffer() {
  const state = this._writableState;
  if (state.corked || state.destroyed)
    return;

  const { chunk, encoding, callback } = state.buffered.shift();
  this._doWrite(chunk, encoding, callback);
}
```

And finally, update the `end` method to check if it has already been destroyed then throw an error.

```javascript
if (state.destroyed) {
  throw new Error('write after destroyed');
} else {
  state.ending = true;
  this._finishMaybe();
  state.ended = true;
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/55ae2090284b4b22cd16990c9423289656096275)

Now our module needs a way to handle the errors. Because as you might have noticed, all the errors that occur in the methods do not need to be thrown directly. The better way is to emit an `error` event and pass an error object to the listeners and also sometimes we can use some callbacks to pass the error through them.

Let’s define a new property named `errored`:

```javascript
this.errored = null;
```
And then define a method named `_onError` that is just in charge of emitting `error` event with the value of `errored` property.

```javascript
_onError() {
  process.nextTick(err => {
    this.emit('error', err);
  }, this._writableState.errored);
}
```

We pass the value of `errored` as the parameter of `process.nextTick` callback instead of using it directly, to make sure that we have a copied value of the `errored` until the next tick because its value might be changed again in the current round.

Now we have a method for emitting errors, let’s see where we need to use it or we can pass the error through the callbacks.

First, start with the `write` method and remove those three conditions in the middle of the function for checking `ending` and `destroyed` flags and also the type of `_write` method. Then instead of them put this part of the code there:

```javascript
if (state.ending) {
  state.errored = new Error('cannot write after stream has already been ended');
} else if (state.destroyed) {
  state.errored = new Error('cannot write after stream has already been destroyed');
}
if(state.errored) {
  if (typeof callback === 'function')
    process.nextTick(callback, state.errored);
  return false;
}
```

And define a method named `_write` to just throw an error. It’s a nicer way to force the destination to implement this method.

```javascript
_write() {
  throw new Error('_write() must be implemented!');
}
```

After that let’s update the `_onwrite` method to takes an error parameter that might be passed from the destination.

```javascript
_onwrite(err) {}
```

And update part of its body as well.

```javascript
if (err) {
  state.errored = err;
  throw state.errored;
} else {
  if (state.buffered.length)
    this._clearBuffer();
  if (state.needDrain && state.length === 0 && !state.ending && !state.destroyed) {
    state.needDrain = false;
    this.emit('drain');
  }
  if (typeof cb === 'function')
    cb();
}
```

Then update the conditions in the `_needFinish` method to add the `errored` property there.

```javascript
_needFinish() {
  const state = this._writableState;
  return (
    state.ending &&
    state.length === 0 &&
    !state.errored &&
    !state.finished &&
    !state.writing
  );
}
```

We also need to update the `end` method and accept another parameter as a callback to pass the error if needed.

```javascript
end(chunk, encoding, cb) {}
```

Before using the `cb` parameter in this method, let’s add an `if` statement to keep the flexibility of this method for calling with a different order of arguments.

Add this part of code at the beginning of the function:

```javascript
if (typeof chunk === 'function') {
  cb = chunk;
  chunk = null;
  encoding = null;
} else if (typeof encoding === 'function') {
  cb = encoding;
  encoding = null;
}
```

And then update the body of this if statement:

```javascript
if (state.destroyed) {
  state.errored = new Error('write after destroyed');
  if (typeof cb === 'function') {
    process.nextTick(cb, state.errored);
  }
}
```

> *Note: Don’t ruin the `else` body and leave it as it is.*

After updating the `end` method, it’s time to go to the `destroy` method.

First, let this method get two parameters which include an error and a callback.

```javascript
destroy(err, cb) {}
```

And then inside this method first check if the type of `cb` is `function`, then call it and pass the `err` value to it.

After that, check if the `err` parameter has any truthy value then put it in the `errored` property and call the `_onError` method to emit an `error` event.

This is how this method would look like after the changes:

```javascript
destroy(err, cb) {
  const state = this._writableState;
  state.destroyed = true;

  if (typeof cb === 'function')
    cb(err);

  if (err) {
    state.errored = err;
    this._onError();
  }

  if (state.emitClose)
    this._close();

  return this;
}
```

Also, there is another method named `_destroy` that is supposed to be implemented by the destination - but it is arbitrary – which could be in charge of informing the destination that the stream object is destroying and it also has two parameters for passing the existing error and also a callback to get the latest error from the destination to be emitted in the `error` event.

This is the alternative implementation of the `_destroy` method if the destination wouldn’t redefine it.

```javascript
_destroy(err, cb) {
  cb(err);
}
```

And this is also the implementation of the callback that is going to be passed to the `_destroy` method.

```javascript
_onDestroy(err) {
  if (err) {
    this._writableState.errored = err;
    this._onError();
  }
}
```

And we are going to call the `_destroy` method at the end of the `destroy` method.

```javascript
this._destroy(state.errored, this._onDestroy.bind(this));
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/1e3857af1b58da98370f4d550c1bb0c48729c549)

The strategy for handling errors in the built-in stream module is more complicated compare to what we did here. But the idea of handling the errors is the same.

The other thing that we are going to take care of is a `_construct` method, that might be implemented by the extended class. If this method is implemented, we should make sure that some operations have a safe behavior until this method be executed.

Usually, this method is used to do some kind of stuff like opening a connection for further pushing data to a destination or similar to that – so it might even have some asynchronous operations.

For handling this circumstance, let’s define a flag named `constructed` with the default value of `true` that can be used to check whether the implemented `_construct` method is executed.

```javascript
this.constructed = true;
```

At the end of the `constructor` method of the `Writable` class put this part of code.

```javascript
if (typeof this._construct === 'function') {
  const state = this._writableState;
  state.constructed = false;
  process.nextTick(() => {
    this._construct(err => {
      state.constructed = true;
      if (err)
        state.errored = err;
      this._finishMaybe();
    });
  });
}
```

We just run it in the next tick of the event loop and pass it a callback to catch the further possible error.

Also, we call the `_finishMaybe` to make sure about the current state of the stream. Because we don’t know what was happened inside the `_construct` method and the stream might need to be finished at this point.

After that update the condition of pushing data to the buffer in the `write` method to avoid any data loss.

```javascript
if (state.writing || state.corked || !state.constructed) {
  state.buffered.push({ chunk, encoding, callback });
}
```

Also, add this condition to the `_clearBuffer` method.

```javascript
if (state.corked || state.destroyed || !state.constructed)
  return;
```

And finally, update the `_needFinish` method.

```javascript
_needFinish() {
  const state = this._writableState;
  return (
    state.ending &&
    state.constructed &&
    state.length === 0 &&
    !state.errored &&
    !state.finished &&
    !state.writing
  );
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/297983e0e4a3475f82709ab8a8c29588a9d36ab8)

The last thing that we are going to do to make our module to be more flexible is to enable it to get some methods i.e. `_write`, `_destroy` and `_construct` from the `options` object that is given on instantiating.

Doing that is so simple and we just need to add a bunch of `if` statements to the `constructor` of the `Writable` class.

```javascript
if (options) {
  if (typeof options.write === 'function')
    this._write = options.write;

  if (typeof options.destroy === 'function')
    this._destroy = options.destroy;

  if (typeof options.construct === 'function')
    this._construct = options.construct;
}
```

> *Note: These methods still can be implemented through extending class. We just added another option if the user wants to only create an instance of the `writable` stream without creating an extended class.*

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/b7fcfad30709047b37b0592a360de5919f890e5b)

At this moment, our `writable` stream module is complete and we can finish this chapter by implementing an example to use our module.

Here we plan to create a polyfill of the `fs.createWriteStream` method by using our own `writable` module.

Now we just create a file then inside of it `require` our `writable` module along with the built-in `fs` module over there.

Next, create a new brand class to extend from our `Writable` class, and then we only need to define the required methods i.e. `_construct`, `_write` and `_destroy`.

```javascript
class CreateWriteStream extends Writable {
  constructor(filePath, options) {
    super(options);
    this._filePath = filePath;
  }
  _construct(callback) {
    fs.open(this._filePath, 'w', (err, fd) => {
      if (err) {
        callback(err);
      } else {
        this._fd = fd;
        callback();
      }
    });
  }
  _write(chunk, encoding, next) {
    fs.write(this._fd, chunk, next);
  }
  _destroy(err, callback) {
    fs.close(this._fd, er => callback(er || err));
  }
}

const createWriteStream = (filePath, options) => new CreateWriteStream(filePath, options);
```

In the end, we create a function that returns a new instance of the `CreateWriteStream` every time.

> *Note: Don’t forget to export the created function.*

Now you can use the `createWriteStream` as an alternative to the `fs.createWriteStream` method.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/05e92aaad3638b76ec803f26bd38e513e60a2e4d)

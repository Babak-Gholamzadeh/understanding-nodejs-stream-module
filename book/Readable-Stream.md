# Readable Stream

The `readable` stream is the next stream module that we are going to study. The general idea of the `readable` stream is similar to the `writable` stream but kind of in a reverse way.

The `readable` stream has more responsibilities and methods than the `writable` and also has a method named `pipe` (which you might have heard before and we will discuss it later) that somehow is in charge of managing the `writable` stream either – this is one of the main reasons that first we examined the `writable` stream.

<p align="center">
  <img alt="readable stream" src="/book/assets/figure-10_readable-stream.png" />
</p>

According to the image above it seems that the whole responsibility of the `readable` stream is to somehow make it possible to read chunk by chunk of the pushed data – which is pushed by the source - from a `readable` stream object.

A `readable` stream object provides a variety of ways to read from it and we will investigate all of them.

Let’s start to implement our `readable` stream module by building a `class` based structure.

```javascript
// =============================
//   Inside readable.js module
// =============================
class Readable {
  constructor() {}
}
```

Before implementing different methods for reading from the stream, first, we ought to examine to see how the source is going to push its data to the stream.

The `readable` stream has a method named `push` to get the chunk of data from a source.

```javascript
push(chunk) {}
```

Now for reading the chunks from the stream we can’t just create another method to read the chunks.

Because there isn’t any connection that could act as a bridge between these two methods that enable us to get the pushed chunk from the `push` method unless we define a property on `this` object to store the pushed chunk that the other method could read from that property, or let the user define the second method for us and we just call it inside the `push` method.

Here we plan to use an event system that seems to be a much cleaner approach.

So inside the `push` method we just emit a `data` event and send the pushed chunk to the listeners.


In order to use an event system, first, we need to `require` an events module – as we said before you are free to use our implemented events module or the built-in one.

```javascript
const EventEmitter = require('./events');
```

After requiring an events module then let the `Readable` class extends from that.

> *Note: Don’t forget to call the `super` method inside of the `constructor` of the `Readable` class to construct the `EventEmitter` class.*

```javascript
class Readable extends EventEmitter {
  constructor() {
    super();
  }
}
```

Now that we can use the event system, let’s emit a `data` event inside the `push` method.

```javascript
push(chunk) {
  this.emit('data', chunk);
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/17a326f48473a875e9ea8e66c1ba8994fe00d482)

So far we have reached our purpose to somehow get the pushed chunks from the source.

But we don’t have any control over it and also this strategy has some disadvantages. For example, we would lose data if we don’t have any listener for the `data` event and we cannot notify the source that we haven’t consumed the previous chunks yet or pause it somehow.

To overcome these issues, first of all, we need to be able to buffer the data in the memory.

Because in this module we must have more control over the buffer, so we should implement a queue module for buffering data that could meet our requirements.

The `Buffer` object in Node.js is so similar to the array object. Thus it is already a list of data but we are going to use a combination of singly linked list structure and `Buffer` object.

> *Note: If you already know how a queue of `Buffer` nodes works, you can skip this part or just skim it and go for the next part.*

If you don’t know about linked list structure, it is simply a list of nodes – imagine a node as a JavaScript object - in the memory that each node usually contains a data property and some properties for pointing to the other nodes.

A singly linked list is just a type of linked list that each node of it only points to the next node.

<p align="center">
  <img alt="singly linked list" src="/book/assets/figure-11_singly-linked-list.png" />
</p>

For traversing a linked list, we can’t access each node directly and we have to always start to traverse from the first node – it’s called `head` - and access the next node through the current node and so on.

Now let’s implement our `BufferList` module by building a `class` based structure as usual.

```javascript
// ================================
//   Inside buffer-list.js module
// ================================
class BufferList {
  constructor() {}
}
```

The required properties that we need to implement a queue are a `head` property to point to the first node of the queue, a `tail` property to point to the last node of the queue, and also a `length` property to store the number of nodes in the queue.

```javascript
constructor() {
  this.head = null;
  this.tail = null;
  this.length = 0;
}
```

Before going any further, we should define a `Node` class to instantiate new nodes for the queue.

```javascript
class Node {
  constructor(data, next = null) {
    this.data = data;
    this.next = next;
  }
}
```

It takes two arguments to build a new node. The `data` argument is obvious but the `next` argument is used to point to another node – like a chain.

Now it’s time to implement the required methods.

The first method we are going to implement is a `push` method to add a new node to the end of the queue.

```javascript
push(data) {
  const node = new Node(data);
  if (this.length > 0)
    this.tail.next = node;
  else
    this.head = node;
  this.tail = node;
  ++this.length;
}
```

In this method first, we create a new node that contains the data.

For adding a new node to a queue, there are two possible states. If the queue is not empty, we need to make the `next` property of the `tail` node points to the new node and then the `tail` itself points to the new node because the `tail` should always point to the last node.

But if the queue is empty, we need to make the `head` points to the new node and then make the `tail` points to the new node too.

> *Note: In both situations, the `tail` needs to point to the new node anyway.*

In the end, we should increase the `length` to could track the size of the queue.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/4a1c89c7e3b736a7720966a01e01ef918c4fe7be)

The next method that we are going to implement is named `unshift` to add a new node to the front of the queue. There are some situations that this method can be useful.

```javascript
unshift(data) {
  const node = new Node(data, this.head);
  this.head = node;
  if (this.length === 0)
    this.tail = node;
  ++this.length;
}
```

First, we create a new node again but because with the `unshift` method the new node must be added to the front of the queue, so the `next` property of the new node should always point to the `head` and then the `head` itself points to the new node.

After that, we check if the queue is empty, make the `tail` points to the new node either, and then increase the `length`.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/dce2be0f40ecf904b6007e70854664adb6da355e)

The next method is named `shift` that we need to implement to return the data (only data not the node) of the first node in the queue.

```javascript
shift() {
  if (this.length === 0)
    return;
  const ret = this.head.data;
  if (this.length === 1)
    this.head = this.tail = null;
  else
    this.head = this.head.next;
  --this.length;
  return ret;
}
```

First, check if the queue is empty just returns nothing (`undefined` value). Otherwise after storing the `data` value of the `head` in a variable, then check if the size of the queue is equal to 1, which means that the `head` and `tail` both point to the same node, so make both of them equal to `null`.

But if the queue has more than one node, then make the `head` points to the next node in the queue. Also, don’t forget to decrease the `length` value before returning the data.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/c3914b5458525119451539a6309307ccf654b09e)

We also need a method to make the queue empty.

```javascript
clear() {
  this.head = this.tail = null;
  this.length = 0;
}
```

Making a queue empty is just like resetting the queue properties to their initial values.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/e4e7536336b30b446ede4aa9be06054231017a5c)

We should also define a very short but handy method named `first` just to get the data of the first node in the queue.

```javascript
first() {
  return this.head.data;
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/8b648b9da3d7f831a8ee16ed64177188c6b121e7)

Now it’s time to implement some more complicated methods to aid to work with `Buffer` data more efficiently.

As we said before because this queue must be able to store the data with the type of `Buffer` and also because the `Buffer` object is like a list of bytes, so we need a method to return `n` bytes of data from the queue.

To understand it better, let’s imagine the queue contains two nodes and each node contains a `Buffer` data with the size of 10 bytes. Now if we want to read 15 bytes of data, we need a method to return the concatenation of the whole bytes of the first node plus the half of the second one.

But keep in mind that this method just returns the data and won’t remove them from the queue.

```javascript
concat(n) {
  if (this.length === 0)
    return Buffer.alloc(0);
  const ret = Buffer.allocUnsafe(n >>> 0);
  let p = this.head;
  let i = 0;
  while (p) {
    Uint8Array.prototype.set.call(ret, p.data, i);
    i += p.data.length;
    p = p.next;
  }
  return ret;
}
```

Since this method is supposed to work with `Buffer` data, so it concatenates the data inside a new `Buffer`.

The `alloc` and `allocUnsafe` methods of `Buffer` just allocate a new `Buffer` of `n` bytes in the memory. The difference between them is when you use the `alloc` method, it automatically fills the allocated memory with zero value. But the `allocaUnsafe` method does not initialize the allocated memory and you might even see it contains the previous data that are left in the memory – but this method is faster.

Using unsigned right shift operator `>>>` with a zero value converts any float number to an integer number and any value that cannot be converted to a number will become 0.

After allocating a `Buffer` with the size of `n` then we just need to traverse the linked list to read the required length of bytes of data from the nodes.

The purpose of using this line of code:

```javascript
Uint8Array.prototype.set.call(ret, p.data, i);
```

is because we have already had an allocated `Buffer` and the value of `p.data` is also a `Buffer`, so we need a way to put the value of `p.data` to the `ret` by starting from the offset of `i`, and also `Buffer` itself doesn’t have such a method.

The `Uint8Array` is a JavaScript type and the `Buffer` is only a subclass of the `Unit8Array` that is implemented by Node.js. So a `Buffer` object is kind of a `Uint8Array` object and because of that by binding a `Buffer` object to the `set` method of the `Uint8Array` class we can benefit from that.

> *Note: If we don’t use this method, another approach is to loop through the bytes of `p.data` and put each byte inside of the `ret` and also take care of the offset of `i`.*

The only problem with this `set` method is that if the length of the `p.data` be greater than from the offset to the end of the `ret` length, it would throw a `RangeError` because it loops through the `p.data` bytes. But we don't need to take care of that here because we only use the `concat` method to read the entire data in the list of nodes and we won’t send an `n` value that is out of range – in the implementation of the next method, we will overcome this problem where we need it.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/fa7d4bfa4251f1b1a22e689fd3ff2cf7998bf0eb)

The last but not least method that we should implement is a method for consuming `n` bytes of data – which means return them and also remove them from the queue.

Let’s create a method named `consume` that takes a parameter named `n`.

```javascript
consume(n) {}
```

There are three possible states for consuming `n` bytes of data that we need to take care of them inside this method.

The first state is when the value of `n` is less than the length of the value of `data` in the `head`, then use the `slice` method to slice `n` bytes of data from the `data` property. Also before returning that piece of data, we should use the `slice` method again to put the rest of the data back to the `data` property.

```javascript
const data = this.head.data;
if (n < data.length) {
  const ret = data.slice(0, n);
  this.head.data = data.slice(n);
  return ret;
}
```

The second state is when the value of `n` is equal to the length of the value of the `data` property in the `head`, then we just need to use the `shift` method we implemented earlier and return it.

```javascript
if (n === data.length)
  return this.shift();
```

The third – aka the last - state is the most complicated one. Because you need to allocate a buffer with enough space, then traverse the queue and set the data to the allocated buffer. Also, you must move the `head` forward as reading its data – don’t forget the `tail` if you reach the end of the queue too.

```javascript
const ret = Buffer.allocUnsafe(n);
const retLen = n;
let p = this.head;
let c = 0;
do {
  const buf = p.data;
  if (n > buf.length) {
    Uint8Array.prototype.set.call(ret, buf, retLen - n);
    n -= buf.length;
  } else {
    if (n === buf.length) {
      Uint8Array.prototype.set.call(ret, buf, retLen - n);
      ++c;
      if (p.next)
        this.head = p.next;
      else
        this.head = this.tail = null;
    } else {
      Uint8Array.prototype.set.call(ret,
        new Uint8Array(buf.buffer, buf.byteOffset, n),
        retLen - n);
      p.data = buf.slice(n);
      this.head = p;
    }
    break;
  }
  ++c;
} while (p = p.next);
this.length -= c;
return ret;
```

The code is sort of straightforward and you would understand it by walking through it slowly.

The only thing that might confuse you is this part:

```javascript
Uint8Array.prototype.set.call(ret,
  new Uint8Array(buf.buffer, buf.byteOffset, n),
  retLen - n);
```

This part is a solution to the problem that we pointed in the implementation of the previous method (`concat`). The problem occurs when the length of the second argument be greater than the remained length of the first argument mines the value of the third argument.

This line of code can solve the problem by creating a new buffer with the required size.

```javascript
new Uint8Array(buf.buffer, buf.byteOffset, n)
```

> *Note: If you are curious about Node.js `Buffer` and want to know what exactly are the values of `buf.buffer` and `buf.byteOffset` you should check out the [Node.js documentation](https://nodejs.org/dist/latest-v15.x/docs/api/buffer.html), because it is out of the scope of this book.*

At this moment, we implemented the required methods to work with our `BufferList` module and we are finished.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/c39b72510e81530306bb188ede46ffd1cce77554)

Now we have a `BufferList` module so we can go back to the `readable` stream module to continue the implementation.

As we discussed before, we created a `BufferList` module to use to buffering data to avoid data losing as much as possible. But how are we supposed to benefit from that?

If we review the simple workflow of our `readable` module, we can see that the data is always flowing through it. But we need a methodology to control this flowing.

In order to do that, the built-in Node.js stream module came up with an idea of defining two modes for the module – when the data flow through the stream and when they just are buffered in the memory. Thus a flag is defined in the module named `flowing` to use to switch between these two modes.

For adding states to our module, we should do what we did with the `Writable` module. So first let’s build another class named `ReadableState`.

```javascript
class ReadableState {
  constructor() {}
}
```

And then add the required properties to its `constructor`.

```javascript
constructor() {
  this.buffer = new BufferList();
  this.length = 0;
  this.flowing = null;
}
```

> *Note: Don’t forget to `require` the `BufferList` module.*

The `length` property is for tracking the current size of data in the buffer.

If you wonder about the value of the `flowing` flag – because we said it only has two modes earlier – we can see that it can be a `null` value that causes to have more control over changing the value of this flag, now we can also find out a little more about its current mode – it will be used in some conditions later. The `null` value here is considered as a non-flowing mode.

Also after adding the properties to the `ReadableState` class, we should create an instance of this class inside the `constructor` of the `Readable` class.

```javascript
constructor() {
  super();
  this._readableState = new ReadableState();
}
```

Hence the default value of the `flowing` stats is equal to a non-flowing value, so we need to make sure that the pushed chunks of data won’t be emitted to the `data` event unless the flowing mode is switched. And also we should buffer the data while the `flowing` flag is not `true` - which could be `false` or `null`.

```javascript
push(chunk) {
  const state = this._readableState;
  if (state.flowing) {
    this.emit('data', chunk);
  } else {
    state.length += chunk.length;
    state.buffer.push(chunk);
  }
}
```

Certainly, now we need another method to change the value of the `flowing` flag to `true`.

```javascript
resume() {
  const state = this._readableState;
  state.flowing = true;
}
```

Let’s review the workflow again. At first, the module is in the paused mode. At this moment if we first call the `resume` method to change the mode and then push data, the data will be emitted to the `data` event even when there is no listener.

We can fix this issue by adding another condition to the `if` statement for emitting data to ensure that we have any listener for the `data` event.

```javascript
if (state.flowing && this.listenerCount('data') > 0) {
  this.emit('data', chunk);
}
```

Great.

Another possible way for calling the methods is when first we push some data and because it’s in paused mode, the data will be buffered and after that, calling the `resume` method causes the `flowing` flag to become `true` and if we have any listener for the `data` event, the newly coming chunks will be sent while the previous chunks remain in the memory.

To overcome this problem, first, we ought to add another condition to the `if` statement of the `push` method to guarantee that the new chunks won’t be sent while there exist some data in the buffer.

```javascript
if (state.flowing && this.listenerCount('data') > 0 && state.length === 0) {
  this.emit('data', chunk);
}
```

We also need to consume the buffered data after start flowing. For doing that we need a method to call it in the `resume` method that reads the existing chunks in the buffer and sends them to the `data` event as well.

```javascript
read() {
  const state = this._readableState;

  let ret = null;
  if (state.length > 0)
    ret = state.buffer.shift();

  if (ret !== null) {
    state.length -= ret.length;
    this.emit('data', ret);
  }

  return ret;
}
```

Now we must keep calling this method inside the `resume` method as long as there is data to be read.

```javascript
resume() {
  const state = this._readableState;
  state.flowing = true;
  while (this.read() !== null);
}
```

> *Note: As you might have noticed, the emitting `data` event inside the `read` method would cause it to lose data if no listener is defined at that point. But don’t worry about it because this issue has existed in the built-in stream module either.*

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/71d26eb30b86981c0041449794ef904756591a6c)

So far, in order to be able to read the chunks from the `readable` stream, we have to call the `resume` method explicitly. But it could be better if our stream module was smart enough to change the mode when we add a listener to the `data` event. Because when we add a listener, obviously we are going to read from the stream.

For the reason of overcoming this feature, we need to override the `on` method that is used to add listeners to our stream module. But we are not going to override it inside the events module because we require the main functionality of the `on` method either.

```javascript
on(eventName, listener) {
  super.on(eventName, listener);
}
```

The result of this implementation so far is exactly as before because we just get the required parameters and pass them to the main `on` method in the superclass – which is the EventEmitter class.

> *Note: Because the term `class` in JavaScript does not mean the same thing as it does in other languages, so overriding a method of superclass might be a little different and we need to call the `super.method` in our implemented method explicitly.*

Now let’s use this overridden method to change the mode on adding a `data` event listener.

```javascript
on(eventName, listener) {
  super.on(eventName, listener);

  if (eventName === 'data')
    this.resume();
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/f1326ccf1029cac2f83bc9982518d99e0548ab11)

Another issue that we should handle is when we push data and then call the `resume` method and finally add a listener to the `data` event synchronously. In that circumstance, we will lose some data.

To handle that we need to upgrade the `resume` method a little bit.

Whenever you want to run some codes after the synchronous codes, simply you can run them in the next tick of the event loop by using some methods like `process.nextTick`.

But until some codes are going to run in the next tick, we should be careful about the states that might affect the codes.

Thus we define a flag that could help us to avoid recalling the `process.nextTick` before the previous one has not been run yet.

```javascript
this.resumeScheduled = false;
```

Because the main part of the `resume` method is the `while` loop that causes the `read` method to be called constantly, we put that part inside another method named `_resume`.

At this moment, we have the method that is going to run in the next tick plus the required flags to control this process, we just need to add an `if` statement to the `resume` method to use the `process.nextTick` method inside of its body.

This is how the `resume` method looks after the changes:

```javascript
resume() {
  const state = this._readableState;
  state.flowing = true;
  if (!state.resumeScheduled) {
    state.resumeScheduled = true;
    process.nextTick(this._resume.bind(this));
  }
}
```

And this is inside of the `_resume` method that is ready to run in the next tick:

```javascript
_resume() {
  const state = this._readableState;
  state.resumeScheduled = false;
  this.emit('resume');
  while (this.read() !== null);
}
```

Besides, we thought that this could be a good idea to notify the user that the resume operation starts running by emitting a `resume` event.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/e5539a8a4c823ef767a6e22acd0c0b0d3ddb9c4a)

Due to the existed methodology of having two modes in our stream module and the methods for changing the current mode to a flowing mode, we need another method to turn it back to the paused mode likewise.

But despite the `null` value in the `flowing` flag that could affect the workflow, we should be careful about the conditions in the methods that be affected by this flag.

```javascript
pause() {
  const state = this._readableState;
  if (state.flowing !== false) {
    state.flowing = false;
    this.emit('pause');
  }
  return this;
}
```

Because we emitting a `pause` event here, we ought to first check if the current mode is not already paused then change the flag and emit the event.
This could be a good idea that we can apply it to the `resume` method as well.

```javascript
resume() {
  const state = this._readableState;
  if (!state.flowing) {
    state.flowing = true;
    if (!state.resumeScheduled) {
      state.resumeScheduled = true;
      process.nextTick(this._resume.bind(this));
    }
  }
}
```

And because whilst the `while` loop inside the `_resume` method is calling the `read` method, the stream mode must be changed to paused mode, we should be informed and break the `while` loop. Additionally, it can be cool to make a separate method for this functionality.

```javascript
_resume() {
  const state = this._readableState;
  state.resumeScheduled = false;
  this.emit('resume');
  this._flow();
}
```

So inside the `_resume` method we just call the new `_flow` method.

```javascript
_flow() {
  const state = this._readableState;
  while (state.flowing && this.read() !== null);
}
```

And inside the `_flow` method we added another condition to the `while` for checking the `flowing` flag each round.

Also when we call the `pause` method and update the value of the `flowing` flag to `false` we can literary say that the mode is in paused mode now – because it could have a `null` value which is considered a non-flowing mode – so it could be useful if we have a method to check if the mode is in paused mode.

```javascript
isPaused() {
  return this._readableState.flowing === false;
}
```

Ultimately we shouldn’t forget about the `on` method that implicitly changes the mode by adding a listener to the `data` event.

When the user calls the `pause` method to pause the flowing mode explicitly, only by adding an event listener shouldn’t be allowed to change the mode. So we add an `if` statement to limit the action inside the `on` method.

```javascript
const state = this._readableState;
if (eventName === 'data') {
  if (state.flowing !== false)
    this.resume();
}
```

> *Note: After calling the `pause` method you can only call the `resume` method to change the mode.*

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/c70a695186fd8af2005ba788ff7bc1877ef5fc2d)

The next issue that should be covered, is when there isn’t any consumer on the other side of the `readable` stream, what is the point of pushing data when they would never be consumed? And so far with our implementation the source still has no idea that the chunks of data that it pushes, whether they are consumed.

To the reason of handling this circumstance, we can call a method that is going to be implemented by the source and assuming that the `push` method is called inside of that method. In this way, we can kind of give the awareness to the source that whenever this method is called, it means that there is a consumer on the other side that wants you to push more data.

First things first, we should define that method to throw an error to force the source to implement it.

```javascript
_read() {
  throw new Error('_read method must be implemented');
}
```

Now we can call this method inside the `read` method.

Because we assume that the `push` method is supposed to be called inside the `_read` method, so it’s better to be aware somehow that if the last time we called the `_read` method and the `push` method hasn’t been called yet, avoid recalling the `_read` method.

To handle that, we define a new flag named `reading` which is equal to `false` by default and becomes `true` on calling the `_read` method, then only inside of the `push` method would be turned back to `false` again.

```javascript
this.reading = false;
```

After defining the flag, add this part of the code to the beginning of the `read` method.

```javascript
if (!state.reading) {
  state.reading = true;
  this._read();
}
```

Then add this code at the beginning of the `push` method too (before the `if` statement).

```javascript
state.reading = false;
```

Now there exist some corner cases that make our code a little buggy.

Because of the conditions in the `if` statement of the `push` method, the chunk of data would be sent to the `data` event immediately before the `read` method be called.

```javascript
if (state.flowing && this.listenerCount('data') > 0 && state.length === 0) {
  this.emit('data', chunk);
}
```

And also because we want the reading process to be started by calling the `read` method, we need to define a flag named `sync` to be set to `true` by default to avoid this action and make the emitting of the `data` event to happen on the next tick.

```javascript
this.sync = true;
```

Update the body of the `if` statement that we added to the `read` method earlier.

```javascript
if (!state.reading) {
  state.sync = true;
  state.reading = true;
  this._read();
  state.sync = false;
}
```

Finally, add another condition to the `if` statement in the `push` method.

```javascript
if (state.flowing && this.listenerCount('data') > 0 && state.length === 0 && !state.sync) {
  this.emit('data', chunk);
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/422c19e35aea33fc24997f7106b040c91a1b9b95)

So far we sort of have control over the calling of the `push` method. But we don’t have any control over the size of the buffer yet. To handle that we are intending to use the same approach as we used for the `writable` stream, which is defining a `highWaterMark` property.

Accordingly, inside the `constructor` of the `ReadableState` class, we define the `highWaterMark` with the default value of 16K and also make its value optional that the user could change it on instantiating.

```javascript
class ReadableState {
  constructor({ highWaterMark = 16 * 1024 }) {
    this.buffer = new BufferList();
    this.length = 0;
    this.flowing = null;
    this.resumeScheduled = false;
    this.reading = false;
    this.sync = true;
    this.highWaterMark = highWaterMark;
  }
}
```

And then a few changes inside the `constructor` of the `Readable` class.

```javascript
constructor(options = {}) {
  super();
  this._readableState = new ReadableState(options);
}
```

Now we plan to make the `push` method to return a value based on the `highWaterMark`. At the end of the method add this code to notify the source whether the length of the buffer exceeds the threshold.

```javascript
return state.length < state.highWaterMark;
```

Similarly, the `read` method requires some changes for the conditions of calling the `_read` method to benefit from the `highWaterMark` value.

First, we need to define a method named `_howMuchToRead` to get to know about the size of chunks that we are about to read.

```javascript
_howMuchToRead() {
  const state = this._readableState;
  if (state.length === 0)
    return 0;
  return state.buffer.first().length;
}
```

Basically, this method returns the size of the first chunk of data in the buffer if the buffer is not empty. Then inside of the `read` method, first define a variable to store the returned value of the `_howMuchToRead` method.

```javascript
let n = this._howMuchToRead();
```

Then replace this part of the code with the `if` statement to calling the `_read` method.

```javascript
let doRead = false;

if (state.length - n < state.highWaterMark)
  doRead = true;

if (state.reading)
  doRead = false;
else if (doRead) {
  state.sync = true;
  state.reading = true;
  this._read(state.highWaterMark);
  state.sync = false;

  if (!state.reading)
    n = this._howMuchToRead();
}
```

> *Note: I know the structure of the `if` statements might sound a little confusing to you and you feel that could have been simpler and cleaner. But this is what is used in the built-in stream module.*

The whole point of these `if` statements are for recognizing when calling the `_read` method and when not.

So we assume that if after consuming the `n` bytes of data from the buffer, the size of the buffer is less than the `highWaterMark` and the value of `reading` flag is `false`, the `_read` method should be called and the value of `highWaterMark` should be passed to inform the source about the threshold.

Moreover, the intention of calling the `_howMuchToRead` method again is because at first, the buffer might be empty but after the calling of `_read` method and then in the case of pushing new data – that causes the `reading` flag to be changed – we need to calculate the value of `n` again.

As another update, it could be more helpful if we separate the code for consuming data from the buffer into another method.

We can do that by creating a method named `_fromList` to read from the buffer list.

```javascript
_fromList(n) {
  const state = this._readableState;
  return state.buffer.shift();
}
```

And then replace this code inside the `read` method with the one that just read from the buffer directly.

```javascript
ret = this._fromList();
```

> *Note: If this method seems unnecessary, don’t worry about it, because we are intending to handle more stuff inside of it later.*

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/a7e258b355b38754742651910b85fa2d2ec33bee)

For reading from a `readable` stream, there is more than one way. The other way that we are about to investigate is via using the `read` method explicitly.

The `read` method is a public method and also returns the read data, which means that it is capable to be used as an approach to read from the stream.

> *Note: We just distinguish the methods to be public or private by starting the name of the private methods with an underline (`_`) character as a convention.*

The only feature that we can add to the `read` method to make it more capable, is adding a parameter to it to represent the number of bytes that is going to be read.

We can make the `n` variable - that we defined before - as the parameter of this method.

Now because the value of `n` is given by the user, so it’s safer to be a little normalized.

Put this code at the beginning of the `read` method:

```javascript
if (n === undefined) {
  n = NaN;
} else if (!Number.isInteger(n)) {
  n = parseInt(n, 10);
}
const nOrig = n;
```

We replace the `undefined` value with a `NaN` value to use it in the conditions of the `_howMuchToRead` method.

> *Note: We store the original value of `n` in another variable called `nOrig` to use it later.*

Thus, update the `_howMuchToRead` method to accept the `n` as a parameter.

```javascript
_howMuchToRead(n) {
  const state = this._readableState;
  if (n <= 0 || state.length === 0)
    return 0;
  if (Number.isNaN(n)) {
    if (state.flowing && state.length)
      return state.buffer.first().length;
    return state.length;
  }
  if (n <= state.length)
    return n;
  return 0;
}
```

And now inside the `read` method, we should call the `_howMuchToRead` method by passing the value of `n` to normalize it.

```javascript
n = this._howMuchToRead(n);
```

But for the second calling of `_howMuchToRead` (in the middle of the method), we should pass the original value on `n` which is stored in the `nOrig` variable already. Because the first call of the `_howMuchToRead` might have changed the value of `n` so here it's safer to continue with the original one.

```javascript
n = this._howMuchToRead(nOrig);
```

Similarly, we should update the final part of this method with this new code.

```javascript
let ret = null;
if (n > 0)
  ret = this._fromList(n);

if (ret !== null) {
  state.length -= n;
  this.emit('data', ret);
}
```

As you can see here, we pass the `n` to the `_fromList` method to read the required bytes from the buffer. So we need to update that method either.

```javascript
_fromList(n) {
  const state = this._readableState;
  if (state.length === 0)
    return null;
  let ret;
  if (n >= state.length) {
    if (state.buffer.length === 1)
      ret = state.buffer.first();
    else
      ret = state.buffer.concat(state.length);
    state.buffer.clear();
  } else {
    ret = state.buffer.consume(n);
  }
  return ret;
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/0c9eab00bcd331a7fbe566c37b4529c70b19d1b6)

By adding the ability to the `read` method to read any size of data, we can be aware of the potential of consumer reading power. So if the consumer can read bigger chunks of data, why not benefit from that by increasing the value of `highWaterMark`.

But the question is, how much should we increase the `highWaterMark`?

A wise approach is to increase the `highWaterMark` based on the value on `n`. This means if the value of `n` is a small number, we should increase the `highWaterMark` in a tiny amount, but if the value of `n` is a large number, we should increase the `highWaterMark` massively.

> *Note: We only increase the `highWaterMark` if its value is less than the value of `n`.*

Also, we plan to follow an approach which is raising the value based on the power of 2.

For example, if the current value of `highWaterMark` is 10 and the value of `n` is 26 then we increase the `highWaterMark` to the smallest square number that is greater than `n`, which would be 32 here.

For doing that, first at this code to the `read` method before calling the `_howMuchToRead` method for the first time.

```javascript
if (n > state.highWaterMark)
  state.highWaterMark = this._computeNewHighWaterMark(n);
```

Now we need to implement the `_computeNewHighWaterMark` method to meet the expectation.

In order to achieve that functionality, we are using some bitwise operators.

To understand how we are going to use the bitwise operators for that operation, let’s investigate the required calculations along with the values from a binary perspective.

Let’s go with the number of the previous example. So we have the number 26 for the `n` which is represented in binary as:

<p align="center">
  <img alt="binary 26" src="/book/assets/figure-12_binary-26.png" />
</p>

And the output that we need to reach from this input is 32 that in binary is:

<p align="center">
  <img alt="binary 32" src="/book/assets/figure-13_binary-32.png" />
</p>

So as you noticed, by making the bit after the leftmost bit of the input to `1` and make all the bits from right to the leftmost bit to `0`, we can reach the desired output from any input.

But to implement this algorithm, it would be simpler first we reach from the input to the output minus `1` – which means at the end we need to add `1` to the output.

Thus, the output minus `1` is equal to:

<p align="center">
  <img alt="binary 31" src="/book/assets/figure-14_binary-31.png" />
</p>

Now it seems more achievable.

In order to reach from any input to this output we just need to find out their common bit, which is the leftmost bit that we sure it always equals `1`.

<p align="center">
  <img alt="binary 1x" src="/book/assets/figure-15_binary-1x.png" />
</p>

Regardless of the other bits, for making all of the `x` bits to be equal to `1`, we can use `|` (aka `OR`) operator with this current value and the value that is shifted to the right one time as its operands.

```javascript
n |= n >>> 1;
```

<p align="center">
  <img alt="binary 11x" src="/book/assets/figure-16_binary-11x.png" />
</p>

After that, repeat this operation with shifting one time, is kind of wasting time. Because at this point we sure that we have double `1`s next to each other. So for the optimizing purpose continue with value `2` that causes to have four value `1`s next to each other and so on.

As the result, we should repeat this process with the square numbers as shifting value and because the bitwise operators are performed on 32 bits binary numbers, so we only need to do that until we reach number 16.

Furthermore, the input should be minus 1 before starting the operation, because if the input is already a square number, we want that the output to be generated as the same as input without any effect. (i.e. 16 => 16)

```javascript
_computeNewHighWaterMark(n) {
  const MAX_HWM = 0x40000000;
  if (n >= MAX_HWM) {
    n = MAX_HWM;
  } else {
    n--;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    n++;
  }
  return n;
}
```

> *Note: There is a defined max value for the `highWaterMark` that guarantees its value never exceeds 1GB (HEX: 0x40000000).*

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/925e6c029835b72e5d5f4b3ed4e6554555bf63b6)

As we stated before that for reading from a `readable` stream, there is more than one way, now it’s time to examine the next approach.

Here we are intending to get help from an event named `readable` to read from the stream.

As it is mentioned in the [Node.js documentation](https://nodejs.org/dist/latest-v15.x/docs/api/stream.html#stream_event_readable):

> “The `readable` event is emitted when there is data available to be read from the stream. In some cases, attaching a listener for the `readable` event will cause some amount of data to be read into an internal buffer.”

Thus, according to the first part of the definition, we can use the `readable` event to figure out when there are some data in the buffer, and then via using the `read` method, start the reading process until it returns a `null` value.

It is more suitable to start the implementation with only the first part of the definition to avoid any further complexity and ambiguity.

First of all, we require a method that can be used to emit the `readable` event in the next tick.

For managing the code to run in the next tick and avoid scheduling new emitting before running the previous one, using some flags is essential.

The first flag we need is named `emittedReadable` to know whether there is any scheduled emitting already.

```javascript
this.emittedReadable = false;
```

The next flag which is named `needReadable` is going to be set in some places which we feel that emitting a `readable` event is needed, we set this flag to be used at some points for calling the `_emitReadable` method – that we plan to implement it.

```javascript
this.needReadable = false;
```

At this moment, we can implement the required method.

```javascript
_emitReadable() {
  const state = this._readableState;
  state.needReadable = false;
  if (!state.emittedReadable) {
    state.emittedReadable = true;
    process.nextTick(() => {
      if (state.length) {
        this.emit('readable');
        state.emittedReadable = false;
      }
    });
  }
}
```

The workflow is straightforward. First, we make the `needReadable` flag to `false`, after that, we check if there isn’t any scheduled emitting already then set it.

Also in the callback of `process.nextTick`, it’s better to check whether there is any data in the buffer before emitting the event. Because the buffered data might have been read via another way before that point.

The point for calling this method is inside of the `push` method after buffering the data. But for this method to be called over there, the value of the `needReadable` must be equal to `true`.

```javascript
if (state.needReadable)
  this._emitReadable();
```

Therefore, now we ought to find out the places and conditions that cause to make a need for emitting `readable` event.

By reviewing the definition again which is said “when there is data available to be read from the stream” we can realize that when the buffer was empty already and now we just pushed some new data to it, it’s the time for emitting a `readable` event.

Because the `_emitReadable` method is supposed to be called inside the `push` method, so before the `push` method to be called, we can verify if the buffer is empty, then make the value of `needReadable` to `true`.

Hence, inside the `read` method, exactly before the line code for calling the `_read` method put this code.

```javascript
if (state.length === 0)
  state.needReadable = true;
```

Plus, change the final part of the `read` method and add a bunch of `if` statements to make the value of `needReadable` equal to `true`.

```javascript
if (ret === null) {
  state.needReadable = state.length <= state.highWaterMark;
} else {
  state.length -= n;
}

if (state.length === 0) {
  state.needReadable = true;
}

if (ret !== null)
  this.emit('data', ret);

return ret;
```

We make the `needReadable` to be `true` at the end of the `read` method because it could also affect the calling of the `push` method after ending the `read` method.

The next spot is when we add a listener to the `readable` event, it causes the `needReadable` to become `true`.

```javascript
if (eventName === 'data') {
  if (state.flowing !== false)
    this.resume();
} else if (eventName === 'readable') {
  state.needReadable = true;
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/3da4982d626528f6765d167ccacd7b0ddbca6325)

Adding a listener to the `readable` event makes the stream mode be paused. Because from that on, the data are supposed to be read through the `read` method explicitly.

This action will change the behavior of some previous methods that are affected by the flowing mode. So we must take care of them.

Also, we are going to implement the second part to the `readable` event responsibility, which is “In some cases, attaching a listener for the `readable` event will cause some amount of data to be read into an internal buffer.”

First, let’s define another flag named `readableListening` to know whether there is any listener for the `readable` event.

```javascript
this.readableListening = false;
```

After that, we can go to the `on` method to add some changes.

```javascript
if (eventName === 'data') {
  state.readableListening = this.listenerCount('readable') > 0;
  if (state.flowing !== false)
    this.resume();
} else if (eventName === 'readable') {
  if (!state.readableListening) {
    state.readableListening = state.needReadable = true;
    state.flowing = false;
    state.emittedReadable = false;
    if (state.length) {
      this._emitReadable();
    } else if (!state.reading) {
      process.nextTick(() => {
        this.read(0);
      });
    }
  }
}
```

It sounds like a huge change but its workflow is pretty understandable. The call of the `read` method with the value `0` is the starting point for achieving the second part of the `readable` event responsibility.

At this point, we also need to add some codes to the callback of the `process.nextTick` inside the `_emitReadable` method.

```javascript
process.nextTick(() => {
  if (state.length) {
    this.emit('readable');
    state.emittedReadable = false;
  }
  state.needReadable = !state.flowing && state.length <= state.highWaterMark;
  this._flow();
});
```

We just determine the value of the `needReadable` flag to make it ready for calling the `_flow` method.

Besides, we can’t ignore the effect of it on the `resume` method.

```javascript
resume() {
  const state = this._readableState;
  if (!state.flowing) {
    state.flowing = !state.readableListening;
    if (!state.resumeScheduled) {
      state.resumeScheduled = true;
      process.nextTick(this._resume.bind(this));
    }
  }
}
```

Now because the operation of the `_resume` method might run in the paused mode, so we should add some changes to the `_resume` method either.

```javascript
_resume() {
  const state = this._readableState;
  if (!state.reading) {
    this.read(0);
  }

  state.resumeScheduled = false;
  this.emit('resume');

  this._flow();

  if (state.flowing && !state.reading)
    this.read(0);
}
```

In this way, we nearly have to control the flow explicitly by calling the `read` method with value `0` as its argument value,

Up till now, because the `read` method might be called from many spots, we need to manipulate the `emittedReadable` flag to be able to emit the `readable` event more than once in some circumstances by adding an `if` statement to the `read` method at this position.

```javascript
if (n > state.highWaterMark)
  state.highWaterMark = this._computeNewHighWaterMark(n);

if (n !== 0)
  state.emittedReadable = false;

n = this._howMuchToRead(n);
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/8512ec2a37db81794b8b1486149c8b13c15f416f)

By this time, adding a method to the end of the `push` method to try more for reading, can be useful for the workflow.

Because this method intends to start its operations on the next tick, we should have another flag to avoid it been recalled.

```javascript
this.readingMore = false;
```

After that, we can implement the method.

```javascript
_maybeReadMore() {
  const state = this._readableState;
  if (!state.readingMore) {
    state.readingMore = true;
    process.nextTick(() => {
      while (!state.reading && state.length < state.highWaterMark) {
        const len = state.length;
        this.read(0);
        if (len === state.length)
          break;
      }
      state.readingMore = false;
    });
  }
}
```

And now let’s call it at the end of the `push` method before the `return` statement.

```javascript
this._maybeReadMore();
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/ff023bed3b9d4ac90d746af6ad3314ca3f4da998)

The next thing that we plan to care about is the type of data that the `readable` stream works with them.

The supported types in the `readable` stream are exactly like the `writable` stream. It accepts three types of data include `string`, `Buffer`, and `Uint8Array` by default. There exists another flag named `objectMode` which is `false` by default and if you change it to `true` then your stream object can work with any JavaScript data type except `null` (the `null` value is defined for a special purpose).

> *Note: Also be mindful that when the value of `objectMode` is `true`, then the length of each chunk of data, regardless of its type, is always considered equal to one.*

Here we ignore the `Uint8Array` type for simplicity reason too.

These are the required flags:

```javascript
constructor({
  objectMode = false,
  defaultEncoding = 'utf8',
}) {
  this.objectMode = objectMode;
  this.defaultEncoding = defaultEncoding;
  this.encoding = null;
  // the rest of the properties are here
}
```

Then we need to update the `push` method to normalize the chunks. For making this method to be cleaner and more readable, we should create some other methods to divide the tasks of this method.

Besides, it’s a proper time to implement another method named `shift` that acts very similar to the `push` method, but in the opposite way, which means it adds the chunks to the front of the buffer list.

To meet our requirement, first, let’s define a method named `_addChunk` which will be in charge of buffering data or send them to the `data` event (similar to what the `push` method does now).

```javascript
_addChunk(chunk, addToFront) {
  const state = this._readableState;
  if (state.flowing && state.length === 0 && !state.sync && this.listenerCount('data') > 0) {
    this.emit('data', chunk);
  } else {
    state.length += (state.objectMode ? 1 : chunk.length);
    if (addToFront)
      state.buffer.unshift(chunk);
    else
      state.buffer.push(chunk);

    if (state.needReadable)
      this._emitReadable();
  }
  this._maybeReadMore();
}
```

As you can see, it takes another parameter - named `addToFront` - to know to add the chunks to the buffer from which side.

Afterward, we need another method named `_readableAddChunk` to get the chunks from the `push` or `shift` method and then call the `_addChunk` method to give the normalized chunks to it. Additionally, it takes care of the states and the returned value of the `push` and `shift` methods.

```javascript
_readableAddChunk(chunk, encoding, addToFront) {
  const state = this._readableState;

  if (!state.objectMode) {
    if (typeof chunk === 'string') {
      encoding = encoding || state.defaultEncoding;
      if (state.encoding !== encoding) {
        if (addToFront && state.encoding) {
          chunk = Buffer.from(chunk, encoding).toString(state.encoding);
        } else {
          chunk = Buffer.from(chunk, encoding);
          encoding = '';
        }
      }
    } else if (chunk instanceof Buffer) {
      encoding = '';
    } else if (chunk != null) {
      throw new Error('invalid types! only string and buffer are accepted');
    }
  }

  if (state.objectMode || (chunk && chunk.length > 0)) {
    if (addToFront) {
      this._addChunk(chunk, true);
    } else {
      state.reading = false;
      this._addChunk(chunk, false);
    }
  } else if (!addToFront) {
    state.reading = false;
    this._maybeReadMore();
  }

  return state.length < state.highWaterMark;
}
```

The first `if` statement is similar to the `writable` stream and we just do the type checking and encoding stuff.

The second `if` statement is for adding the data to the buffer and also notice that when we add the chunks to the front of the buffer list, we don’t change the state of `reading`. And near the end, we are trying to read more if the value of chunk is not a truthy value.

Finally, the `push` and `shift` methods would simply look like these:

```javascript
push(chunk, encoding) {
  return this._readableAddChunk(chunk, encoding, false);
}

unshift(chunk, encoding) {
  return this._readableAddChunk(chunk, encoding, true);
}
```

We should also add a new `if` statement to the `_howMuchToRead` method for checking the `objectMode` state.

```javascript
_howMuchToRead(n) {
  const state = this._readableState;
  if (n <= 0 || state.length === 0)
    return 0;
  if (state.objectMode)
    return 1;
  if (Number.isNaN(n)) {
    if (state.flowing && state.length)
      return state.buffer.first().length;
    return state.length;
  }
  if (n <= state.length)
    return n;
  return 0;
}
```

And the `_fromList` method as well.

```javascript
_fromList(n) {
  const state = this._readableState;
  if (state.length === 0)
    return null;
  let ret;
  if (state.objectMode)
    ret = state.buffer.shift();
  else if (n >= state.length) {
    if (state.buffer.length === 1)
      ret = state.buffer.first();
    else
      ret = state.buffer.concat(state.length);
    state.buffer.clear();
  } else {
    ret = state.buffer.consume(n);
  }
  return ret;
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/3dfaf4bce089cb46b7ccc4380886b944ba221ec7)

The ability to end the stream is what our `readable` stream needs now.

The action of ending the `readable` stream happens by pushing a `null` value to the stream which it is be assumed that the source is done with pushing data.

Afterward, some flags would be set and an `end` event would be emitted if the essential conditions are met to end the stream.

For implementing this feature - like the others - we would start by defining some flags.

```javascript
this.ended = false;
this.endEmitted = false;
```

Then let’s create a method to be used for emitting the `end` event in the next tick.

```javascript
_endReadable() {
  const state = this._readableState;
  if (!state.endEmitted) {
    state.ended = true;
    process.nextTick(() => {
      if (!state.endEmitted && state.length === 0) {
        state.endEmitted = true;
        this.emit('end');
      }
    });
  }
}
```

we should also verify if the buffer is empty and then emit the `end` event.

The next method we want to create is supposed to be called if the pushed check is equal to `null` then it would update the `end` flag.

```javascript
_onEofChunk() {
  const state = this._readableState;
  if (state.ended) return;
  state.ended = true;
  if (state.sync)
    this._emitReadable();
}
```

Also notice that at the end we call the `_emitReadable` method, because in the Node.js documentation is mentioned that “The `readable` event will also be emitted once the end of the stream data has been reached but before the `end` event is emitted.”

Now update the final `if` statement in the `_readableAddChunk` method along with the conditions for the returned value.

```javascript
if (chunk === null) {
  state.reading = false;
  this._onEofChunk();
} else if (state.objectMode || (chunk && chunk.length > 0)) {
  if (addToFront) {
    if (state.endEmitted)
      throw new Error('cannot shift chunk after stream has already been ended');
    this._addChunk(chunk, true);
  } else if (state.ended) {
    throw new Error('cannot push chunk after stream has already been ended');
  } else {
    state.reading = false;
    this._addChunk(chunk, false);
  }
} else if (!addToFront) {
  state.reading = false;
  this._maybeReadMore();
}

return !state.ended && state.length < state.highWaterMark;
```

Next, go to the `read` method and after calling the `_howMuchToRead` method for the first time, place this part of code over there to end the stream if conditions meet.

```javascript
if (n === 0 && state.ended) {
  if (state.length === 0)
    this._endReadable();
  return null;
}
```

Afterward, update the condition of this `if` statement in that method.

```javascript
if (state.reading || state.ended)
  doRead = false;
```

And then update this `if` statement either.

```javascript
if (state.length === 0) {
  if (!state.ended)
    state.needReadable = true;
  if (nOrig !== n && state.ended)
    this._endReadable();
}
```

At this moment, go to the `_howMuchToRead` method to update the conditions of the first `if` statement.

```javascript
if (n <= 0 || (state.length === 0 && state.ended))
  return 0;
```

Then add a condition to its `return` statement.

```javascript
return state.ended ? state.length : 0;
```

We should update this part of the codes in the `_emitReadable` method.

```javascript
process.nextTick(() => {
  if (state.length || state.ended) {
    this.emit('readable');
    state.emittedReadable = false;
  }
  state.needReadable =
    !state.flowing &&
    !state.ended &&
    state.length <= state.highWaterMark;
  this._flow();
});
```

Also, don't forget about the `_maybeReadMore` method. Update the conditions of the `while` statement.

```javascript
while (!state.reading && state.length < state.highWaterMark && !state.ended)
```

We are done with this feature.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/54762b3108f4b9161148b37ba8167934fd5d9da3)

The next thing that we should take care of is to update the flowing mode of the stream when the listener of the `readable` event is removed.

For doing that we just need to override the `removeListener` method – like what we did with the `on` method.

```javascript
removeListener(eventName, callback) {
  super.removeListener(eventName, callback);

  const state = this._readableState;
  if (eventName === 'readable') {
    process.nextTick(() => {
      state.readableListening = this.listenerCount('readable') > 0;
      if (state.resumeScheduled) {
        state.flowing = true;
      } else if (this.listenerCount('data') > 0) {
        this.resume();
      } else if (!state.readableListening) {
        state.flowing = null;
      }
    });
  }
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/bd1cdfdcc30c64a722f44af06830ae556386e441)

So far so good.

Finally, it’s time to implement the most interesting method of `readable` stream which is named `pipe` method that it is the beauty of the stream module and causes the streaming data to be magical and effortless (this is what we all might have heard about it).

In fact, the `pipe` method is nothing more except a wrapper for all the operations that we need to use for reading from a `readable` stream and then write them to a `writable` stream.

As you might guess, this method takes a `writable` stream object as the parameter that we call it `dest` - also it could be a good convention here to use the name `src` instead of `this` to distinguish the source object and the destination object clearly.

```javascript
const src = this;
```

The first thing that we need to do inside of the `pipe` method is adding a listener to the `data` event on the `src` object to enable us to get the chunks for writing to the `dest` object.

But as you might remember, if the size of buffered data in the `writable` stream exceeds the `highWaterMark` then the `write` method would return `false`. This means that we should pause the flowing mode of the `readable` stream until the `writable` stream drains.

> *Note: If you hardly remember the workflow of the `writable` stream, you should review the previous chapter again before going any further.*

```javascript
pipe(dest) {
  const src = this;
  const state = this._readableState;

  const ondata = chunk => {
    const ret = dest.write(chunk);
    if (ret === false) {
      pause();
    }
  };
  src.on('data', ondata);
}
```

We only set a listener for the `drain` event when the pause function is called for the first time.

```javascript
let ondrain;
const pause = () => {
  src.pause();

  if (!ondrain) {
    ondrain = this._pipeOnDrain.bind(this);
    dest.on('drain', ondrain);
  }
};
```

The listener for the `drain` event is defined as a private method of the class.

```javascript
_pipeOnDrain() {
  const state = this._readableState;
  if (this.listenerCount('data')) {
    state.flowing = true;
    this._flow();
  }
}
```

We should also don’t forget to emit a `pipe` event on the destination.

```javascript
dest.emit('pipe', src);
```

The next listener that we should add is the `end` event on the `src`.

```javascript
const onend = () => {
  dest.end();
}
src.on('end', onend);
```

After the `end` method of the `dest` be called, then the `finish` and `close` event will be emitted on the `dest` stream, so we listen to them to unpipe them and do some cleaning.

The `unpipe` method also should be defined as a public method that can be called by the source explicitly.

Because the `pipe` method can be used for more than one destination simultaneously, thus for unpiping them we must be careful to only unpipe the determined destination not all of them.

We can handle this situation by defining an array of piped destinations with the name of `pipes` to have more control over them.

```javascript
this.pipes = [];
```

Now we use it inside of the `unpipe` method.

```javascript
unpipe(dest) {
  const state = this._readableState;
  if (!dest) {
    this.pause();
    state.pipes.forEach(dest => {
      dest.emit('unpipe');
    });
    state.pipes = [];
    return this;
  }
  for (let i = 0; i < state.pipes.length; i++) {
    if (state.pipes[i] === dest) {
      state.pipes.splice(i, 1);
      break;
    }
  }
  if (state.pipes.length === 0)
    this.pause();
  dest.emit('unpipe');
  return this;
}
```

The workflow is simple. If the destination does not be determined, we pause the `src` entirely then unpipe all of them. otherwise, if the destination is determined, then we just unpipe that specific one, but if that one was the only one, then we have to pause the `src` stream entirely.

Besides, don’t forget to push the newly piped destination to the `pipes` list. So add this code to the beginning of the `pipe` method.

```javascript
state.pipes.push(dest);
```

Also, add listeners to the `finish` and `close` events to unpipe the ended destination inside the `pipe` method.

Which event that be emitted faster would get the control of the operation of unpiping.

```javascript
const onclose = () => {
  dest.removeListener('finish', onfinish);
  src.unpipe(dest);
}
dest.once('close', onclose);
const onfinish = () => {
  dest.removeListener('close', onclose);
  src.unpipe(dest);
}
dest.once('finish', onfinish);
```

And for completing the unpipe operation, add this code to the `pipe` method.

This is just a cleaning operation.

```javascript
const onunpipe = () => {
  cleanup();
};
dest.on('unpipe', onunpipe);

const cleanup = () => {
  dest.removeListener('close', onclose);
  dest.removeListener('finish', onfinish);
  dest.removeListener('unpipe', onunpipe);
  if (ondrain) {
    dest.removeListener('drain', ondrain);
  }
  src.removeListener('end', onend);
  src.removeListener('data', ondata);
};
```

The last but not least, remember to return the `dest` object from the `pipe` method to make sure it is chainable.

```javascript
return dest;
```

That’s it. As you can see the `pipe` method is only a wrapper, but we appreciate that it makes life easier.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/cb5ab24cc99e4f72d122040340f2813c4d7ca816)

The other thing that we are going to take care of is a `_construct` method, that might be implemented by an extended class. If this method is implemented, we should ensure that some operations have a safe behavior until this method be executed. Usually, this method is used to do some kind of stuff like opening a connection for further reading data from a source or similar to that – so it might even have some asynchronous operations.

For handling this satiation, let’s define a flag named `constructed` with the default value of `true` that can be used to check whether the implemented `_construct` method is executed.

Plus, we define another property named `errored` that is supposed to be used similar to the `writable` stream, but here we don't intend to implement its functionality for handling errors. We just show you a few places in the code that we use the `errored` property to give the idea of how to use it.

> *Note: Handling errors in the stream module needs a lot of detailed investigation and description that is out of the scope of this book.*

```javascript
this.constructed = true;
this.errored = null;
```

And at the end of the `constructor` method of the `Readable` class put this part of the code.

```javascript
if (typeof this._construct === 'function') {
  const state = this._readableState;
  state.constructed = false;
  process.nextTick(() => {
    this._construct(err => {
      state.constructed = true;
      if (err)
        this.errored = err;
      if (state.needReadable)
        this._maybeReadMore();
    });
  });
}
```

We just run it in the next tick of the event loop and pass it a callback to catch a further possible error.

After checking for the error, we need to check the `needReadable` state because, in the meantime, some chunks of data might have been written to the buffer.

Then we update the condition of calling the `_read` method inside of the `read` method to avoid any pushing data to the buffer that causes losing data.

```javascript
if (state.reading || state.ended || state.errored || !state.constructed)
  doRead = false;
```

Next, we update the condition of the `if` statement in the `_maybeReadMore` method.

```javascript
if (!state.readingMore && state.constructed)
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/b1424a492b57b7147640619c916b5d9674a4916f)

The last thing that we plan to do to make our module to be more flexible, is to enable it to get some methods i.e. `_read` and `_construct` from the `options` object that is given on instantiating.

Doing that is so simple and we just need to add a bunch of `if` statements to the `constructor` of the `Readable` class.

```javascript
if (options) {
  if (typeof options.read === 'function')
    this._read = options.read;

  if (typeof options.construct === 'function')
    this._construct = options.construct;
}
```

> *Note: These methods still can be implemented through extending class. We just add another option if the user wants to only create an instance of the `readable` stream without creating an extended class.*

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/25e4a6531fd498869bdf3ce59f528a6ba72c9a67)

At this point, our `readable` stream module is complete and we can finish this chapter by implementing an example to use our module.

Here we are going to create a polyfill of the `fs.createReadStream` method by using our own `readable` module.

Now we just create a file then inside of it `require` our `readable` module along with the built-in `fs` module over there.

Next, create a new brand class to extend from our `Readable` class, and then we only need to define the required methods i.e. `_construct` and `_read`.

```javascript
class CreateReadStream extends Readable {
  constructor(filePath, options) {
    super(options);
    this._filePath = filePath;
  }
  _construct(callback) {
    fs.open(this._filePath, 'r+', (err, fd) => {
      if (err) {
        callback(err);
      } else {
        this._fd = fd;
        callback();
      }
    });
  }
  _read(n) {
    const buf = Buffer.alloc(n);
    fs.read(this._fd, buf, 0, n, null, (err, bytesRead) => {
      this.push(bytesRead > 0 ? buf.slice(0, bytesRead) : null);
    });
  }
}

const createReadStream = (filePath, options) => new CreateReadStream(filePath, options);
```

In the end, we create a function that returns a new instance of the `CreateReadStream` every time.

> *Note: Don’t forget to export the created function.*

Now you can use the `createReadStream` as an alternative to the `fs.createReadStream` method.

You can also use this function along with the `createWriteStream` from the previous example together to benefit from the `pipe` method like this:

```javascript
createReadStream('./source.txt')
  .pipe(createWriteStream('./destination.txt'));
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/fb0ba5030a4ed34ad6b8caf245b4e00a1b8a3faa)

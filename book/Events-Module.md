# Events Module

As we said before, because some part of the stream module is implemented by using an event system, so we need to have a solid understanding of how the event system works.

If you already feel conformable with the events module in Node.js and know its behavior exactly, so don’t waste time, skip this part and go forward.

But if you are not sure about it or want to know a little more about its implementation or any other reason, stay here to implement its module together.

Here we are not going to implement the whole events module. We just implement a very simplified and minimalist version of it that just contains the main essential operations. And also because of the simplicity reasons, our implementation might not be the same code as the events module in Node.js core, but be sure the behavior and the structure are the same.

The event system is nothing special except a simple managing system for executing a bunch of callbacks that gives us some advantages to manage the callbacks.

To understand the idea, first, let’s review the callback pattern which we use a lot. In the callback pattern, we send our callback to another function and expect it to be called in a specific situation.

This simple pattern works perfectly in simple situations.

But imagine when we have several callbacks and want to send them to another function then each of them is called in a specific situation or maybe some of them be called at the same point or some of them only be called once and also we be able to remove any of the callbacks to avoid be called whenever we want.

As you can see, we need a lot of features as well as a managing system to help us to manage them.

The event system is exactly what that it does this stuff for us. The internal structure of an event system is very simple.

From a perspective, this module is just a container that holds the callbacks with some methods to manage those callbacks.

Callbacks are stored there with a label that refers to it as an event name. because based on an event we would know to call which one of them.

Any label (event name) can even hold more than one callback (an array of callbacks).

```javascript
events = {
  eventName1: [cb1, cb2],
  eventName2: [cb3, cb4, cb5],
  eventName3: [cb6]
};
```

This is the main container of events module. It’s easy and simple enough.

Now we need some methods to work with this container.

Let’s build the structure of our module based on class, because it’s more understandable for most of the people.

> *Note: If you are not a fan of using class in JavaScript like me, you are free to implement it with function or any other approach you think is better.*

```javascript
// ===========================
//   Inside events.js module
// ===========================
class EventEmitter {
  constructor() {
    this._events = {};
  }
}
```

This is the starting point of our event emitter module.

Now let’s create a method for adding callbacks with the specific event name.

```javascript
addListener(eventName, listener) {
  const events = this._events;
  if (!events[eventName])
    events[eventName] = [];
  events[eventName].push(listener);
  return this;
}
```

There is nothing special about this method. It just takes an event name and a listener – which is equivalent to callback – and checks if any listener has already been defined for that event name. If so, then add this listener to the end of the list, otherwise, first, create an empty list and then add it to the list.

At the end return `this` object to make it chainable.

> *Note: The only reason that we first put the `this._events` to another variable named `events` is to have a shorter name. And because objects in JavaScript are copied by reference, so there would not be any problem with mutation because they both refer to the same address in the memory.*

Also, there is another method named `on` which is exactly as `addListener` method (both of them are the same), and we only implement it because some modules prefer to call the `on` method instead of `addListener`.

```javascript
on(eventName, listener) {
  return this.addListener(eventName, listener);
}
```

If you go with the function approach instead of class and use prototypal inheritance directly, you only need to do this:

```javascript
EventEmitter.prototype.on = EventEmitter.prototype.addListener;
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/18bf4836f8e78581123f7f2143410ed320c5132c)

Now we need to implement the second main method called `emit` that enables our module to do its primary operations as an event emitter module.

This method is in charge of calling the listeners of an event name and also passes the required arguments to them.

So to do that, this method needs to take a parameter as the event name and the rest of the parameters could be defined as the arguments of the listeners.

These are the parameters of the method:

```javascript
emit(eventName, ...args)
```

Inside of this method we need to check if that particular event name has any listener, then call them by passing the arguments to them.

Also because as the convention, the listeners should be able to access the `this` object internally, we ought to bind the `this` object on calling them – we can use the `apply` or `call` method for doing that.

```javascript
emit(eventName, ...args) {
  const events = this._events;
  if (events[eventName]) {
    const listeners = events[eventName];
    for (let i = 0; i < listeners.length; i++) {
      listeners[i].apply(this, args);
    }
  } else {
    return false;
  }
  return true;
}
```

If the method could find any listener for that event name, it would return `true`, otherwise the returned value is `false`.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/126463eefed9721c5b0d4bb098ed11aaeddd31a6)

The next method that we are going to define is a method for removing a listener from an event. But because when we add listeners to the end of the list and the list here is supposed to act sort of like a stack, so for removing a listener we need to loop through the list backward and remove the first match, and then `break` the loop.

This subject would matter only when we added a listener to an event more than once.

Also after removing the listener from that event, if no listener is left, we would `delete` that event name from the `this._events` completely.

```javascript
removeListener(eventName, listener) {
  const events = this._events;
  if (events[eventName]) {
    for (let i = events[eventName].length - 1; i >= 0; i--) {
      if (events[eventName][i] === listener) {
        events[eventName].splice(i, 1);
        break;
      }
    }
    if (events[eventName].length === 0)
      delete events[eventName];
  }
  return this;
}
```

Besides this method, we might need another method named `off` that does the same thing as `removeListener`. So we do what we did with the `on` method again.

```javascript
off(eventName, listener) {
  return this.removeListener(eventName, listener);
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/bb0918b1b27f90e197883f0adec0ed12a736d966)


Furthermore, sometimes we need a method to remove an event with all of its listeners or even remove all the events.

For doing that, we create a method named `removeAllListeners`.

```javascript
removeAllListeners(eventName) {
  if (eventName === undefined) {
    this._events = {};
    return this;
  }
  delete this._events[eventName];
  return this;
}
```

If we pass an event name to it, only that event would be removed, otherwise all the events would be removed.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/31306c78767c0cec505cf17fd8add4cdb9ef60d0)

The built-in events module also has another mothed called `once` which is less used but very useful.

The responsibility of this method is simple but its implementation is a little tricky.

This method is supposed to act like the `addListener` or `on` method (which both are the same), but it has to guarantee that the listener is added by this method is called at most once.

In order to achieve this functionality, we need an approach that gives the listener the ability to remove its self from the `this._events` object.

Because we can’t modify the body of the listener functions and also a function object doesn’t have any magical property that could be run before or after itself, the common technique is to wrap the listener function inside another function which could give us the control of before and after running the listener.

Now we need to create a method to wrap the listener. We name this method `_onceWrap` that takes the event name in addition to the listener because we would need both of them at the time of removing the listener.

The `_onceWrap` method is in charge to return a new function that some data being bound to it.

The data that are necessary to be bound include:

- `this` object: To access the `removeListener` method and be used to be bound to the listener as its own `this` object.
- `eventName`: To be used for removing the corresponding listener.
- `listener`: To be called on emitting event.
- And also the wrapper function itself: To be used as the reference to 
remove the listener. Because this wrapper function is supposed to be placed in the `this._events` object instead of the listener.

Describing is enough, let’s jump to the code to see how we are going to implement this algorithm.

First of all, this is our final expectation of the `once` method:

```javascript
once(eventName, listener) {
  return this.addListener(eventName, this._onceWrap(eventName, listener));
}
```

This is going to call the `addListener` method (just like the `on` method), but as we discussed, instead of sending the listener directly, we need a wrapper function.

The body of the wrapper function that is going to be returned from the `_onceWrapper` is like this:

```javascript
_onceWrapper({eventName, listener, wrapFn}, ...args) {
  this.removeListener(eventName, wrapFn);
  listener.apply(this, args);
}
```

It just takes some necessary data - that we talked about before, and these are supposed to be bound to this function – and also the arguments to act like a real listener.

The `wrapFn` variable is a pointer to the `_onceWrapper` method, and as you can see it is used to remove itself as a listener.

Finally, after removing itself, it simply just executes the listener as normal.

Now we know our expectation of the `_onceWrap` function and also the body of the wrapper function itself, the only thing that we need is to implement the `_onceWrap` function to meet our expectation.

```javascript
_onceWrap(eventName, listener) {
  const state = { eventName, listener, wrapFn: null };
  state.wrapFn = this._onceWrapper.bind(this, state);
  return state.wrapFn;
}
```

The body of this method is straightforward, and the only tricky part might be the `null` value for the `wrapFn` property.

Because we needed to bind a pointer of a function to itself, and as we all know objects in JavaScript are reference types, so first we created an object named `state` with a property named `wrapFn` and bound its address to the function. After binding the function, we get the address of the bound function and put it into the corresponding property of the object that we already bound it.

This action causes the value of the property `wrapFn` to be changed in the bound object as well.

And at the end, we also need to return the address of the wrapper and it’s done.

But before going further, let’s review our code and investigate the workflow of the `once` method to make sure that everything works as expected.

When we set a listener via the `once` method, the list of the listeners for a particular event would be eventually something like this:

<p align="center">
  <img alt="listener list" src="/book/assets/figure-07_listener-list.png" />
</p>

Till here everything is normal, but once the `emit` method wants to loop through the list and call each one by one when it runs the listener in index 1 (which is our listener that has been set via the `once` method), the listener would run the `removeListener` method to remove itself from this list.

As we remember, the `removeListener` uses the `splice` method to remove an item from the array. And again because arrays are objects and they are reference types, so removing the listener by the `removeListener` method means removing it from the array above as well.

The problem is here when the counter of the loop inside the `emit` method reach index 1 and call its listener and its listener removes itself from the list, and the entire list from that point to the end would be shifted one index to the left, afterward, the final list will end up like this:

<p align="center">
  <img alt="listener list" src="/book/assets/figure-08_listener-list.png" />
</p>

At this point, the loop has no idea that it has been modified and thinks it’s done with index 1 and goes for index 2.

Thus, as you noticed, we lost `listener3`.

There are some approaches to handle these kinds of situations, and one of them is to clone the list that might be modified in the future. It’s so simple and easy to implement.

First, let’s create a helper method to clone an array.

```javascript
_arrayClone(arr) {
  return arr.map(v => v);
}
```

We just chose the easiest way.

And then replace this code in the `emit` method:

```javascript
const listeners = this._arrayClone(events[eventName]);
```

That’s it. We’re done with this problem.

Let’s continue and check the other methods to see whether they work well.

As you may have noticed by now, in order for our `removeListener` method to work properly, it needs a listener – which is a pointer to a function – and then loops through the list of the existing listeners and compare their address.

But when we use the `once` method to set a listener, we don’t store the listener directly in the list, we first wrap it inside another function and store the wrapper function in the list. So the listener address that is passed by the user would never be found in the list. And also the user doesn’t access to the wrapper function address either.

To overcome this issue, we ought to find a way to store the address of the listener along with the wrapper function inside of the list.

The technique we are going to use is so simple. Because function in JavaScript is an object, so we can also treat it as an object and set a property on it to store the listener address.

Inside the `_onceWrap` method, just put this line of code before returning the wrapper address:

```javascript
state.wrapFn.listener = listener;
```

And then go to the `removeListener` method and update the condition of the `if` statement to this:

```javascript
if (events[eventName][i] === listener || events[eventName][i].listener === listener)
```

This means that first, we check the address of the listener inside the list directly, if it doesn't match then check the `listener` property of it.

We are done with this problem as well.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/275f8e13c5a6d73b8c97dffb3a8f8cf53be35b45)

Finally, to wrap up our events module, let’s add another simple method to count the number of listeners of the given event.

```javascript
listenerCount(eventName) {
  const events = this._events;
  if (events[eventName])
    return events[eventName].length
  return 0;
}
```

Our events module is complete now and it is ready for our further use cases.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/7f17b96b3f78e66b126b84e73e83dd4df3ad1fd3)

The main purpose of implementing the events module is to see that everything inside this module works very simple and **synchronously** - contrary to what some people believe.

So be careful about the order of adding listeners and emitting them. Because if you set a listener after it’s been emitted and that part of your code be synchronous, your listener won’t be called, and to handle this circumstance you need to emit the events in the next tick of the `event loop` to assure that the listeners have been set earlier.

To fully grasp its workflow, you should just spend more time playing with it.

At the time of implementing the stream module, you are completely free to choose to use this events module that we implemented together or the built-in one. You can also switch between them in the future, so don’t be hard about it.

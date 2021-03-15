# Duplex Stream

At this moment that we have both the `writable` and `readable` streams, it’s time to combine them to create a new type of stream which is going to be called `duplex` stream.

As we talked about the `duplex` stream in the early chapter, we said that this stream is nothing more than a container that just simply holds a `writable` stream object in addition to a `readable` stream object completely separated and doesn’t have any more extra functionality.

<p align="center">
  <img alt="duplex stream" src="/book/assets/figure-05_duplex-stream.png" />
</p>

The all the things that you can do with a `duplex` stream, you can do them without it by using a `readable` stream object along with a `writable` stream object.

Its implementation is also pretty simple and straightforward.

We start by creating a file for this module and then first things first, we `require` both of the implemented stream modules (`readable` and `writable`) here.

```javascript
const Readable = require('./readable');
const Writable = require('./writable');
```

Before going any further, let’s pause for a moment to see what exactly we want and how we suppose to implement that.

Because the `duplex` stream object is going to be a combination of those other stream objects, thus it means that we need an object that can access through it to all of their methods and properties.

This sounds like multiple inheritances that other programming languages support, but JavaScript does not!

But no need to worry. Because we already know how inheritance works in JavaScript, so we can do that ourselves.

> *Note: If the prototypal inheritance is still a bit vague for you, we suggest taking a look at this [MDN web page](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain) before continuing.*

Here we can still benefit from the single inheritance, so we create a `Duplex` class and extend from one of those streams (which we choose `readable` stream here to extend from).

```javascript
class Duplex extends Readable {
  constructor(options = {}) {
    super(options);
  }
}
```

Now we just need to find a way to inherit from the `writable` stream as well.

For having the properties of a `writable` stream object, we can instantiate a new object from the `Writable` class and mix it with `this` object in the `constructor` of the `Duplex` class.

```javascript
constructor(options = {}) {
  super(options);
  Object.assign(this, new Writable(options));
}
```

As we know, class member functions are simply held as the properties of the `Class.prototype` object, and the `Object.assign` method doesn’t go deep to the `[[prototype]]` of an object.

So we need to assign the methods of the `Writable.prototype` object to the `Duplex.prototype` object by looping through them.

```javascript
for (const method of Object.getOwnPropertyNames(Writable.prototype)) {
  if (method !== 'constructor' && !Duplex.prototype[method])
    Duplex.prototype[method] = Writable.prototype[method];
}
```

Just be careful not to overwrite the `constructor` and any other existing method.

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/a88874e6784ffe386eab5adfb62ec84b3c390c52)

At this point, our `duplex` stream module is complete and we can finish this short chapter by implementing an example to use our module.

Here we are not going to polyfill any existing methods. We just create a very simple and maybe useless module that is the combination of two previous examples.

Now we just create a file then inside of it `require` our `duplex` module along with the built-in `fs` module over there.

Next, create a new brand class to extend from our `Duplex` class, and then we only need to define the required methods i.e. `_construct`, `_read`, `_write` and `_destroy`.

```javascript
class CreateDuplexStream extends Duplex {
  constructor(readFilePath, writeFilePath, options) {
    super(options);
    this._readFilePath = readFilePath;
    this._writeFilePath = writeFilePath;
  }
  _construct(callback) {
    fs.open(this._writeFilePath, 'w', (err, file) => {
      if (err) {
        callback(err);
      } else {
        this._writefd = file;
        fs.open(this._readFilePath, 'r+', (err, file) => {
          if (err) {
            callback(err);
          } else {
            this._readfd = file;
            callback();
          }
        });
      }
    });
  }
  _read(n) {
    const buf = Buffer.alloc(n);
    fs.read(this._readfd, buf, 0, n, null, (err, bytesRead) => {
      this.push(bytesRead > 0 ? buf.slice(0, bytesRead) : null);
    });
  }
  _write(chunk, encoding, next) {
    fs.write(this._writefd, chunk, next);
  }
  _destroy(err, callback) {
    fs.close(this._writefd, er => callback(er || err));
  }
}

const createDuplexStream = (readFilePath, writeFilePath, options) => new CreateDuplexStream(readFilePath, writeFilePath, options);
```

In the end, we create a function that returns a new instance of the `CreateDuplexStream` every time.

> *Note: Don’t forget to export the created function.*

Also, you can test this function simply by piping it to itself.

```javascript
const duplexStream = createDuplexStream('./source.txt', './destination.txt');

duplexStream
  .pipe(duplexStream);
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/f746d5f94a79edb29a89172f4bfc0b306af57c42)

<div align="center">
  <a title="Previous Chapter | Duplex Stream" href="/book/Duplex-Stream.md"> < prev </a>
  |
  <a title="Go to the Table of Contents" href="/book#table-of-contents"> ToC </a>
  |
  <a title="Next Chapter | Conclusion" href="/book/Conclusion.md"> next > </a>
</div>

***

# Transform Stream

In the previous chapter, as we examined the `duplex` stream module, we figured out that this stream is just a wrapper of the combination of both `readable` and `writable` stream objects that hold them **separately**.

The `transform` stream is another type of stream that is a subclass of the `duplex` stream. This stream only comes to create a bridge between those two separated streams inside of the `duplex` stream.

<p align="center">
  <img alt="transform stream" src="/book/assets/figure-06_transform-stream.png" />
</p>

The main role of the `transform` stream is to act as a transferor in the pipeline.

This stream doesn’t allow the destination and source to implement the `_write` and `_read` methods. Therefore, it can’t be used as a starter or even a finisher in a pipeline. It only can be placed in the middle of a pipeline and also it is the only one that can be used to be in the middle, but it plays very well in that position.

The `transform` stream provides a method named `_transform` which is the heart of the bridge that is implemented by the user to control every single chunk of transferred data.

Now we have a bird’s eye view of the `transform` stream. Let’s start the implementation by building a class-based structure - as usual - that is extended from the `Duplex` class. Don’t forget to `require` the `duplex` module.

```javascript
class Transform extends Duplex {
  constructor(options) {
    super(options);
  }
}
```

Also, it’s good that we define the `_transform` method to throw an error that forces the user to implement it.

```javascript
_transform() {
  throw new Error('_transform method must be implemented!');
}
```

The workflow that we intend to implement for this stream is pretty simple and we just need to define the `_write` method to get the written chunk from the `writable` stream and pass it to the `_transform` method. We also need to give the `_transform` method a callback that sends us the chunk – which could be manipulated – and finally push this chunk to the `readable` stream through its `push` method.

Moreover, we should store the `next` method that is passed to the `_write` method (you might remember it as the `_onwrite` method that is passed to the `_write` method as a callback in the third argument) in a property to be called in the `_read` method – which means that the chunk is consumed, so send the next one.

Add this code to the `constructor` method to store the callback method that is passed to the `_write` method.

```javascript
this._callback = null;
```

Next, implement the `_write` method to do what we talked about.

```javascript
_write(chunk, encoding, callback) {
  this._transform(chunk, encoding, (err, val) => {
    if (err) {
      callback(err);
      return;
    }
    if (val !== null)
      this.push(val);
    this._callback = callback;
  });
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/20e8f4adcdbecdbbd67761ec562ec0ecabf7bbd3)

Now we can create the `_read` method to call that callback.

```javascript
_read() {
  if (this._callback) {
    const callback = this._callback;
    this._callback = null;
    callback();
  }
}
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/973d10598dcb53c5fdc8b82b3abee4a1aa74a72d)

Currently, our `transform` stream module is complete and this chapter is like the previous chapter is very short. Now we can finish it by implementing an example to use our module.

Here we are going to create a very simple transformer that just makes the transferred chunks of data be upper case.

Now we just create a file then inside of it `require` our `transform` module.

Next, create a new brand class to extend from our `Transform` class, and then we only need to define the `_transform` method to do its job.

```javascript
class CreateTransformStream extends Transform {
  constructor(options) {
    super(options);
  }

  _transform(data, encoding, callback) {
    callback(null, data.toString().toUpperCase());
  }
}

const createTransformStream = options => new CreateTransformStream(options);
```

In the end, we create a function that returns a new instance of the `CreateTransformStream` every time.

> *Note: Don’t forget to export the created function.*

Also, you can test this function simply by placing it in the middle of a pipeline. You can use a `duplex` stream or a `readable` stream along with a `writable` stream to create the pipeline.

```javascript
duplexStream
  .pipe(transformStream)
  .pipe(duplexStream);
```

Or

```javascript
createReadStream('./source.txt')
  .pipe(transformStream)
  .pipe(createWriteStream('./destination.txt'));
```

> [Source code](https://github.com/Babak-Gholamzadeh/stream-module/tree/8115715db05c98227144d9d4be36656244851bcb)

***

<div align="center">
  <a title="Previous Chapter | Duplex Stream" href="/book/Duplex-Stream.md"> < prev </a>
  |
  <a title="Go to the Table of Contents" href="/book#table-of-contents"> ToC </a>
  |
  <a title="Next Chapter | Conclusion" href="/book/Conclusion.md"> next > </a>
</div>

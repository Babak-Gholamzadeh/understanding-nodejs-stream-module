# What is Stream?

Accessing some data without streaming is just like that we have to access the whole data at once, which has the disadvantage of time and space consuming.

<p align="center">
  <img alt="access data without stream" src="/assets/figure-01_access-data-without-stream.png" />
</p>

But when stream comes into play, it gives us the advantage of accessing the data, chunk by chunk, and each chunk at a time.

<p align="center">
  <img alt="access data with stream" src="/assets/figure-02_access-data-with-stream.png" />
</p>

The stream module has implemented this mechanism through 2 main submodules that can define a **`readable`** or **`writable`** stream.

Once we want to read some chunks of data from a source, we need a `readable` stream object.

<p align="center">
  <img alt="readable stream" src="/assets/figure-03_readable-stream.png" />
</p>

And when we want to write some chunks of data to a destination, we need a `writable` stream object.

<p align="center">
  <img alt="writable stream" src="/assets/figure-04_writable-stream.png" />
</p>

Also when we wrap these two types of stream objects inside a new object, we would create a **`duplex`** stream. Which simply contains these two objects inside of itself separately.

<p align="center">
  <img alt="duplex stream" src="/assets/figure-05_duplex-stream.png" />
</p>

And finally, if we create a method inside a `duplex` stream that acts as a bridge between the `writable` and the `readable` stream, the new brand created object is called **`transform`** stream.

<p align="center">
  <img alt="transform stream" src="/assets/figure-06_transform-stream.png" />
</p>

Thus, a stream object is simply just a container with a bunch of methods that help us to store and retrieve the data. Also, be aware that some part of this workflow is built on the event system.

If these definitions seem a little vague to you, don’t worry, because that’s the main reason that we want to implement this module from scratch to make everything clear for ourselves.

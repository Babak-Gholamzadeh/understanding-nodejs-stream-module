<div align="center">
  <span> < prev </span>
  |
  <a title="Go to the Table of Contents" href="/book#table-of-contents"> ToC </a>
  |
  <a title="Next Chapter | What is Stream?" href="/book/What-is-Stream.md"> next > </a>
</div>

***

# Preface

As you came here, you might have already heard about the stream or have some experience of using this module in Node.js (directly or indirectly). You also might have heard that there are some other modules like `http`, `fs`, `zlib`, `crypto`, `TCP sockets`, `process.stdout, process.stdin, 
process.stderr` that use `stream` module internally.

In this book, we are not discussing other modules that use stream or addressing any use cases of streaming data, or even talking about how to use its API.

Here we are going to take a deep dive into the `stream` module itself to understand its basic workflow by implementing a simplified version of this module step by step and investigate each step together to see the reasons behind each property and method.

Also because we intend our final implementation to be similar to the built-in `stream` module in the Node.js core, so it’s tried to use the same structure and name convention for the implementation (but still there might be some differences). In addition to that, we don’t intend to have a very optimized implementation here. But of course, you can do the optimization on your own if you wish.

In this manner, it will also make you familiar with the source code of Node.js core that might even help you to understand the parts of the code that aren’t covered here or motivate you to become a future contributor in Node.js project to aid to improve it.

Before starting, keep in mind that for the reason of avoiding ambiguity, we even implement some side modules like `EventEmitter` and `BufferList` (which you will see later) from scratch that remain no magical or black-boxing behavior. But feel free to skip any part of the book that might confuse you or you can only skim them to just keep yourself on track. Do not let too much detail distract you from having an abstract view.

> *Note: Throughout the book you will see that after the implementation of each small feature is finished, there is a link labeled **Source code**, which means that you can access the source code of the implementation till that point in a GitHub repository.*

***

<div align="center">
  <span> < prev </span>
  |
  <a title="Go to the Table of Contents" href="/book#table-of-contents"> ToC </a>
  |
  <a title="Next Chapter | What is Stream?" href="/book/What-is-Stream.md"> next > </a>
</div>

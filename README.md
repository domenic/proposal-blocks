# JavaScript "blöcks" proposal

This is a proposal to add a new syntactic construct, called (for now) "blöcks", to the JavaScript language. It is meant primarily to help with use cases around parsing and serializing blocks of code for use in other contexts, especially in other threads.

The name "blöcks" is certainly temporary; we picked it for now because we keep calling these things "blocks", but we realize that that word already has meaning in JavaScript, so we shouldn't overlap with it. Hopefully we'll figure something better out soon.

## Motivation

Running multi-threaded code in other languages is easy: it is often a matter of just passing a function to a threaded scheduler. Some examples:

```c++
// C++
std::thread my_thread([](param) {
  // ... lots and lots of work ...
}, arg);

my_thread.join();
```

```csharp
// C#
var result = await Task.Run(() => {
  // ... lots and lots of work ...
});
```

In JavaScript this is not so easy. The web provides multi-threading through workers, which require separate source files and verbose message-passing. Libraries can help paper over some of the awkwardness here ([see below](#libraries-accepting-functions)), but in our opinion fall short without language integration.

We would like to enable similarly frictionless parallelism on the web, and we think extending JavaScript's syntax would help to do so, while also giving benefits for other environments like Node.js with similar needs.

## The basic idea

We introduce a new syntactic construct, the "blöck", which is meant to encapsulate a syntactic chunk of code isolated from its surroundings. We propose use the `{| ... blöck contents ... |}` syntax for this:

```js
const blöck = {|
  // ... code goes here ...
|};
```

Code inside the blöck is parsed, and in doing so, the implementation checks that the code does not reference any bindings from outside the blöck. (At least by default; [see below](#variable-capture).) Thus blöcks are very different from normal blocks (i.e. those denoted with `{ ... }`) and from functions.

Another important difference is that the result of this syntax, i.e. the value in the `blöck` variable above, is an opaque handle. This handle can be later reified into a function, via its `reify()` method. Importantly, it can also be transferred across realms or agents (at least in environments like the web which support the notion of transferrable objects), and then reified on the other side.

Finally, we tentatively think that the code inside a blöck should have the syntax of an async function body. Or perhaps of a module body, if [top level `await`](https://github.com/MylesBorins/proposal-top-level-await/) happens? The important part is that it be able to use `await`, and that it have some way of `return`ing a value to the main thread.

### Example usage

Consider wanting to fetch a file at a known URL, transform its contents, and return the result to the main thread. This could be done with the basic blöck syntax as follows:

```js
const result = await worker({|
  const res = await fetch("people.json");
  const json = await res.json();

  return json[2].firstName;
|});
```

Here `worker()` is a function, provided by a library or by the browser, to transfer the blöck, run it in another thread, and pass back the result. Expand the below to see an example implementation:

<details>
<summary>Example <code>worker()</code> implementation</summary>

```js
// worker-library.js
const theWorker = new Worker("blöck-receiver.js");

function worker(blöck) {
  return new Promise((resolve, reject) => {
    const channel = new MessageChannel();
    theWorker.postMessage(
      { blöck, port: channel.port2 },
      [blöck, channel.port2] // transferList
    );
    channel.port1.onmessage = e => {
      if (e.data.fulfilled) {
        resolve(e.data.value);
      } else {
        reject(e.data.value);
      }
  })
}
```

```js
///blöck-receiver.js
self.onmessage = e => {
  const { blöck, port } = e.data;
  const fn = blöck.reify();
  fn().then(
    v => port.postMessage({ fulfilled: true, value: v }),
    r => port.postMessage({ fulfilled: false, value: r })
  );
}
```
</details>

## Further ideas, by example

On top of this basic construct and syntax, we propose a number of smaller features, which are best illustrated by example. Remember that our overriding goal is to make it easy and ergonomic to move code off the main thread.

### Tagged blöcks

Our above example is a bit punctuation-heavy. Could we omit the parenthesis, and just write this instead?

```js
const result = await worker{|
  const res = await fetch("people.json");
  const json = await res.json();

  return json[2].firstName;
|};
```

The idea is that `x{| ... |}` desugars to `x({| ... |})`. This might feel more worthwhile when you see the next extension:

### Variable capture

One advantage other languages have over our blöck syntax so far is that they make it very easy to capture variables in the closure passed to the thread/task-factories. This is largely possible due to those languages' shared-memory nature, which we don't want to bring to blöcks. But we think we can get pretty close, with just a bit more syntax.

Consider code like the above, but it wants to determine the endpoint from data available only on the main thread, instead of hard-coding this. With blöcks so far, this is not so easy, as blocks are unable to refer to bindings from outside the blöck:

```js
const endpoint = document.querySelector("#endpoint").value;

const result = await worker{|
  const res = await fetch(????); // We want to use endpoint, but can't!
  const json = await res.json();

  return json[2].firstName;
|};
```

We could make this work using the basic blöcks syntax with something like the following:

```js
const resultComputer = workerFunction{|
  const [endpoint] = arguments;
  const res = await fetch(endpoint);
  const json = await res.json();

  return json[2].firstName;
|};

const result = await resultComputer(endpoint);
```

where `workerFunction()` is similar to `worker()`, but instead of returning a promise like `worker()` does, it returns a promise-returning function that clones its arguments.

This is repetitive and annoying to write. Instead, we propose the following syntax, where you can explicitly declare a list of bindings to be "captured" by the blöck:

```js
const result = await worker<endpoint>{|
  const res = await fetch(endpoint); // OK to use now
  const json = await res.json();

  return json[2].firstName;
|};
```

The semantics here are as follows:

* Unlike `worker{| ... |}`, which is just sugar for a call to `worker({| ... |})`, this becomes a call to `worker({| ... |}, { endpoint })`. That is, we pass both the binding name and the binding's value to the `worker()` function.
* The created blöck handle is explicitly noted as being "incomplete"; it cannot be reified into a function without supplying a value for the missing `endpoint` binding. This is done by calling `blöck.reify({ endpoint })`.

<details>
<summary>See the implementation of <code>worker()</code> that allows capture in this way</summary>

Changed lines are highlighted with `// (*)` comments

```js
// worker-library.js
const theWorker = new Worker("block-receiver.js");

function worker(blöck, bindingValues) {
  return new Promise((resolve, reject) => {
    const channel = new MessageChannel();
    theWorker.postMessage(
      { blöck, port: channel.port2, bindingValues }, // (*)
      [blöck, channel.port2] // transferList
    );
    channel.port1.onmessage = e => {
      if (e.data.fulfilled) {
        resolve(e.data.value);
      } else {
        reject(e.data.value);
      }
  })
}
```

```js
///block-receiver.js
self.onmessage = e => {
  const { blöck, port, bindingValues } = e.data; // (*)
  const fn = blöck.reify(bindingValues);         // (*)
  fn().then(
    v => port.postMessage({ fulfilled: true, value: v }),
    r => port.postMessage({ fulfilled: false, value: r })
  );
}
```
</details>

An alternative would be to not require declaring the capture outside the block, but instead provide special syntax inside the block for notating captured variables. For example:

```js
const result = await worker{|
  const res = await fetch(${endpoint}); // Special marker syntax
  const json = await res.json();

  return json[2].firstName;
|};
```

This would have the same semantics, with the list of missing bindings assembled by the parser scanning the contents of the blöck.

## Open questions

Here are some things we aren't sure about:

* What's a real name for these things, i.e. not blöcks?
* Is it OK to bake in the async function body as the blessed source code type for blöcks? Should this be configurable somehow? Async generators would allow an interesting case of multiple outputs back to the calling thread.
* Relatedly, is it OK that the way to use libraries inside of blöcks is with `import()`? Should we encourage/allow statically-analyzable `import` statements somehow? This seems hard without breaking the idea of these things reifying into functions.
* Can we make this work with existing web APIs that expect whole source files, such as `new Worker()` or `CSS.paintWorklet.load()`?

## Alternatives considered

There are many concepts already in this space. Here we give a brief survey of them, and note why we think proposing blöck syntax is worthwhile over using these existing alternatives.

### Template literals

Why are blöcks better than just embedding source code in a template literal? E.g.

```js
const thePrime = await worker({ endpoint }, `
  // Use endpoint inside here
`);
```

A few reasons:

* Template literals are treated as strings by the JavaScript engine and by the tooling ecosystem. This treatment is generally quite different from code.
* The contents of these literals cannot be validated during parsing, e.g. to check for syntax errors, or to ensure no incorrect variable bindings are used.
* Template literals cannot be nested. So, if your off-main-thread work wanted to use a multiline string, you're out of luck. Because blöcks have proper opening and closing sigils, they can be nested arbitrarily.
* This requires first parsing the literals as strings, and then reassembling them into dynamically-created functions at their destination, which is not performant and runs afoul of anti-eval policies.

### Libraries accepting functions

A few existing libraries, such as [Clooney](https://github.com/GoogleChromeLabs/clooney) or [greenlet](https://github.com/developit/greenlet), allow syntax such as the following:

```js
const thePrime = await greenlet(async endpoint => {
  // Use endpoint inside here
})(endpoint);
```

This approach is not quite as bad as using template literals, but it has similar drawbacks:

* The JavaScript engine, and tooling ecosystem, has no way of knowing that these functions are not real closures; for example, they allow using closed-over variables.
* The code gets parsed as code, then serialized via `fn.toString()`, then sent to its destination, then dynamically assembled and re-parsed as code again. This is even worse than the template literal case.
* The syntax is less pleasant than it could be, further increasing friction.

### Tasklets

[Tasklets](https://discourse.wicg.io/t/proposal-tasklets/2199) were a WICG proposal that tried to enable off-thread computing with a strong focus on ergonomics. They involved creating a module file, which was loaded into the "tasklet" (mini-worker), and exposed an interface via its exports that became a bunch of async function proxies on the other side.

Tasklets got considerable pushback from Microsoft and Mozilla. The main concerns were:

* No clear reason why tasklets were different from workers or weren’t using workers
* Tasklets still required the off-thread code to live in a separate file

In general we think the ergonomics improvements of tasklets help when you have a clear separation between classes or services that can live in another thread, and communicate back and forth via well-defined interfaces. However, they don't make it simple enough to move smaller chunks of work off the main thread, in the way we see as prevalent in other languages.

## Acknowledgments

This proposal is a collaboration between [@domenic](https://github.com/domenic) and [@surma](https://github.com/surma).

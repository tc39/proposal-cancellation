> NOTE: This was the initial stage 0 proposal presented to TC39. However, the proposal did not advance in this form.
> The following is retained for future discussion.

# Cancellation API (initial stage 0 proposal)

This proposal defines an approach to user-controlled cancellation of asynchronous operations
through the adoption of a set of native platform objects.

## Status

**Stage:** 0  
**Champion:** Ron Buckton (@rbuckton), Brian Terlson (@bterlson)

_For more information see the [TC39 proposal process](https://tc39.github.io/process-document/)._

## Authors

* Ron Buckton (@rbuckton)

# Motivations

* A clear and consistent approach to cancelling asynchronous operations:
  * Asynchronous functions or [iterators](https://github.com/tc39/proposal-async-iteration)
  * [Fetching](https://fetch.spec.whatwg.org/#fetch-api) remote resources (HTTP, I/O, etc.)
  * Interacting with background tasks (Web Workers, forked processes, etc.)
  * Long-running operations ([animations](https://w3c.github.io/web-animations/), etc.)
* A general-purpose coordination primitive with many use cases:
  * Synchronous observation (e.g. in a game loop)
  * Asynchronous observation (e.g. aborting an XMLHttpRequest, stopping an animation)
  * Easy to use in async functions.
  * Scale from simple single source -> token relationships to [complex cancellation graphs](#complex-cancellation-graphs).
* A single shared API that is reusable in multiple host environments (Browser, NodeJS, IoT/embedded, etc.).

# Prior Art

* [Cancellation in Managed Threads](https://msdn.microsoft.com/en-us/library/dd997364(v=vs.110)) in the .NET Framework

# Proposal

```ts
class CancellationTokenSource {
  constructor(linkedTokens?: Iterable<CancellationToken>);
  readonly token: CancellationToken;
  cancel(): void;
  close(): void;
}

class CancellationToken {
  static readonly none: CancellationToken;
  static readonly canceled: CancellationToken;
  constructor(source: CancellationTokenSource);
  readonly cancellationRequested: boolean;
  readonly canBeCanceled: boolean;
  throwIfCancellationRequested(): void;
  register(callback: () => void): { unregister(): void; };
}
```

Cancellation consists of two main components:

* `CancellationTokenSource` - Created by the caller of an asynchronous operation, a *CancellationTokenSource*
is responsible for signaling cancellation.
* `CancellationToken` - Each *CancellationTokenSource* is entangled with a single *CancellationToken*
which is supplied to an asynchronous operation by the caller. A *CancellationToken* can only be
used to observe a cancellation signal.

## Architecture
The following are some architectural observations provided by **Dean Tribble** on the [es-discuss mailing list](https://mail.mozilla.org/pipermail/es-discuss/2015-March/041887.html):

> *Cancel requests, not results*  
> Promises are like object references for async; any particular promise might
> be returned or passed to more than one client. Usually, programmers would
> be surprised if a returned or passed in reference just got ripped out from
> under them *by another client*. this is especially obvious when considering
> a library that gets a promise passed into it. Using "cancel" on the promise
> is like having delete on object references; it's dangerous to use, and
> unreliable to have used by others.
>
> *Cancellation is heterogeneous*  
> It can be misleading to think about canceling a single activity. In most
> systems, when cancellation happens, many unrelated tasks may need to be
> cancelled for the same reason. For example, if a user hits a stop button on
> a large incremental query after they see the first few results, what should
> happen?
>
> - the async fetch of more query results should be terminated and the
> connection closed
> - background computation to process the remote results into renderable
> form should be stopped
> - rendering of not-yet rendered content should be stopped. this might
> include retrieval of secondary content for the items no longer of interest
> (e.g., album covers for the songs found by a complicated content search)
> - the animation of "loading more" should be stopped, and should be
> replaced with "user cancelled"
> - etc.
>
> Some of these are different levels of abstraction, and for any non-trivial
> application, there isn't a single piece of code that can know to terminate
> all these activities. This kind of system also requires that cancellation
> support is consistent across many very different types of components. But
> if each activity takes a cancellationToken, in the above example, they just
> get passed the one that would be cancelled if the user hits stop and the
> right thing happens.
>
> *Cancellation should be smart*  
> Libraries can and should be smart about how they cancel. In the case of an
> async query, once the result of a query from the server has come back, it
> may make sense to finish parsing and caching it rather than just
> reflexively discarding it. In the case of a brokerage system, for example,
> the round trip to the servers to get recent data is the expensive part.
> Once that's been kicked off and a result is coming back, having it
> available in a local cache in case the user asks again is efficient. If the
> application spawned another worker, it may be more efficient to let the
> worker complete (so that you can reuse it) rather than abruptly terminate
> it (requiring discarding of the running worker and cached state).
>
> *Cancellation is a race*  
> In an async system, new activities may be getting continuously scheduled by
> asks that are themselves scheduled but not currently running. The act of
> cancelling needs to run in this environment. When cancel starts, you can
> think of it as a signal racing out to catch up with all the computations
> launched to achieve the now-cancelled objective. Some of those may choose
> to complete (see the caching example above). Some may potentially keep
> launching more work before that work itself gets signaled (yeah it's a bug
> but people write buggy code). In an async system, cancellation is not
> prompt. Thus, it's infeasible to ask "has cancellation finished?" because
> that's not a well defined state. Indeed, there can be code scheduled that
> should and does not get cancelled (e.g., the result processor for a pub/sub
> system), but that schedules work that will be cancelled (parse the
> publication of an update to the now-cancelled query).
>
> *Cancellation is "don't care"*  
> Because smart cancellation sometimes doesn't stop anything and in an async
> environment, cancellation is racing with progress, it is at most "best
> efforts". When a set of computations are cancelled, the party canceling the
> activities is saying "I no longer care whether this completes". That is
> importantly different from saying "I want to prevent this from completing".
> The former is broadly usable resource reduction. The latter is only
> usefully achieved in systems with expensive engineering around atomicity
> and transactions. It was amazing how much simpler cancellation logic
> becomes when it's "don't care".
>
> *Cancellation requires separation of concerns*  
> In the pattern where more than one thing gets cancelled, the source of the
> cancellation is rarely one of the things to be cancelled. It would be a
> surprise if a library called for a cancellable activity (load this image)
> cancelled an unrelated server query just because they cared about the same
> cancellation event. I find it interesting that the separation between
> cancellation token and cancellation source mirrors that separation between
> a promise and it's resolver.
>
> *Cancellation recovery is transient*  
> As a task progresses, the cleanup action may change. In the example above,
> if the data table requests more results upon scrolling, it's cancellation
> behavior when there's an outstanding query for more data is likely to be
> quite different than when it's got everything it needs displayed for the
> current page. That's the reason why the "register" method returns a
> capability to unregister the action.

## Observing Cancellation Requests
A request for cancellation may be observed either synchronously or asynchronously. To observe a
cancellation request synchronously you may either check the `token.cancellationRequested` property, or
invoke the `token.throwIfCancellationRequested()` method. To observe a cancellation request
asynchronously, you may register a callback using the `token.register()` method which returns an object
that can be used to unregister the callback once you no longer need to observe the signal.

## Finalizing a Cancellation Request
When you invoke `source.cancel()`, it evaluates each registered callback with an empty stack. Once all 
registered callbacks have run to completion, the method will return. If any registered callback results in 
an exception, the exception is raised to the host's unhandled exception mechanism.

## Complex Cancellation Graphs
You can model complex cancellation graphs by further entangling a `CancellationTokenSource` with one or more
`CancellationToken` objects.

For example, you can have a multiple `CancellationTokenSource` objects for various asynchronous operations
(such as fetching data, running animations, etc.) that are linked back to a root `CancellationTokenSource`
that can be used to cancel all operations (such as when the user navigates to another page):

```ts
const root = new CancellationTokenSource();
const animationSources = new WeakMap();
let completionsSource;

function onNavigate() {
  root.cancel();
}

function onKeyPress(e) {
  // cancel any existing completion
  if (completionsSource) completionsSource.cancel();

  // create and track a cancellation source linked to the root
  completionsSource = new CancellationTokenSource([root.token]);

  // fetch auto-complete entries
  fetchCompletions(e.target.value, completionsSource.token);
}

function fadeIn(element) {
  // cancel any existing animation
  const existingSource = animationSources.get(element);
  if (existingSource) existingSource.cancel();

  // create and track a cancellation source linked to the root
  const fadeInSource = new CancellationTokenSource([root.token]);
  animationSources.set(element, fadeInSource);

  // hand off element and token to animation
  beginFadeIn(element, fadeInSource.token);
}
```

Another usage is to create a `CancellationTokenSource` linked to other asynchronous operations:

```ts
async function startMonitoring(timeoutSource, disconnectSource) {
  const monitorSource = new CancellationTokenSource([timeoutSource, disconnectSource]);
  while (!monitorSource.cancellationRequested) {
    await pingUser();
  }
}
```

# Cancellation Objects

## Class: CancellationTokenSource
Signals a [CancellationToken](#class-cancellationtoken) that it should be canceled.

#### Syntax

```ts
class CancellationTokenSource {
  constructor(linkedTokens?: Iterable<CancellationToken>);
  readonly token: CancellationToken;
  cancel(): void;
  close(): void;
}
```

### new CancellationTokenSource(linkedTokens?)
Initializes a new instance of a CancellationTokenSource.
* `linkedTokens` [&lt;Iterable&gt;][Iterable] An optional iterable of tokens to which to link this source.

By supplying a set of linked tokens, you can model a complex cancellation graph that allows you to signal
cancellation to various subsets of a more complex asynchronous operation. For example, you can create a
cancellation hierarchy where a root `CancellationTokenSource` can be used to signal cancellation for all
asynchronous operations (such as when signing out of an application), with linked `CancellationTokenSource`
children used to signal cancellation for subsets such as fetching pages of asynchronous data or stopping
long-running background operations in a Web Worker. You can also create a `CancellationTokenSource` that
is attached to multiple existing tokens, allowing you to aggregate multiple cancellation signals into
a single token.

### source.token
Gets the CancellationToken linked to this source.
* Returns: [&lt;CancellationToken&gt;](#class-cancellationtoken)

### source.cancel()
Cancels the source, evaluating any registered callbacks. If any callback raises an exception,
the exception is propagated to a host specific unhandled exception mechansim (e.g. `window.onerror` 
or `process.on("uncaughtException")`).

### source.close()
Closes the source, preventing the possibility of future cancellation. If the *source* is linked to any
existing tokens, the links are unregistered.

## Class: CancellationToken
Propagates notifications that operations should be canceled.

#### Syntax
```ts
class CancellationToken {
    static readonly none: CancellationToken;
    static readonly canceled: CancellationToken;
    constructor(source: CancellationTokenSource);
    readonly cancellationRequested: boolean;
    readonly canBeCanceled: boolean;
    throwIfCancellationRequested(): void;
    register(callback: () => void): { unregister(): void; };
}
```

### CancellationToken.none
Gets a token which will never be canceled.
* Returns: [&lt;CancellationToken&gt;](#class-cancellationtoken)

### CancellationToken.canceled
Gets a token that is already canceled.
* Returns: [&lt;CancellationToken&gt;](#class-cancellationtoken)

### new CancellationToken(source)
Creates a new CancellationToken linked to an existing CancellationTokenSource.
* `source` [&lt;CancellationTokenSource*gt;][#class-cancellationtokensource]
* Returns: [&lt;CancellationToken&gt;](#class-cancellationtoken)

### token.cancellationRequested
Gets a value indicating whether cancellation has been requested.
* Returns: [&lt;Boolean&gt;][Boolean]

### token.canBeCanceled
Gets a value indicating whether the underlying source can be canceled.
* Returns: [&lt;Boolean&gt;][Boolean]

### token.throwIfCancellationRequested()
Throws a [CancelError](#class-cancelerror) if cancellation has been requested.

### token.register(callback)
Registers a callback to execute when cancellation is requested.
* `callback` [&lt;Function&gt;][Boolean] The callback to register.
* Returns: [&lt;Object&gt;][Object] An object that can be used to unregister the callback.

## Class: CancelError
An error thrown when an operation is canceled.

#### Inheritance hierarchy
* [Error][Error]
  * CancelError

#### Syntax
```ts
class CancelError extends Error {
    constructor(message?: string);
}
```

### new CancelError(message?)
Initializes a new instance of the CancelError class.
* `message` [&lt;String&gt;][String] An optional message for the error.

# Examples
The following examples demonstrate some of the key concepts of the `CancellationTokenSource` and `CancellationToken`:

## Promise Producer, Cancellation Consumer
The `fetchAsync` method below produces a Promise, and can consume a cancellation signal:

```js
function fetchAsync(url, cancellationToken = CancellationToken.none) {
  return new Promise((resolve, reject) => {
    // throw (reject) if cancellation has already been requested.
    cancellationToken.throwIfCancellationRequested();

    const xhr = new XMLHttpRequest();

    // save a callback to abort the xhr when cancellation is requested
    const oncancel = () => {
      // abort the request
      xhr.abort();

      // reject the promise
      reject(new CancelError());
    }

    // register the callback to execute when cancellation is requested
    const registration = cancellationToken.register(oncancel);

    // wait for the remote resource
    xhr.onload = event => {
      // async operation completed, stop waiting for cancellation
      registration.unregister();

      // resolve the promise
      resolve(event);
    }

    xhr.onerror = event => {
      // async operation failed, stop waiting for cancellation
      registration.unregister();

      // reject the promise
      reject(event);
    }

    // begin the async operation
    xhr.open('GET', url, /*async*/ true);
    xhr.send(null);
  });
}
```

## Cancellation Producer, Promise Consumer
The `fetchConsumer` method below can produce a cancellation signal, and consumes a Promise.

```js
function fetchConsumer(url) {
  const source = new CancellationTokenSource();
  setTimeout(() => source.cancel(), 1000); // cancel after 1sec.
  return fetchAsync(url, source.token);
}
```

## Promise Consumer, Cancellation Consumer
The `fetchMiddle` function below receives a *CancellationToken* from its caller, which it can choose to
listen to and pass along, but cannot cancel the CancellationTokenSource of its upstream caller. In addition,
this function will receive the Promise produced by `fetchAsync` and can listen to the result, but cannot
resolve or reject the Promise of the downstream Promise producer.

```js
function fetchMiddle(url, cancellationToken = CancellationToken.default) {
  document.querySelector("#loading").style.display = 'block';

  // Secondary consumer *can* listen for cancellation...
  const ondone = () => document.querySelector("#loading").style.display = 'none';
  const registration = cancellationToken.register(ondone);

  return fetchAsync(url, cancellationToken)
    .then(value => {
      registration.unregister();
      ondone();
      return value;
    }, reason => {
      registration.unregister();
      ondone();
      return Promise.reject(reason);
    })
}
```

## Upstream Promise Consumer
Another benefit to this mechanism for cancellation, is that upstream consumers of a Promise from a library can only affect the canceled state of a downstream Promise producer if the API of the downstream library accepts a CancellationToken argument. Upstream consumers cannot directly affect the state of the downstream Promise. For example, consider a grid that performs UI virtualization:

```js
class Grid {
  constructor(dataUrl) {
    // cancels all network traffic when the Grid is destroyed
    this._cleanupSource = new CancellationTokenSource();
    this._rows = new Array();
    this._dataUrl = dataUrl;
  }

  // ...

    // somewhat naive, we always fetch the rows in the background so that we can cache them in memory,
    // even when a new fetch is requested.
  _fetchRows(offset, count) {
    if (this._hasCachedRows(offset, count)) {
      return Promise.resolve();
    }

    return fetchAsync(dataUrl + "?offset=" + offset + "&count=" + count, this._cleanupSource.token)
      .then(event => {
        const result = JSON.parse(event.source.responseText);
        for (var i = 0; i < result.length; i++) {
          this._rows[offset + i] = result[i];
        }
      });
  }

  // handles the "click" event of a next page button. prevPage would be similar.
  // an external caller can request a new page, but cannot cancel the underlying network operation.
  nextPage() {
    // cancel any previous page change
    if (this._pageSource) {
      this._pageSource.cancel();
    }

    // set the current page change, linking it to the cleanup source.
    const pageSource = new CancellationTokenSource([this._cleanupSource.token]);
    this._pageSource = pageSource;

    const page = this.page + 1;
    const count = this.pageSize;
    const offset = page * count;

    // fetch the rows (either from cache or from the remote store)
    return this._fetchRows(offset, count).then(() => {
      // if a new page was requested, stop processing.
      if (pageSource.token.cancellationRequested) {
        return;
      }

      pageSource.close();
      this._displayPage(page);
    });
  }

  // destroys the Grid, cancel any pending network activity
  destroy() {
    // this both cancels any pending network activity as well as cancels any linked page changes.
    this._cleanupSource.cancel();
  }
}
```

# Extensibility

See [EXTENSIBILITY.md](EXTENSIBILITY.md).

# Stretch Goals

The following are possible stretch goals to the above proposal that may be "nice to have" but should not
be considered blockers for possible adoption:

* Add a `reason` argument to the callback supplied to `CancellationToken#register` that can be used to observe the cancellation signal to better interoperate with the `reject` callback for a `Promise`.
* Add an optional `reason` argument to `CancellationTokenSource#cancel` that can be used to provide a custom cancellation signal other than `CancelError`.

# Reference Implementation

A reference implementation can be found in the `prex` library:

* npm: https://www.npmjs.com/package/prex or `npm install prex`
* source: https://github.com/rbuckton/prex

# Resources

- [Overview slides](https://tc39.github.io/proposal-cancellation/CancellationPrimitives-tc39.pptx)

# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.  
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.  
* [x] Illustrative [examples][Examples] of usage.  
* [x] High-level [API][API].  

### Stage 2 Entrance Criteria

* [ ] [Initial specification text][Specification].  
* [ ] _Optional_. [Transpiler support][Transpiler] (for syntax) or [Polyfill][Polyfill] (for API).  

### Stage 3 Entrance Criteria

* [ ] [Complete specification text][Specification].  
* [ ] Designated reviewers have [signed off][Stage3ReviewerSignOff] on the current spec text.  
* [ ] The ECMAScript editor has [signed off][Stage3EditorSignOff] on the current spec text.  

### Stage 4 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have been written for mainline usage scenarios and [merged][Test262PullRequest].  
* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].  
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.  
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].  

<!-- The following are shared links used throughout the README: -->

[Object]: https://tc39.github.io/ecma262/#sec-object-constructor
[String]: https://tc39.github.io/ecma262/#sec-string-constructor
[Boolean]: https://tc39.github.io/ecma262/#sec-boolean-constructor
[Function]: https://tc39.github.io/ecma262/#sec-function-constructor
[Error]: https://tc39.github.io/ecma262/#sec-error-constructor
[Iterable]: https://tc39.github.io/ecma262/#sec-symbol.iterator
[JobQueue]: https://tc39.github.io/ecma262/#sec-jobs-and-job-queues
[Champion]: #status
[Prose]: #proposal
[Examples]: #examples
[API]: #cancellation-objects
[Specification]: https://tc39.github.io/proposal-cancellation
[Transpiler]: #todo
[Polyfill]: https://github.com/rbuckton/prex
[Stage3ReviewerSignOff]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo
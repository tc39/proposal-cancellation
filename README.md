# ECMAScript Cancellation

This proposal seeks to define an approach to user-controlled cancellation of asynchronous operations
through the adoption of a set of native platform objects.

## Status

**Stage:** 1  
**Champion:** Ron Buckton (@rbuckton), Brian Terlson (@bterlson), Domenic Denicola (@domenic), Yehuda Katz (@wycats)

_For more information see the [TC39 proposal process](https://tc39.github.io/process-document/)._

> NOTE: TC39 has decided to investigate a cancellation mechanism in the core library.
> As such, Cancellation has moved to Stage 1, but **not** in the form of the previous Stage 0 proposal.
> Instead, TC39 believes this is a space that requires further investigation and discussion.
> The previous Stage 0 proposal can be found [here](stage0/README.md).

## Authors

* Ron Buckton (@rbuckton)

# Motivations

* A clear and consistent approach to cancelling asynchronous operations:
  * Asynchronous functions or [iterators](https://github.com/tc39/proposal-async-iteration)
  * [Fetching](https://fetch.spec.whatwg.org/#fetch-api) remote resources (HTTP, I/O, etc.)
  * Interacting with background tasks (Web Workers, forked processes, etc.)
  * Long-running operations ([animations](https://w3c.github.io/web-animations/), etc.)
* A general-purpose coordination mechanism with many use cases:
  * Synchronous observation (e.g. in a game loop)
  * Asynchronous observation (e.g. aborting an XMLHttpRequest, stopping an animation)
  * Easy to use in async functions.
* A common API that is reusable in multiple host environments (Browser, NodeJS, IoT/embedded, etc.).

# Prior Art

* [Cancellation in Managed Threads](https://msdn.microsoft.com/en-us/library/dd997364(v=vs.110)) in the .NET Framework

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
[Specification]: #todo
[Transpiler]: #todo
[Polyfill]: #todo
[Stage3ReviewerSignOff]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo

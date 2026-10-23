# ECMAScript Cancellation

This proposal seeks to add a to-be-determined subset of the WHATWG AbortController and AbortSignal interfaces to JavaScript, with minor modifications if absolutely necessary.

Historically this proposal considered other approaches, which you can see in [the stage 0 README](stage0/README.md) and in git history. Since then, AbortController has shipped and become ubiquitous across a variety of JavaScript runtimes, and the committee does not believe it makes sense to standardize a different cancellation mechanism. As such the goal is now to do the spec work necessary for specifications (including the JavaScript specification itself) to use AbortController without needing the rest of the entailed machinery (such as EventTarget).

This is not expected to affect developers on platforms that already provide AbortController, except that [some extensions may be necessary](https://github.com/whatwg/dom/pull/1425) and will be included on those platforms.

## Status

**Stage:** 1  
**Champions:** Ron Buckton ([@rbuckton](https://github.com/rbuckton)), James M. Snell ([@jasnell](https://github.com/jasnell)), Kevin Gibbons ([@bakkot](https://github.com/bakkot))

_For more information see the [TC39 proposal process](https://tc39.es/process-document/)._

## Authors

* Ron Buckton ([@rbuckton](https://github.com/rbuckton))
* Kevin Gibbons ([@bakkot](https://github.com/bakkot))

## Motivations

* A clear and consistent approach to cancelling asynchronous operations
* A general-purpose coordination mechanism with many use cases:
  * Synchronous observation (e.g. in a game loop)
  * Asynchronous observation (e.g. a network request)
  * Easy to use in async functions
* A common API that is reusable in multiple host environments (Browser, NodeJS, IoT/embedded, etc.).

## Prior Art

* [Cancellation in Managed Threads](https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads) in the .NET Framework

## Architecture
The following are some architectural observations provided by **Dean Tribble** on the [es-discuss mailing list](https://web.archive.org/web/20151210071401/https://mail.mozilla.org/pipermail/es-discuss/2015-March/041887.html):

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

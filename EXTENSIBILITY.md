The following examples illustrate CancellationTokenSource extensibility (leverages the [private names](https://github.com/littledan/proposal-unified-class-features) proposal):

## DOM EventTarget

```js
// Custom source for cancellation
class CustomSource extends CancellationTokenSource {
    #token;

    constructor(...args) {
        super(...args);
        #token = new CustomToken(this); // customize token returned by source
    }

    get token() {
        return #token;
    }
}

// Custom cancellation token
class CustomToken extends CancellationToken {
    static #none;
    static #canceled;

    #oncancel = null;
    #listeners = new Map();
    #registration;

    static get none() {
        if (!#none) {
            const source = new CustomSource();
            source.close(); // prevent cancellation
            #none = source.token;
        }
        return #none;
    }

    // Could make this easier with [Symbol.species]...
    static get canceled() {
        if (!#canceled) {
            const source = new CustomSource();
            source.cancel(); // force cancellation
            #canceled = source.token;
        }
        return #canceled;
    }

    // emulate obj.onevent
    get oncancel() { 
        return #oncancel; 
    }

    set oncancel(callback) {
        if (#oncancel === callback) {
            return;
        }
        if (typeof #oncancel === "function") {
            this.removeEventListener("cancel", #oncancel);
        }
        #oncancel = callback;
        if (typeof #oncancel === "function") {
            this.addEventListener("cancel", #oncancel);
        }
    }

    throwIfCancellationRequested() {
        if (this.cancellationRequested) {
            throw new CustomError(); // throw custom error
        }
    }

    addEventListener(type, callback) {
        let eventListeners = #listeners.get(type);
        if (!eventListeners) {
            #listeners.set(type, eventListeners = []);
        }
        eventListeners.push(callback);
        if (type === "cancel" && !#registration) {
            #registration = this.register(() => #canceled());
        }
    }

    removeEventListener(type, callback) {
        const eventListeners = #listeners.get(type);
        if (!eventListeners) {
            return;
        }
        for (let i = 0; i < eventListeners.length; i++) {
            if (eventListeners[i] === callback) {
                eventListeners.splice(i, 1);
                break;
            }
        }
        if (type === "cancel" && eventListeners.length === 0) {
            #registration.unregister();
            #registration = undefined;
        }
    }

    dispatchEvent(event) {
        const eventListeners = #listeners.get(event.type);
        if (!eventListeners) {
            return true;
        }
        event.target = this;
        for (const callback of eventListeners) {
            callback.call(this, event);
        }
        return !event.defaultPrevented;
    }

    #canceled() {
        this.dispatchEvent(new CustomEvent("cancel")); // dispatch cancellation using EventTarget
    }
}

// Custom error
class CustomError extends Error {
    name = "CustomError";
}
```

## NodeJS EventEmitter (mix-in)

```js
const events = require("events");

// Custom source for cancellation
class CustomSource extends CancellationTokenSource {
    #token;

    constructor(...args) {
        super(...args);
        #token = new CustomToken(this); // customize token returned by source
    }

    get token() {
        return #token;
    }
}

// Custom cancellation token
class CustomToken extends CancellationToken {
    static #none;
    static #canceled;

    #registration;

    constructor(source) {
        super(source);
        // initialize EventEmitter as mix-in
        EventEmitter.init.call(this);
        this.on("newListener", eventName => #newListener(eventName));
        this.on("removeListener", eventName => #removeListener(eventName));
    }

    // Could make this easier with [Symbol.species]...
    static get none() {
        if (!#none) {
            const source = new CustomSource();
            source.close(); // prevent cancellation
            #none = source.token;
        }
        return #none;
    }

    // Could make this easier with [Symbol.species]...
    static get canceled() {
        if (!#canceled) {
            const source = new CustomSource();
            source.cancel(); // force cancellation
            #canceled = source.token;
        }
        return #canceled;
    }

    throwIfCancellationRequested() {
        if (this.cancellationRequested) {
            throw new CustomError(); // throw custom error
        }
    }

    #newListener(eventName, listener) {
        if (eventName === "cancel" && !#registration) {
            #registration = this.register(() => #canceled());
        }
    }

    #removeListener(eventName, listener) {
        if (eventName === "cancel" && #registration && this.listenerCount("cancel") === 0) {
            #registration.unregister();
            #registration = undefined;
        }
    }

    #canceled() {
        this.emit("cancel");
    }
}

// mix-in EventEmitter methods
for (const key of Object.getOwnPropertyNames(EventEmitter.prototype)) {
    Object.defineProperty(CustomToken.prototype, key, Object.getOwnPropertyDescriptor(EventEmitter.prototype, key));
}

// Custom error
class CustomError extends Error {
    name = "CustomError";
}
```

# Notes

* We could simplify the extensibility mechanism by defining two symbols:
    * `Symbol.tokenSpecies` - Set on the `CancellationTokenSource` constructor, specifies the species constructor
      of the entangled token to create (used by `source.token`).
    * `Symbol.tokenSourceSpecies` - Set on the `CancellationToken` constructor, specifies the species constructor
      of the entangled source for the token (used by `CancellationToken.none` and `CancellationToken.canceled`).

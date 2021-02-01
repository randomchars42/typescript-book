## Typesafe Event Emitter

Conventionally in Node.js and traditional JavaScript you have a single event emitter. This event emitter internally tracks listener for different event types e.g. 

```ts
const emitter = new EventEmitter();
// Emit: 
emitter.emit('foo', foo);
emitter.emit('bar', bar);
// Listen: 
emitter.on('foo', (foo)=>console.log(foo));
emitter.on('bar', (bar)=>console.log(bar));
```
Essentially `EventEmitter` internally stores data in the form of mapped arrays: 
```ts
{foo: [fooListeners], bar: [barListeners]}
```
Instead, for the sake of *event* type safety, you can create an emitter *per* event type:
```ts
const onFoo = new TypedEvent<Foo>();
const onBar = new TypedEvent<Bar>();

// Emit: 
onFoo.emit(foo);
onBar.emit(bar);
// Listen: 
onFoo.on((foo)=>console.log(foo));
onBar.on((bar)=>console.log(bar));
```

This has the following advantages: 
* The types of events are easily discoverable as variables.
* The event emitter variables are easily refactored independently.
* Type safety for event data structures.

### Reference TypedEvent
```ts
export interface Listener<T> {
  (event: T): any;
}

export interface Disposable {
  dispose();
}

/** passes through events as they happen. You will not get events from before you start listening */
export class TypedEvent<T> {
  private listeners: Listener<T>[] = [];
  private listenersOncer: Listener<T>[] = [];

  on = (listener: Listener<T>): Disposable => {
    this.listeners.push(listener);
    return {
      dispose: () => this.off(listener)
    };
  }

  once = (listener: Listener<T>): void => {
    this.listenersOncer.push(listener);
  }

  off = (listener: Listener<T>) => {
    var callbackIndex = this.listeners.indexOf(listener);
    if (callbackIndex > -1) this.listeners.splice(callbackIndex, 1);
  }

  emit = (event: T) => {
    /** Update any general listeners */
    this.listeners.forEach((listener) => listener(event));

    /** Clear the `once` queue */
    if (this.listenersOncer.length > 0) {
      const toCall = this.listenersOncer;
      this.listenersOncer = [];
      toCall.forEach((listener) => listener(event));
    }
  }

  pipe = (te: TypedEvent<T>): Disposable => {
    return this.on((e) => te.emit(e));
  }
}
```

### Typed EventEmitter emitting multiple type safe events

```ts
// modified after:
// https://codereview.stackexchange.com/a/215317/236617
// and
// https://basarat.gitbook.io/typescript/main-1/typed-event

interface Disposable {
  dispose(): void;
}

// used to check if parameters for .emit() are valid parameters for the event
// `Symbol` is needed to tell the compiler the resulting type can be `spread`
type EventTypes<T> = Symbol & T extends (...args: infer U) => any ? U: never;
// create a map:
// EventMap = { "ListenerType1": Function[], ..., "ListenerTypeN": Function[] }
type ListenerRecord<T> = {[K in keyof T]: T[keyof T]};
type EventMap<T extends ListenerRecord<T>> = {[K in keyof T]: T[K][]};

class EventEmitter<T extends ListenerRecord<T>> {
  protected _listeners: EventMap<T>;
  protected _single_use_listeners: EventMap<T>;

  // the EventMap must be constructed manually, see example below
  constructor(events: EventMap<T>) {
    this._listeners = JSON.parse(JSON.stringify(events));
    this._single_use_listeners = JSON.parse(JSON.stringify(events));
  }

  on<K extends keyof T>(event_name: K, listener: T[K]): Disposable {
    this._listeners[event_name].push(listener);
    return {
      dispose: () => {
        this.off(event_name, listener);
      }
    }
  }

  once<K extends keyof T>(event_name: K, listener: T[K]): void {
    this._single_use_listeners[event_name].push(listener);
  }

  off<K extends keyof T>(event_name: K, listener: T[K]): void {
    const listeners = this._listeners[event_name];
    const index = listeners.indexOf(listener);
    if (index !== -1) {
      listeners.splice(index, 1);
    }
  }

  emit<K extends keyof T>(event_name: K, ...param_list: EventTypes<T[K]>): void {
    this._listeners[event_name].forEach((listener) => listener(...param_list));

    if (this._single_use_listeners[event_name].length > 0) {
      const callstack = this._single_use_listeners[event_name];
      this._single_use_listeners[event_name] = [];
      callstack.forEach((listener) => listener(...param_list));
    }
  }
}

// example implementation:

// events "congratulate" and "clap_hands" expect different types (/ numbers) of parameters
interface HappyEvents {
  congratulate(event: string): void,
  clap_hands(event: number): void,
}

class HappyEmitter extends EventEmitter<HappyEvents> {
  // your additional functions go here
}
const happy_emitter = new HappyEmitter({"congratulate": [], "clap_hands": []});

// or (if you don't need additional methods)

const happy_emitter2 =  new EventEmitter<HappyEvents>({"congratulate": [], "clap_hands": []});

// register events

happy_emitter.on("congratulate", (param) => console.log(param + "Yay!"));
happy_emitter.once("congratulate", () => console.log("Lalala"));
happy_emitter.on("clap_hands", () => console.log("clap"));
// do not attempt:
happy_emitter.on("congratulate", (param: number) => console.log(param));
// throws: Argument of type '(param: number) => void' is not assignable to parameter of type '(event: string) => void'.
happy_emitter.on("clap_hands", (param: number, param2: string) => console.log(param));
// throws: Argument of type '(param: number, param2: number) => void' is not assignable to parameter of type '(event: string) => void'.

// fire events

happy_emitter.emit("congratulate", "Yippey ya ");
happy_emitter.emit("congratulate", "Hooray! ");
happy_emitter.emit("clap_hands", 2);
happy_emitter.emit("clap_hands", 1);
// do not attempt:
happy_emitter.emit("congratulate", 2);
// throws: Type 'number' is not assignable to type 'string'
happy_emitter.emit("clap_hands", "r");
// throws: Type 'string' is not assignable to type 'number'
```

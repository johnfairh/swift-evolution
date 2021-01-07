# Distributed Actors

* Proposal: [SE-NNNN](NNNN-distributed-actors.md)
* Authors: [Konrad 'ktoso' Malawski](https://github.com/ktoso), [Doug Gregor](https://github.com/DougGregor)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: Available in [recent `main` snapshots](https://swift.org/download/#snapshots) behind the flag `-Xfrontend -enable-experimental-concurrency`

## Introduction


Swift's introduction of [actors](https://github.com/DougGregor/swift-evolution/blob/actors/proposals/nnnn-actors.md) allows us to fully embrace the [actor model](https://en.wikipedia.org/wiki/Actor_model) of computation. By modeling various entities, such as devices, players, and any other sub-system of an application as actors, we are able to easily communicate between those concurrent entities without having to worry too much about low level synchronization and concurrency issues.

That is already great, however actors are _much more_ than just concurrency primitive. They are a general model allowing us to abstract over computation performed by entities with which we communicate _only_ by sending messages. This also happens to be exactly how distributed applications communicate!

This proposal introduces `distributed actor`s, which can be used to represent actors located on remote processes or hosts, and can be communicated with using transparent message passing, in a similar way as local actors enable communication between threads within a process.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

#### Related Proposals

Pre-requisites:

- [SE-0296: async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md) - asynchronous functions are used to express distributed functions,
- [SE-NNNN: Actors](https://forums.swift.org/t/concurrency-actors-actor-isolation/41613) - are the basis of this proposal, as it refines them in terms of a distributed actor model by adding a few additional restrictions to the existing isolation rules already defined by this proposal,

Related:

- [SE-NNNN: Task Local Values](https://github.com/apple/swift-evolution/pull/1245) - are accessible to transports, which use task local values to transparently handle request deadlines and implement distributed tracing systems (this may also apply to multi-process Instruments instrumentations),
- [SE-0295: Codable synthesis for enums with associated values](https://github.com/apple/swift-evolution/blob/main/proposals/0295-codable-synthesis-for-enums-with-associated-values.md) - because distributed functions relying heavily on `Codable` types, and runtimes may want to express entire messages as enums, this proposal would be tremendously helpful to avoid developers having from ever dropping down to manual Codable implementations,

Follow-ups:

- SwiftPM Extensible build-tools – which will enable source-code generators, necessary to fill-in distributed function implementations by specific distributed actor transport frameworks.

## Motivation

Wether we want it or not, most of the systems we write nowadays are distributed. They may:

- use multiple processes and communicate across them using IPC mechanisms (such as XPC on Apple platforms, or custom mechanisms on other platforms), in order to increase security and isolate parts of an application from hard faults, 
- or they may communicate with other nearby devices directly, or through the cloud
- or they may to interact with server-side systems which often entails much boilerplate and ceremony surrounding the encoding/decoding of the data and putting it on the wire, 
- and last but not least, complex clustered server-side systems often need to interact between nodes of the same application in order to meet their consistency and availability requirements.

All those use-cases, rightfully so, are very different and have very different underlying transports and mechanisms that enable them. However the general concept of wanting to communicate with a named, identifiable entity on a remote host comes up very frequently, yet each time the code looks tremendously differently once we get to actually interact with it.

Distributed actors provide an extension point in Swift's actor runtime that enables it to truly reach enable actors as a general conceptual model for asynchronous communication, regardless if in-process or not.

This design _does not_ define any specific runtime, however it's design is made such, that implementations of `ActorTransport`s serving the following primary use-cases are well served by the very flexible nature of this proposal:

- Network Actor Transports
  - clustered server systems, simplifying implementation of scalable server-side systems and distributed algorithms, these can be implemented on top of Swift NIO,
  - an alternative approach to RPC libraries, offering a more Swifty API while calling into known underlying RPC libraries.
- IPC (Inter-Process Communication) Actor Transports
  - on Apple platforms XPC is the primary way applications communicate between daemons and apps. It is widely used yet has yet to offer a good Swift API. It's existing APIs are `libxpc` (C), and `NSXPC` (Objective-C) which work well, but feel cluncky when used from Swift.
  - other platform's, or bespoke ipc mechanisms.


Developers spend large amounts of time building almost the same thing again and again. Parsing JSON payloads, stuffing parameters into and out of protocol buffer messages and each time doing it slightly differently again. 

Our goal is to bring back productivity and sanity to the world of modern multi-device, multi-node distributed systems, by using a model that is inherently native to distributed systems: actors.

Developers spend far too much time and effort on wrangling with various almost the same but different ways to ship bytes around their systems. Sometimes manually decoding JSON, sometimes using binary payloads using NSSecureCoding over XPC to process daemons and other times using custom protocols and raw NIO implementations.

## Proposed solution

### Distributed Actors

This proposal introduces an extra modifier to actors "`distributed`" which enables some additional restrictions to what the typesystem already is checking in terms of actor isolation (more details below). 

Instead of implementing time and time again logic around serializing, sending over the network, receiving, and then serializing the same payloads as many of us do today, distributed actors allow us to encapsulate all of the serialization and message passing logic and regain our focus on the focus on logic:

```swift
distributed actor Player { 
  distributed func shot(ball at: Location) async throws -> Shot {
    // ... 
  }
}
```

So our application can much simpler, and now we can actually reason about what is happening: 

```swift
let shot = await player.shot(ball: )
if shot.wasHit { 
  showHit()
}
```

## Detailed design

### Distributed Actors

This proposal introduces the `distributed` modifier that may be used with an `actor class` as well as functions within such distributed actor. It is spelled as:

```swift
distributed actor Greeter { }
```

> NOTE: An alternative spelling we would consider acceptable is `remote` as in `remote actor`, which describes the nature of such actors equally well.


In the same way that an `actor Greeter` implicitly is made to conform to the `Actor` protocol, a type declared as a `distributed actor` is implicitly conforming to the special `DistributedActor` protocol. The distributed actor protocol is defined as:

```swift
public protocol DistributedActor: Actor, Codable {
    var transport: ActorTransport { get } 
    
    var address: ActorAddress { get }
    
    // Alternatively, a good spelling for this type would be:
    // var context: DistributedActorContext { get } 
    //
    // since the following then gain a more natural spelling:
    //   context.transport: ActorTransport { get } 
    //   context.address: ActorAddress { get } 
}
```

It is _not_ permitted to conform to this protocol manually, in the same manner as it is not allowed to conform to `Actor` manually.

> The Codable conformance *crucial* for the system to compose and actors work correctly in a distributed system. It's implementation is special and provided in the library itself. It is discussed in depth in the [Distributed Actors are Codable](#distributed-actors-are-codable) section.

Distributed actors are the only place where `distributed func` declarations are allowed. This is because in order to implement a distributed function, a transport and identity (actor address) are necessary.

Furthermore, such declarations extend the language model by introducing the concept of instances being either an "local actor" or a "proxy actor." A local actor is effectively the same as a non-distributed actor at runtime, it does have the `transport` property and more rigid type-checking was applied to any of its distributed functions, however in terms of runtime functioning and execution semantics it is exactly the same.

A "proxy" actor instance however, differs from a local actor in those two ways:

- how storage for the actor is allocated
- implementation of `distributed` function

### Distributed Actor "Proxy" Instance Allocation

Creating a proxy for an actor type is done using this special `resolve(address:using:)` function, defined on the DistributedActor protocol:

```swift
protocol DistributedActor { 
    /// Creates new, local, instance 
    init(transport: ActorTransport)
}

protocol ActorTransport { 
    // Resolve a local or remote address returning either a real reference (local actor) or proxy instance (remote actor).
    func resolve<Act: DistributedActor>(_ type: Act.Type, _ address: ActorAddress) throws -> Act
    
    // TODO: show to how to actually instanciate the proxy
}
```

This function can only be invoked on specific actor types–as usual with static functions on protocols–and serves as a factory function for actor proxies of given specific type. The returned value is of the same type as the invoked on actor, however it is a proxy of it, delecating all calls to the passed in transport. 

This function is a fundamental building block of distributed systems witha ctors, however it is not expected to be used directly by application developers. Instead, higher level abstractions such as clusters and de-serialization mechanisms (such as decoding an Actor passed in a Codable message) make use of this mechanism to turn addressess into actor references. Note that the such resolved address may be on the same host as the function is invoked on, or on a different host.

The `resolve` function is intentionally not asynchronous, in order to invoke it from inside `decode` implementations, as they may need to decode actor addresses into actor references.

To see this in action, consider:

```swift
distributed actor Greeter { ... }
```

Which can be used to create a proxy for by the following invocation:

```swift
let greeter = try Greeter.resolve(address: someAddress, using: clusterTransport)
```

Some transports may choose to never throw if encountering an "unknown" address, and they may instead return a "dead letters" style object, which will log each message sent to such unresolved actor. The specifics of how a resolve works are left up to the transport though, as their semantics depend on the capabilities of the underlying protocols the transport uses.

### Distributed Actors are `Codable`

In order to be true to the actor model's idea of freely shareable references, regardless of their location (known as the property of *location transparency*), we need to be able to pass distributed actor references to other distributed actors.

This implies that distributed actors must be `Codable`. However, the way that encoding and decoding is to be implemented differs tremendously for actors and non-actors. Specifically, it does not make sense to serialize the actor's state - it is after all wha it is isolating from the outside world, and external access 

As we know, declaring an `distributed actor` makes the type conform to `DistributedActor`.

```swift
protocol DistributedActor: Actor, Codable {
    var transport: ActorTransport { get }

    // Alternatively:
    // var context: DistributedActorContext { get } 
    //  where:
    //      context.address: ActorAddress { get }
    //      context.transport: DistributedActorTransport { get } 
    
    // ... synthesized Codable conformance ... 
}
```

The `DistributedActor` protocol also conforms to `Codable`, however it implements it in a _special_ way.

It _does not_ make sense to encode/decode "the actor," per se, however it _does_ make sense to encode/decode an "address" (in the distributed sense, not in the memory address sense) to the actor, which is understood by an `ActorTransport`.

Specifically, the conformance is implemented by encoding/decoding the `transport.address` as a single 

For example, we might implement an actor cluster transport, where the actor addressess are a form of URIs, uniquely identifying the actor instance, and as such encoding an actor turns it into `"[transport-name]://system@10.0.0.1:7337/Greeter#349785`, or using some other encoding scheme, depending on the transport used.

This allows us to pass actors as references across distributed boundaries:

```swift
distributed actor Person {
  distributed func greet(_ greeting: String) async throws {
    log.info("I was greeted: '\(greeting)' yay!")
  }
}

distributed actor Greeter { 
  var greeting: String = "Hello, there!"
  
  distributed func greet(person: Person) async throws {
    person.greet("\(greeting)!
  }
}
```

The `ActorTransport` will be explained in depth in the detailed design sections, in general though the transport is to be considered an actor local value, which is not intended to be shared with other actors (i.e. unmovable). While the `self.transport.address` is possible to be shared with other actors, or even stored by them. It is a common pattern to store a set of known actors or addresses, in order to maintain a "watched" set of actors, which we can use to check incoming addresses against - is this message from a "new" actor, or one which I already have seen and stored in my "watched" set? These kinds of situations come up frequently in distributed system programming, and it is useful to be able to store addresses, with detachment to any actual reference to the real remote actor.


### Distributed Actor Functions

### `distributed func` declarations

Distributed functions need to be marked as such to participate in code generations.

A distributed actor function is declared as follows:

```swift
distributed actor Greeter { 
  distributed func greet(name: String) async throws -> String { 
    "Hello, \(name)!"
  }
}
```

A distributed actor function must:

- have all it's parameters conform to `Codable`
- have a `Codable` (or `Void`) return type
- be declared as `async throws` because the transport may need to decide to cancel a call, due to network issues, timeouts, or other transport issues


### `distributed func` internals

Developers implement distributed functions the same way as they would any other function. This is a core gain from this model, as compared to external source generation schemes which force users to implement and provide so-called "Stubs". Using the distributed actor model, the types we program with are the types that can be used as proxies - there is no need for additional intermediate types.

A local (or "real") actor instance is a simple actor which implements all of its async functions by enqueueing the calls to its queue by the `Actor.enqueue(partialTask:)` function, conceptually this could be expressed as:

```swift
// local-actor pseudo-code (!)
func greet(name: String) async -> String { 
  let partialTask = __makeTaskRepresentingThisInvocation()
  self.enqueue(partialTask)
  return await partialTask.get()
}
```

Distributed actors are very much the same, just that instead of enqueueing a task into the local queue, they convert the call into a message and send it over the wire (using whichever transport is active for this actor). Local actors and distributed actors are much more similar to eachother than one might think at first - the main difference is the addition of message creation (serialization) and instead of enqueue a different transport mechanism is used to dispatch the message. 

The distributed actors design is purposefully detaching the implementation of such transport from the language, as we cannot and will not ship all kinds of different transport implementations as part of the language. Instead, what a distributed function does, is delegate to functions it assumes will exist at compile time, which are to be filled in by external code generation mechanisms (more on that in the coming sections).

```swift
// distributed actor pseudo-code (!)
func greet(name: String) async throws -> String { 
  // exact shape of this invocation TBD
  await try s.$distributed_greet(self.transport, name: name)
}
```

The function at compile time will generate where the `$distributed_[name]([params])` function is assumed to be provided by _someone_. In reality these functions will be provided by code generators which are able to turn the function calls (name + parameters) into `Codable` messages and dispatch such message onto the `transport` via `transport.send(address:message:expectingReply:)`.

An example of such code generated function would look like this:

```swift
// ~~~ A "specific transport" would generate such code ~~~
extension Greeter { 
  func $distributed_greet(
    name: String, 
    expectingReply: Reply.Type = Reply.self
  ) throws async -> Reply {
    // simplified implementation example
    
    // 1) serialize to message representation
    let message = Greeter.$Message.greet(name: name)
    let bytes = try self.transport.encoder(for: Greeter.$Message.self).encode(message)
    
    // 2) send the serialized bytes over the transport and await a response
    return await try self.transport.send(bytes)
  }
}
```

> Note: While it is up to a transport to decide the specific details of returning 

As we can see this is very similar to the local actor case and has two steps for the sending part:

- create representation of the invocation,
  - in the local case: a raw task to be executed on the actor,
  - in the distributed case: a Codable message representation to be sent over the wire to the recipient,
- enqueue the message
  - in the local case: on the actor's local mailbox / `queue`
  - in the distributed case: onto the transport, which will send it over the wire

### Distributed Actor Isolation

Distributed actor isolation inherits all the isolation properties of local actors, however it removes a few of the special/convenience cases a local actor is allowed to make, specifically:

#### 1. No permissive special case for accessing constant properties

This is a removal of a special case that local actors allow for convenience, however somewhat circumventing the model that any access to an actor's owned memory must be performed asynchronously. Local actors in Swift make a special case to permit _synchronous_ access to _constant properties_, e.g. `let name: String`, since they are known to be safe to access and cannot be modified either. Swift (as proposed currently) takes the stance that the convenience of this access is more useful than the general actor model take on it.

Such losening of the actor model is _not_ permissible for distributed actors.

Specifically, the following is permitted under Swift's local actor's permissive model:

```swift
actor LocalGreeter { 
  let details: String // "constant"
}
LocalGreeter().name // ok
```

yet is incorrect and must not be allowed for distributed actors (since such access _would_ involve remote calls):

```swift
distributed actor Greeter { 
  let details: String // "constant", yet distributed actor isolated
}
Greeter().details // error: property 'details' is distributed actor-isolated
```

Alternatively, it could be argued that the losening of the actor model restrictions should be removed entirely.

#### 2. Codable parameters and return values

(2) All parameter types and return type of a `distributed func` of Distributed actor functions, represent messages which will be sent over the `ActorTransport` and as such are required to be `Codable`. This helps developers do the right thing, and only send types which are intended for serialization. Note however, that a transport may have various limitations on what it allows to be sent -- it is left up to the distributed actor runtime to determine if and how to actually serialize a message. Including what (if any) `Encoder` to use when doing so.

```swift
distributed actor Greeter { 
  distributed func greet(name: String) async throws -> String {
    "Hello, \(name)"
  }
```

#### 3. Throwing functions

Currently, all distributed functions must be declared as `async throws`.

This is sadly conflating networking issues with logical errors that a programmer may want to express in their APIs. Swift does not have a different way to model errors other than throws, and it would be a benefitial future direction to explore what "panics" or "soft faults" can look like, an "actor crashing" could clean up all of it's related memory.

Alternative / Future direction:

- It would be better if we were able to express "failures" separately from "errors", however this goes beyond the initial scope of this work.
  - 

### Actor Transport

Distributed actors are always associated with a specific transport that handles a specific instance.


```swift


protocol ActorTransport { 
  var address: ActorAddress { get } 

  func send(
    bytes: Bytes, 
    expectingReply: Reply.Type
  ) async throws -> Reply
}
```

#### Transporting `Error`s

A transport _may_ attempt to transport errors back to the caller if it is able to encode/decode them, however this is _not encouraged_.

Instead, logic errors should be modelled by returning `Result<User, InvalidPassword>` as it allows for typed handling of such errors as well as automatically enforcing that the returned error type is also `Codable` and thus possible to encode and transport back to the caller. This is a good idea also because it forces developers to consider if an error really should be encoded or not (perhaps it contains large amounts of data, and a different representation of the error would be better suited for the distributed function).

Generally, it is encouraged to separate the "any failure" handling related to transport failures (such as timeouts or network errors), which are represented by the untyped `throws` of a distributed function call, from logical errors which (e.g. "invalid password").

The exact errors thrown by a distributed function depends on the underlying transport. Generally one should expect some form of `TheTransportError` to be thrown by a distributed function if transport errors occur -- exact semantics of those throws are intentionally left up to specific transports to document when and how to expect errors to be thrown.

To provide more tangible examples, why a transport may want to throw even if the called function does not, consider the following:

```swift
// Node A
distributed actor Failer { 
  distributed func letItCrash() { fatalError() } 
}
```

```swift
// Node B
let failer: Failer = ... 

// could throw transport-specific error, 
// e.g. "ClusterTransportError.terminated(actor:node:existenceConfirmed:)"
try await failer.letItCrash() 
```

This allows transports to implement failure detection mechanisms which are tailored to the specific underlying transport, e.g. for clustered applications one could make use of Swift Cluster Membership's [SWIM Failure Detector](https://www.github.com/apple/swift-cluster-membership), while for IPC mechanisms such as XPC more process-aware implementations can be provided. The exact guarantees and semantics of detecting failure will of course differ between those transports, which is why the transport must define how it handles those situations, while the language feature of distributed actors _must not_ define it any more specifically than discussed in this section.

### "Escape hatches"

The model provides ways for developers to open up "escape hatches" for specific APIs that may be used "only if the actor is local".

Future direction:

We could offer a builtin to check if the actor is local and promote it to a local reference via:

```swift
extension DistributedActor { 
  public static whenLocal(_ body: (actor) Self throws -> T) async rethrows -> T? {
    if let local = _asLocalActor(self) {
      await try body(local)
    } else return nil
  }
}

let actor: MaybeRemote = ...
actor.withLocal { 
  $0.tell("local, after all!")
}
```

### Bonus: Distributed Tracing & `Deadline` propagation

Thanks to the efforts of Moritz Lang the Swift Tracing ecosystem has already kicked off with good implementations ... 

Distributed Actors take this to the next level – trace across your application's IPC and clusters without lifting a finger (!).

`ActorTransport` authors can utilize all the tracing and instrument infrastructure already provided by the server side work group and simply instrument their transports to carry

Consider the following distributed system:

```swift
enum AppleStore { 
  distributed actor StorePerson { 
    let storage: Storage
  
    distributed func handle(order: Order, customer: Customer) async throws -> Device {
      let device = await try Task.withDeadline(in: .minutes(1)) { 
        await try storage.fetchDevice(order.deviceID)
      }
      
      guard await customer.processPayment(device.price) else {
        throw PaymentError.tryAgain
      }
      
      return device
    }
  }
}

// imagine this actor running on a completely different machine/host
extension AppleStore {
  distributed actor Storage { 
    distributed func fetchDevice(_ deviceID: DeviceID) async throws -> Device { 
      // ...
    }
  }
}
```

#### Transport extensibility


### Future Direction

#### Potential Transport Candidates

While this proposal intentionally does not introduce any specific transport, the obvious reason for introducing this feature is implementing specific actor transports. This proposal would feel incomplete if we would not share our general thoughts about which transports would make sense to be implemented over this mechanism, even if we cannot at this point commit to specifics about their implementations.

It would be very natural, and has been considered and ensured that it will be possible by using these mechanism, to build any of the following transports:

- clustering and messaging protocols for distributed actor systems, e.g. like [Erlang/OTP](https://www.google.com/search?q=erlang) or [Akka Cluster](https://doc.akka.io/docs/akka/current/typed/cluster-concepts.html),
- [inter-process communication](https://en.wikipedia.org/wiki/Inter-process_communication) protocols, e.g. XPC on Apple platforms or shared-memory
- various other RPC-style protocols, e.g. the standard [XML RPC](http://xmlrpc.com), [JSON RPC](https://www.jsonrpc.org/specification) or custom protocols with similar semantics.


#### Returning 
 
## Source compatibility

This change is purely additive to the source language. The additional use of the keyword `distributed` in `distributed actor class` and `distributed func` applies more restrictive requirements to the use of sucha actor class, however this only applies to new code, as such no existing code is impacted.

## Effect on ABI stability

This proposal is mostly additive.

**TODO:** The "proxy actor"... ???

## Effect on API resilience

TODO



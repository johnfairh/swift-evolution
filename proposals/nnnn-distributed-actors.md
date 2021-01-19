# Distributed Actors

* Proposal: [SE-NNNN](NNNN-distributed-actors.md)
* Authors: [Konrad 'ktoso' Malawski](https://github.com/ktoso), [Doug Gregor](https://github.com/DougGregor), Dario Rexin?, Tomer Doron?
* Review Manager: TBD
* Status: **Implementation in progress**
* Implementation: **in progress TODO: add link**

## Introduction


Swift's introduction of [actors](https://github.com/DougGregor/swift-evolution/blob/actors/proposals/nnnn-actors.md) allows us to fully embrace the [actor model](https://en.wikipedia.org/wiki/Actor_model) of computation. By modeling various entities, such as devices, players, and any other sub-system of an application as actors, we are able to easily communicate between those concurrent entities without having to worry too much about low level synchronization and concurrency issues.

That is already great, however actors are _much more_ than just concurrency primitive. They are a general model allowing us to abstract over computation performed by entities with which we communicate _only_ by sending messages. This also happens to be exactly how distributed applications communicate!

This proposal introduces `distributed actor`s, which can be used to represent actors located on remote processes or hosts, and can be communicated with using transparent message passing, in a similar way as local actors enable communication between threads within a process.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

#### Related Proposals

- **[Swift Concurrency Manifesto](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782#design-sketch-for-interprocess-and-distributed-compute)** - distributed actors as part of the language were first roughly outlined as a general future direction in the Concurrency Manifesto. The approach specified in this proposal takes inspiration from the manifesto, however may be seen as a reboot of the effort. We have invested a significant amount of time, research, prototypinig and implementing the approach since, and are confident in the details of the proposed model.

A note below the distributed compute section of the manifesto states:

> In any case, there is a bunch of work to do here, and it will take multiple years to prototype, build, iterate, and perfect it. It will be a beautiful day when we get here though.

Today, after long consideration and experimentation, we're happy to say: that time is now.

**Pre-requisites**

- [SE-0296: **async/await**](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md) - asynchronous functions are used to express distributed functions,
- [SE-NNNN: **Actors**](https://forums.swift.org/t/concurrency-actors-actor-isolation/41613) - are the basis of this proposal, as it refines them in terms of a distributed actor model by adding a few additional restrictions to the existing isolation rules already defined by this proposal,

**Related**

- [SE-NNNN: **Task Local Values**](https://github.com/apple/swift-evolution/pull/1245) - are accessible to transports, which use task local values to transparently handle request deadlines and implement distributed tracing systems (this may also apply to multi-process Instruments instrumentations),
- [SE-0295: **Codable synthesis for enums with associated values**](https://github.com/apple/swift-evolution/blob/main/proposals/0295-codable-synthesis-for-enums-with-associated-values.md) - because distributed functions relying heavily on `Codable` types, and runtimes may want to express entire messages as enums, this proposal would be tremendously helpful to avoid developers having from ever dropping down to manual Codable implementations.

**Follow-ups**

- **SwiftPM Extensible build-tools** – which will enable source-code generators, necessary to fill-in distributed function implementations by specific distributed actor transport frameworks.

**Related work**

- [Swift Distributed Tracing](https://github.com/apple/swift-distributed-tracing) – distributed actors are able to automatically and transparently participate in distributed tracing systems and instrumentation, this allows for a "profiler-like" performance debugging experience across entire fleets of servers,
- [Swift Cluster Membership](https://www.github.com/apple/swift-cluster-membership) ([blog](https://swift.org/blog/swift-cluster-membership/)) – cluster membership protocols are both natural to express using distributed actors, as well as very useful to implement membership for distributed actor systems.

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

This proposal introduces the `distributed` contextual keyword, which may be used in conjunction with  (`actor class`)  as well as `func` declarations within such `distributed actor class` declarations.

This keyword enables a few additional restrictions to what the typesystem already is checking in terms of actor isolation (more details below), and liberates those actors from their local affinity, allowing them to exist across process and network boundaries, fully embracing the message-passing nature of actors.

Instead of implementing time and time again logic around serializing, sending over the network, receiving, and then serializing the same payloads as many of us do today, distributed actors allow us to encapsulate all of the serialization and message passing logic and regain our focus on the focus on logic:

```swift
distributed actor Player { 
  var health: Health
  
  distributed func hit(by other: Player, from location: Location) async throws -> RemainingHealth {
    print("Hit by '\(other)' from \(location)")
    health.hit()
    return health.remaining
  }
}
```

So our application can much simpler, and now we can actually reason about what is happening: 

```swift
let shot = await player.receiveHit(from: ball.location)
if shot.wasHit { 
  showHit()
}
```

## Detailed design

### Distributed Actors

In the same way that an `actor Greeter` is implicitly made to conform to the `Actor` protocol, a type declared as `distributed actor` is implicitly conforming to the `DistributedActor` protocol. 

The distributed actor protocol is defined as:

```swift
public protocol DistributedActor: Actor, Codable {
  
  /// Create a new actor instance and register it with the passed in transport.
  /// The this will automatically allocate and store an `ActorAddress` for this instance.
  init(transport: ActorTransport) // TODO: check with Doug and co
  
  /// Transport using which messages to this actor are to be transported (if remote).
  var transport: ActorTransport { get } 
    
  /// The address, uniquely identifying this actor in a distributed setting.
  /// 
  /// An address typically will include a node/host pair, as well as some form of unique actor identifier.
  var address: ActorAddress { get }
    
  // Alternatively, a good spelling for this type would be:
  //     var context: DistributedActorContext { get } 
  // however the word "context" is higly overloaded, 
  // and we chose to instead go with the exploded form of 2 fields.
  //
  // If we went with context, we'd use up a popular name for fields, however acceses would read nice:
  //     context.transport: ActorTransport { get } 
  //     context.address: ActorAddress { get } 
}
```

Similarily to the `Actor` protocol, it is _not_ permitted to conform to this protocol manually.

> The Codable conformance *crucial* for the system to compose and actors work correctly in a distributed system. It's implementation is special and provided in the library itself. It is discussed in depth in the [Distributed Actors are Codable](#distributed-actors-are-codable) section.

The `transport` and `address` are immutable, and MUST NOT change during the lifetime of the specifc actor instance. Equality as well as messaging internals rely on this guarantee.

A `distributed actor` as well as extension's on it are the only places where `distributed func` declarations are allowed. This is because in order to implement a distributed function, a transport and identity (actor address) are necessary. It is not possible to declare free `distributed` functions since it would be unclear what this would mean. 

It is not allowed to define global actors which are distributed actors. If enough use-cases for this exist, we may losen up this restriction, however generally this is not seen as a strong use-case, and it is possible to add this capability in a source and binary compatible way in the future if necessary.

### Distributed Actors and "Location Transparency"

When an actor is declared using the distributed keyword, like so `distributed actor Greeter {}` it is referred to as a "distributed actor". References to distributed actors, at runtime, can be either "local" or a "remote":

- **local** `distributed actor` references: are semantically the same as a non-distributed `actor class` at runtime. They have the `transport` property and are `Codable` as their actor address (discussed below), however all isolation and execution semantics are exactly the same as plain-old local-only actor references.
- **remote** `distributed actor` references: on which invocations of `distributed func` are actually implemented as message sends over the stored `transport`. It is up to the transport and frameworks using this infrastructure to define what serialization and networking (or IPC) mechanism is used for the messaging. Semantically, it is indistinguishable from "just an actor," since in the actor model, all communication between actors occurs via asynchronous messaging.

It is by design, that by looking at a piece of code like this:

```swift
distributed actor Greeter {
  distributed func hello() async throws
} 

func greet(who greeter: Greeter) {
  await greeter.hello()
}
```

it is not _statically_ possible to determine if the actor is local or remote. This is hugely beneficial, as it allows us to write code generic over the location of such actors–i.e. we can write a complex distributed systems algorithm and test it locally with zero changes to the code, and deploying it to a cluster is merely a configuration and deployment change, without any additional code changes.

This property is often referred to as *Location Transparency* ([wiki](https://en.wikipedia.org/wiki/Location_transparency)), which means that we address resources only by their identity, and not their specific location. This enables distributed actors to be moved between local and remote nodes, have them passivate when not in use, and is a key building block to powerful actor based abstractions (in the vein of Virtual Actors, as popularized by Orleans and Akka).

> Future directions: We can in the future expose `isLocal`,or rather `withLocal`, functionality, allowing to dynamically determine if a distributed actor is local. This is rarely necessary however it may enable specific types of usages which otherwise look a bit awkward.

#### Progressive Disclosure towards Distributed Actors

The introduction of `distributed actor` is purely incremental, and does not imply any changes to the local programming model as defined by `actor class`. However, once developers understand the "share nothing" and "all communication through asynchronous functions (messages)," it is relatively simple to map the same understanding onto the distributed setting. 

None of the distributed systems aspects of distributed actors leak through to local-only actors, and developers who do not wish to use distributed actors, may simply ignore them.

Developers who first encounter a `distributed actor` in some API beyond their control can more easily learn about it if they already have seen actors in other pieces of their programs, since the same mental model applies to distributed as well as local-only actors. The big difference being the inclusion of serialization and networking, this however can be quickly understood with the general "it will be slower than a local call" intuition. This is no different from having _some_ asynchronous functions performing "very heavy" work (like sending HTTP requests by calling `httpClient.post(<file upload>)`) while some other async functions are relatively fast -- developers always need to reason about _what_ a function does in any case to understand performance characteristics. 

Swift's Distributed Actors help because we can explicitly mark such network interaction heavy objects as distributed actors, and therefore we know that distributed functions are going to use e.g. networking, so invoking it repeatedly loops may not be the best idea. Xcode and other IDEs can make use of this static information to even highlight distributed actor functions in some color, helping developers understand where exactly networking costs are to be expected.

### Distributed Actor Initializers

#### Initializing local distributed actors

All distributed actors automatically synthesize a special, required, initializer that accepts an `ActorTransport`. This initializer is _special_ and must not be overriden, and must be called into by all other constructors of such distributed actor. The synthesized initializer boils down to the following:

```swift
distributed actor Greeter {
  /* let actorTransport: ActorTransport*/
  /* @actorIndependent(unsafe) let address: ActorTransport */
  
  /* 
  init(transport actorTransport: ActorTransport) { 
    self.actorTransport = actorTransport
    self.address = actorTransport.allocate(self) // TODO: throwing?
  }
  */
} 
```

Customization of this initializers itself is _not_ allowed, however it is permittable to define other initializers which accept additional parameters etc. They all must eventually invoke this transport initializer. It is special because of the binding of the actor instance and the transport. Thanks to this guarantee, the actor transport _always_ knows about all instances it was asked to manage. This is important as it allows us to trust that any `resolve(address:)` performed by the transport, will correctly yield the apropriate actor reference, or throw.

Some actor types may be designed with the assumption of a specific transport. It is possible to either check and throw/fatalError in such cases in a auxiliary initializer. 

It is also possible to hardcode the transport used by the given distributed actor by implementing the `actorTransport` explicitly, like so:

```swift
distributed actor Greeter {
  let actorTransport = BestTransport.global
  
  // NOT SYNTHESIZED: init(transport: ActorTransport) throws { ... }
}
```

In which case all instances of such actor will use the `BestTransport`. This is useful when a project uses only a single transport, and never uses multiple transports within the same project. 

> (Pending discussions if actors allow inheritance or not): If actors are allowed to inherit other actors, this is possible to make use of in the following pattern, where a type may be defined such as `XPCDistributedActor` which overrides the transport for all sub-classes of such actor type to be the XPC Transport:
>
> ```swift
> distributed actor XPCDistributedActor { 
>   let actorTransport = XPCTransport.global
> }
> 
> distributed actor ImagesActor: XPCDistributedActor { }
> distributed actor FilesActor: XPCDistributedActor { }
> distributed actor BackgroundTasksActor: XPCDistributedActor { }
> ```
>
> 

While we warn against abusing this pattern as it makes it impossible to e.g. test code in a distributed-yet-on-the-same-host setting, which can be very useful in writing tests for distributed algorithms. However we acknowlage that the simplification from not having to always pass the transport to all distributed initializers is a nice win in smaller and not-so-distributed projects, e.g. if a project only ever uses distributed actors to communicate with it's deamon co-process using XPC.

#### Resolving local-or-remote Addresses

A special "resolve initializer" is synthesized for distributed actors. It is not implementable manually, and invokes internal runtime functionality for allocating "proxy" actors which is not possible to achieve in any other way.

A resolve initializer takes the shape of `init(resolve: ActorAddress using: ActorTransport)` 

```swift
distributed actor Greeter { 
  /* 
  init(resolve address: ActorAddress, using transport: ActorTransport) throws {
    switch try transport.resolve(address) {
    case .instance(let instance):
      self = instance
    case .proxy:
      self = <<make proxy to 'address' over 'transport'>>
    }
  }
  */
}
```

A resolve MAY throw when the transport decides that it cannot resolve the passed in address. A common example of a transport throwing would be if the address is for some protocol `"unknown://"` while the transport only can resolve `known://` actor addresses.

A transport MAY decide to return a "dead reference" meaning that the address currently points at an "already dead actor," and it may decide to instead of throwing, return a so-called "dead reference" (also known as "dead letters recipient" in some runtimes), whose only purpose is to receive all messages aimed to the original address and _log that they are being dropped_. This concept is very useful in debugging actor lifecycles, where we accidentally didn't keep the actor alive as long as we hoped etc. It is up to each specific transport to document, and implement, either behavior. 

A resolve initializer MAY transparently create an instance if it decides it is the right thing to do. This is how concepts like "virtual actors" may be implemented: we never actively create an actor instance, but it's creation and lifecycle is managed for us by some server-side component with which the transport communicates. Virtual actors and their specific semantics are outside of the scope of this proposal, but remain an important potential future direction of these APIs.

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


### (Distributed) Actor Equality

> This topic has come up in discussions as plain (local) actors were introduced to the Swift Evolution process already, however with distribution, we are able to focus on this point yet again, and clarify the confusion surrounding the topic.

Distributed actors are `Equatable` and `Hashable` by their _actor address_.

This may stirr opinions

Equality of actors _by address_ is tremendously important, because it enables us to "remember" actors in collections, look them up, and compare if an incoming actor reference is one which we have not seen yet, or is a previously known one.




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

A distributed function MUST have all it's parameters conform to `**Codable**`. It is legal for a distributed function to have zero parameters. 

All parameter and return value types of a distributed function must conform to **`Codable`**. 

It is allowed to declare **`Void` returning functions**, however it is up to the `ActorTransport` to decide (and document) the exact timing semantics and guarantees for when an await on such call will complete. I.e. some transports (e.g. IPC mechanisms) may actually perform a full roundtrip call before completing the waiting call (i.e. `await call()` would only complete once the callee has been called and completed). Other transports MAY treat these as uni-directional messages, and _not_ await a complete request/reply cycle before resuming the suspension point of such call. This is beneficial for typical uni-directional, at-most-once delivery semantics calls which sometimes come in handy in distributed systems. Such uni-directional calls also free the transport from having to perform any timeout and failure detection -- uni-directional sends MAY be implemented as best-effort uni-directional message send (e.g. the message is considered _sent_ once the message is put into a datagram (udp) and flushed, without the need for waiting for an acknowlagement). Having that said, the precise details of thise are up to the `ActorTransport ` to define, and document in detail, such that their users understand their semantics. 

In future we may consider some form `distributed(unidirectional) func` or similar spelling, which could inform both developer and transport code generators what is expected from this function.

All distributed functions must be declared as **`async throws`**  because the transport may need to decide to cancel a call, due to network issues, timeouts, or other transport issues. Errors thrown out of this function MAY be determined by the `ActorTransport`

Distributed function **invocations** are transformed by source generated actor transport frameworks into some specific encoding of the **message**. It is up to the `ActorTransport` framework to determine the disamiguation. Generally though messages are defined by their full signature, i.e. as expected from normal day to day programming, the following two declarations are expected to be encoded as different messages, and each cause an invocation of the apropriate function on the receiving (remote) actor:

```swift
distributed func greet(name: String) async throws -> String 
distributed func greet(who name: String) async throws -> String 
```

as with normal Swift code, it is required that the transport framework encodes the invocation of those functions in such way that they don't get "mixed up." To clarify, it is only the first parameter name which matters for the resolution, the second name should not matter, and be ignored by transport frameworks.

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

### Distributed Actor "Proxy" Instance Allocation

Creating a proxy for an actor type is done using a special `resolve(address:using:)` function, defined on the `ActorTransport` protocol:

```swift
protocol DistributedActor { 
    /// Creates new, local, instance 
    init(transport: ActorTransport)
}

protocol ActorTransport { 
  /// Resolve a local or remote actor address to a real actor instance, or throw if unable to.
  /// The returned value is either a local actor or proxy to a remote actor.
  func resolve<Act>(address: ActorAddress, as actorType: Act.Type) throws -> Act
		where Act: DistributedActor
}
```

This function can only be invoked on specific actor types–as usual with static functions on protocols–and serves as a factory function for actor proxies of given specific type. 

Implementing the resolve function by returning 

The returned value is of the same type as the invoked on actor, however it _may be proxy of it, delecating all calls to the passed in transport. The above functions are user-facing API, however it is not expected to be used directly by application developers. Instead, higher level abstractions such as clusters and de-serialization mechanisms (such as decoding an Actor passed in a Codable message) make use of this mechanism to turn addressess into actor references. 

Specifically, an example transport could implement a `resolve` function as follows

Note that the such resolved address may be on the same host as the function is invoked on, or on a different host.

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
  func resolve
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

#### Support for `AsyncSequence`

This isn't really somethign that the language will need much more support for, as it is mostly handled in the `ActorTransport` and serialization layers, however it is worth pointing out here.

It is possible to implement (and we have done so in other runtimes in the past), to implement distributed references to streams, which may be consumed across the network, including the support for flow-control/back-pressure and cancelation.

This would manifest in returning / accepting values conforming to the AsyncSequence or some more specific marker protocol. Distributed actors can then be used as coordinators and "sources" e.g. of metrics or log-lines across multiple nodes -- a pattern we have seen sucessfully applied in other runtimes in the past.

### Actor Supervision

Since this topic is bound to come up during review of this proposal, let us address it right away.

Supervision trees are a common pattern in actor systems where it is possible to monitor the lifetime of one actor by another, including over the network, and being able to detect network failures etc.

This proposal does not address any of these patterns however it is possible for a specific transport/runtime/framework to implement such mechanisms and add them as protocols, e.g. `DeathWatchable` that actors can adopt and replicate the functionality offered by such systems as OTP or Akka.

## Alternatives Considered

### Special Actor spawning APIs

One of the verbose bits of this API is that a distributed actor must be created or resolved from a specific transport. This makes sense and is _not_ accidental, but rather inherent complexity – because Swift applications often interact over various transports: within the host (e.g. a phone or mac) communicating via XPC with other processes, while at the same time communicating with server side components etc. It is important that it is clear and understandable what transport is used for what actor at construction.

#### Explicit `spawn(transport)` keyword-based API

Rather than provide the specialized `init(transport)` that every distributed actor must invoke, we could solve this using "compiler magic" (which we are usually trying to avoid), and introduce a form of `spawn` keyword. 

This goes against the current Actor proposal. Actors do not currently need to be prefixed with any special keywords when creating them. I.e. a local actor is simply created by constructing it: `Greeter()` rather than `spawn Greeter()` or similar.

In theory, we could require that a distributed actor must be spawned by `spawn(transport) Greeter()` and we would be able to hook up all internals of the distributed actor this way. Local actors could be spawned using `spawn Greeter()`. 

This would be fairly consistent and leaves a natural extension point for additional actor configuration in the spawn function (e.g. configuring an actor's executor at _spawn time_ could be done this way as well). However it is not clear if the core team would want to introduce yet another keyword for actor spawning, and specialize it this way. The burden of adding yet another keyword for this feature may be too high and not exactly worth it, as it only moves around where the transport/configuration passed: from their natural location in actor constructors, to special magical syntax.

#### Global eagerly initialized transport

One way to avoid having to pass a transport to every distributed actor on creation, would be to use some global state to register the transport to be used by all distributed actors in the process. This _seems_ like a good idea at first, but actually is a terrible idea - based on experience from moving an actor system implementation from global to non-global state over many years (during the Akka 1 to 2 series transition, as well as years of migrating off global state and problems caused by it by the Play framework).

The idea is similar in spirit to what SSWG projects do with bootstraping logging, metrics, and tracing systems: 

```swift
GlobalActorTransport.bootstrap(SpecificTransport())
// ... 
let greeter = DistributedGreeter()
```

While _at first glance_ this seems nice, the negative implications of such global state are numerous:

- It is hard to know by browsing the code what transprot the greeter will use,
  - if a transport were passed in via constructor (as implemented by this proposal) it is simpler to understand where the messages will be sent, e.g. via XPC, or some networking mechanism.
- The system would have to crash the actor creation if no transport is bootstrapped before the greeter is initialized.
- Since global state is involved, all actor spawns would need to take a lock when obtaining the actor reference. We would prefer to avoid such locking in the core mechanisms of the proposal.
- It encourages global state and makes testing harder; such bootstrap can only be called __once__ per process, making testing annoying. For example, one may implement an in-process transport for testing distributed systems locally; or simply configure different actors using different "nodes" even though they run in the same process. 

This global pattern discourages good patterns, about managing where and how actors are spawned and kept, thus we are not inclined to accepting any form of global transport.

### Directly adopt Akka-style Actors References `ActorRef<Message>`

Theoretically, distributed actors could be implemented as just a library, same as Akka does on the JVM. However this would be undermining both the distributed actors value proposition as well as the actual local-only actors provided by the language.

First, a quick refreshed how Akka models (distributed) actors: There are two API varieties, the untyped "classic" APIs, and the current (stable since a few years) "typed" APIs. 

Adopting a style similar to Akka's ["classic" untyped API](https://doc.akka.io/docs/akka/current/actors.html#here-is-another-example-that-you-can-edit-and-run-in-the-browser-) is a non-started for Swift - it is unacceptable for such a thighly typed language to come out and say all messages are handled as `Any` with zero typesafety at all. It is important to remember that this is where Akka _started_ more than 10 years ago, however it is _not_ it's current API.

Akka's typed actor API's represent actors as `Behavior<Message>` which is spawned, and then an `ActorRef<Message>` is returned from the spawn operation. The `Message` is a type representing, via sub-classing and immutable case classes, all possible messages this actor can reply to. Semantically this is equivalent to an `enum` in Swift. And we might indeed represent messages like this in Swift (even internally in an `ActorTransport` in the current proposal!) However, this model requires users to manually switch, destructure and handle much boilerplate related to wielding the types in the right ways so the model compiles and works properly. The typed API also heavily relies on Scala sugar for pattern matching within total and partial functions, allowing the expressions such as:

```scala
// scala
Behaviors.receiveMessage { 
  case "hello" => Behaviors.same
}
```

Having introduced the general ideas, let us imagine how this API would look like in Swift:

```swift
enum GreetMessage: Codable {
  case greet(who: ActorRef<String>, language: String)
}

// the actor behavior
let behavior: Behavior<Greet> = Behavior.setup { context in
  // state is represented by capturing it in behaviors
  var greeted = 0

  // actual receive function
  return .receiveMesage { message in 
    switch message {
    case .greet(let who, let language):
      greet(who: who, in: language)
    }
  }
                                                
  func greet(who: ActorRef<String>, language: String) { 
    greeted += 1
    localize("Hi \(who.address.name)!", in: "language")
  }
}

// spawning the actor and obtaining the reference
let ref: ActorRef<GreetMessage> = try system.spawn("greeter", behavior)
ref.tell(.greet(who: "Alice", in: "en")) // messages are used explicitly, so we create the enum values
```



```swift
distributed actor Greeter { 
  var greeted = 0
  distributed func greet(who: String, in language: String) async throws -> Greeting { 
    greeted += 1
    localize("Hi \(who)!", in: language)
  }
}

let greeter = Greeter(transport: ...)
let greeting = await greeter.greet(who: "Alice", in: "en")
```



There are **many** fantastic lessons, patterns and ideas developed by the Akka community–many of which inspire this proposal–however, it is limited in its expression power, because it is _just a library_. In contrast, this effort here is colaborative with the compiler infrastructure, including type checking, and thread-safety, concurrency model hooks and code generation built-into the language on various layers of the project. 



Needless to say, what we are able to achieve API wise and also because who the target audience of this project is, we are taking different tradeoffs in the API design - favoring a more language integrated model, for the benefit of feeling natural for the developers first discovering actor model programming. By doing so, we aim to provide a model that, as Akka has in the past, will mature well over the next 10 years, as systems become more and more distributed and programming such systems becomes more commonplace than ever before.

## Source compatibility

This change is purely additive to the source language. 

The additional use of the keyword `distributed` in `distributed actor class` and `distributed func` applies more restrictive requirements to the use of sucha actor class, however this only applies to new code, as such no existing code is impacted.

## Effect on ABI stability

This proposal is mostly additive.

**TODO:** The "proxy actor"... ???

## Effect on API resilience

TODO



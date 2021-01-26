### Distributed Actors

* Proposal: [SE-NNNN](NNNN-distributed-actors.md)
* Authors: [Konrad 'ktoso' Malawski](https://github.com/ktoso), [Dario Rexin](https://github.com/drexin), [Doug Gregor](https://github.com/DougGregor), [Tomer Doron](https://github.com/tomerd)
* Review Manager: TBD
* Status: **Implementation in progress**
* Implementation: [PR ####](https://github.com/ktoso/swift/tree/wip-distributed-actors-prime)

## Table of Contents

**TODO refresh it**

## Introduction

The recent proposal to introduce [actors](https://github.com/DougGregor/swift-evolution/blob/actors/proposals/nnnn-actors.md) to the Swift language allows benefit from the [actor model](https://en.wikipedia.org/wiki/Actor_model) when developing highly concurrent applications in Swift. By modeling various entities, such as devices, players, and any other fitting sub-system of an application as actors, we are able to easily communicate between those concurrent entities without having to worry too much about low level synchronization and concurrency issues.

This is already quite great, but we can do much better than that. The actor model does not stop there, and in fact it shines the brightest when taken to its natural next level: distribution.

At their core actors define a _general computational model_ relying only on the core concept of exchanging messages. Swift offers a convenient syntax for performing those messaging patterns in the form of asynchronous functions. However conceptually, this all still implements the semantics of message passing back and forth between fully isolated actors. Such isolation guarantees and message-exchange driven semantics are exactly what networks are! As far as the actor model is concerned, there is no difference if an actor is local, or remote.

In order to allow Swift developers to benefit from this unique value proposition of the actor model, this proposal introduces **Distributed Actors**, which can be used to represent actors located on remote processes or hosts. Various actor message transports can be implemented, offering developers the same consistent abstraction layer (actors) whether they are interacting with local actors or remote actors running on other processes, hosts, or even in fully-managed datacenters.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

After reading the proposal, we also recommend having a look at the [Related Proposals](#related-proposals), which may be useful to understand the big picture and how multiple Swift evolution proposals and libraries come together to support this proposal.



## Motivation

Most of the systems we write nowadays (whether we want it or not) are distributed.


For example:
- They might have multiple processes with the parent process using some IPC (inter-process communication) mechanism. 
- They might be part of a multi-service backend system that uses various networking technologies for communication between the nodes. 
- They might be a single service, but spread out across multiple nodes for availability and/or load-balancing reasons.
- They might be devices such as iPhones and iPads, communicating either between eachother for peer-to-peer games or interactive experiences, or communicating with a server-side system.

These use-cases all vary significantly and have very different underlying transports and mechanisms that enable them. Their implementations are also tremendously different. However, the general concept of wanting to communicate with non-local _identifiable_ entities is common in all of them. 

To keep things tangible, we will focus on the following two use-cases, with each having their unique requirements and influence on the design of this proposal:

- Network Actor Transports
  - Clustered server-side systems, simplifying implementation of scalable server-side systems and distributed algorithms
  - An alternative approach to RPC libraries, offering a more "Swifty" API while calling into known underlying RPC libraries
- IPC Actor Transports
  - On Apple platform's, XPC is the primary way applications communicate between daemons and apps. However, it lacks a truly great Swift API.
  - On non-Apple platforms various different IPC mechanisms exist, and they may be implemented very differently, but essentially they all offer the same request/reply semantics.

## Proposed solution

Distributed actors provide an extension point in Swift's actor runtime that enables actors as a general conceptual model for asynchronous communication, regardless if in-process or not.

Distributed programming is notoriously hard, and solutions offered in the space are often one-off solutions weaved throughout the application, often poluting the applications domain logic with concerns of serialization, networking, concurrency and failure handling. Distributed actors offer a way to centralize the complexity related to messaging in actor transports, and keep the domain logic of applications clean and focused on their business needs.

Actors isolate their state from the rest of the program, ensuring that interaction with the actor go through asynchronous calls, so the actor can serialize execution. Distributed actors take this isolation one step further, requiring all of arguments and results of such asynchronous calls to be serialized (via `Codable`), so they can be sent to another process or device. The introduction of other processes and devices necessitates a *transport mechanism* (described by the `ActorTransport` protocol) to ensure that the messages that represent an asynchronous call (or its result) are delivered to the appropriate actor. However, such mechanisms can fail–a node can go down, a process can get killed, a network connection can fail–so distributed actors need to cope with the possibility that an asynchronous call to a distributed actor might simply fail.

End-users and library developers can implement their own custom actor transports, which take care of transporting actor messages over various networking or IPC mechanisms. For example, it is possible to implement a websocket transport, or a custom protocol using TCP, or even implement one using XPC such that various actors spread out throughout XPC services can communicate with each other. 

This proposal _does not_ define any specific runtime--it is designed such that various (first and third-party) implementations of transports (`ActorTransport`s) allow distributed actors to adapt whichever communication patterns that suit a specific use-case.

### Distributed Actors

This proposal introduces the `distributed` contextual keyword, which may be used in conjunction with actor definitions (`distributed actor`), as well as `distributed func` declarations within such actors.

This keyword enables a few additional restrictions to what the typesystem is already checking in terms of actor isolation (more details below), and liberates those actors from their local affinity - allowing them to exist across process and network boundaries, fully embracing the message-passing nature of actors.

Instead of implementing time and time again the same logic around serializing/deserializing payloads and sending/receiving over the network as many of us do today, distributed actors allow us to encapsulate all of the serialization and message passing logic in a transport and regain our focus on the functional logic.

Let's imagine a `Player` actor (such as in the [SwiftShot](https://developer.apple.com/documentation/arkit/swiftshot_creating_a_game_for_augmented_reality) sample app from WWDC18) being expressed as a distributed actor. Instead of having the code related to serialization and sending actions that a player performs scattered across dozens of classes, we can capture the logic where it belongs, as part of the `Player` distributed actor:

```swift
distributed actor Player {
  var name: String

  distributed func grabAndThrow(item: Item, at: Target) async throws {
    print("Player \(name) grabbed \(item)")
    let grabbed = await item.grab()
    print("Player \(name) throwing \(item) at \(target)")
    await grabbed.throw(at: target)
  }
}

distributed actor Item { ... }
```

Compare `Player` to having to manually implement:

- defining "game action" enums which represent e.g. grabbing an item,
- declaring delegate protocols which get called when game actions happen,
- implementing encoding/decoding of the game actions to/from their wire representations,
- implementing logic to route incoming messages to the appropriate player.

### Location Transparency

When an actor is declared using the `distributed` keyword (`distributed actor Greeter {}`), it is referred to as a "distributed actor". At runtime, references to distributed actors can be either "local" or "remote":

- **local** `distributed actor` references: these are semantically the same as non-distributed `actor`s at runtime. They have the `transport` property and `Codable` as their actor address (discussed below), however all isolation and execution semantics are exactly the same as plain-old local-only actor references.
- **remote** `distributed actor` references: `distributed func`s invoked these are actually implemented as messages sent over the stored `transport`. It is up to the transport and frameworks using this infrastructure to define what serialization and networking (or IPC) mechanism are used for the messaging. Semantically, it is indistinguishable from "just an actor," since in the actor model, all communication between actors occurs via asynchronous messaging.

It is by design, that by looking at a piece of code like this:

```swift
distributed actor Greeter {
  distributed func hello() async throws
}

func greet(who greeter: Greeter) {
  await greeter.hello()
}
```

it is not _statically_ possible to determine if the actor is local or remote. This is hugely beneficial, as it allows us to write code independent of the location of the actors. We can write a complex distributed systems algorithm and test it locally. Deploying it to a cluster is merely a configuration and deployment change, without any additional code changes.

This property is often referred to as [*Location Transparency*](https://en.wikipedia.org/wiki/Location_transparency), which means that we address resources only by their identity, and not their specific location. This enables distributed actors to be moved between local and remote nodes, be passive when not in use, and is a key building block to powerful actor based abstractions (in the vein of Virtual Actors, as popularized by Orleans and Akka).

> Future directions: We can expose `isLocal` (or rather `withLocal`) functionality, which allows dynamically determine if a distributed actor is local. This is rarely necessary, but it may enable specific types of usages which otherwise look a bit awkward.

#### Progressive Disclosure towards Distributed Actors

The introduction of `distributed actor` is purely incremental, and does not imply any changes to the local programming model as defined by `actor`. However, once developers understand "share nothing" and "all communications are done through asynchronous functions (messages)", it is relatively simple to map the same understanding onto the distributed setting.

None of the distributed systems aspects of distributed actors leaks through to local-only actors. Developers who do not wish to use distributed actors may simply ignore them.

Developers who first encounter `distributed actor` in some API beyond their control can more easily learn about it if they have already seen actors in other parts of their programs, since the same mental model applies to distributed as well as local-only actors. The big difference is the inclusion of serialization and networking, but it can be quickly understood with the general "it will be slower than a local call" intuition. This is no different from having _some_ asynchronous functions performing "very heavy" work (like sending HTTP requests by calling `httpClient.post(<file upload>)`) while some other asynchronous functions are relatively fast--developers always need to reason about _what_ a function does in any case to understand performance characteristics.

Swift's Distributed Actors help because we can explicitly mark such network interaction heavy objects as `distributed actor`s, and therefore we know that distributed functions are going to use e.g. networking, so invoking them repeatedly in loops may not be the best idea. Xcode and other IDEs can make use of this static information to e.g. highlight distributed actor functions, helping developers understand where exactly networking costs are to be expected.

### Distributed Actor Proxies

In order to efficiently implement references to remote actors, we propose the introduction of proxy distributed actor instances.

A proxy can be understood like an "empty shell", that does not allocate any storage for the distributed actor's declared properties, because it represents a *remote actor*, and those properties never actually exist locally.

For example, given such distributed actor declaration:

```swift
distributed actor StatefulGreeter { 
  let greetings: [String] = [
    "Hello", "Cześć", "Hola", "こんにちは", "你好",
  ]
  
  distributed func greet(name: String) -> String {
    let hello = self.greetings.shuffled().first!
    return "\(hello), \(name)!"
  }
} 
```

In this example, only if the actor is *local* storage for the greetings (and the array itself) is allocated. If we have a reference to a `StatefulGreeter` that is *remote* the underlying object and its storage *does not* have any memory allocated to store the `greetings` property at all! As a reference to a remote actor, or rather a "proxy instance", the actual greetings and logic performed by this actor is *never* executed locally, but invoked remotely by the distributed actor transport layer, and its result is shipped back to the caller transparently.

We will discuss instantiating local and proxy actor instances in depth in the Detailed design section, however for now let's just say that depending on if the actor is remote, or local, it takes as little memory as possible to fulful its purpose. This can be visualized as follows:

```swift
                           [------------------- memory -------------------]
(local) distributed actor: [transport, address, ... stored properties ... ]
(proxy) distributed actor: [transport, address]
```

All while the actual functionality of the actor, and how we interact with it, remains *exactly the same*, thanks to the power of location transparency!

This is a very powerful technique, and allows us to program "heavy" actors which e.g. load up caches and other information when they start, and we need not ever concern ourselfes "only load caches if I'm the real actor" or similar patterns. Such caching patterns are tremendously useful in practice, and allow actors to become in-memory representations of real world entities, such as IoT devices, representations of players in a network game, or their own small hot-caches for an ongoing operation some user may be performing remotely. 

A popular use case for this is a "hot" representation of e.g. a shopping cart, where we keep an actor in memory to represent the card as long as the user remains on the check-out page (or on the site / in the app), as the actor stores all operations the user performs, we need not go back and forth to the database all the time when refreshing the page, we can use the actor as a form of specialized hot cache. If the user goes away, the actor can automatically passivate and flush changes to some actual persistent storage etc.

### Distributed Actor Functions

TODO:

- explain in depth what functions are

#### Transporting Errors

TODO

## Detailed design

### Distributed Actors

#### Distributed Actor Types

Distributed actors are declared using the `distributed actor` keywords, similar to local-only actors which are declared using only the `actor` keyword.

Similar to local-only actors which automatically conform to the `Actor` protocol, a type declared as `distributed actor` implicitly conforms to the `DistributedActor` protocol.

As with the `Actor` protocol, it is not possible to implement the `DistributedActor` protocol directly. The only way to conform to `DistributedActor` is to declare a `distributed actor`. It is not possible to declare any other type (struct, actor, class, ...) and make it conform to the `DistributedActor` protocol manually, for the same reasons as doing so is illegal for the `Actor` protocol: such a type would be missing additional type-checking restrictions and synthesized pieces which are necessary for distributed actors to function properly.

The distributed actor protocol is defined as:

```swift
public protocol DistributedActor: Actor, ... {

  // << Discussed in detail in "Local DistributedActor initializer"
  init(transport: ActorTransport)

  // << Discussed in detail in "Resolve initializer" >>
  init(resolve address: ActorAddress, using transport: ActorTransport)

  // << Discussed in detail in "Actor Transports" >>
  var actorTransport: ActorTransport { get }

  // << Discussed in detail in "Actor Address"
  var actorAddress: ActorAddress { get }
}
```

The `DistributedActor` protocol includes a few more conformances which will be covered in depth in their own dedicated sections, as we discuss the importance of the [actor address](#actor-address) property.

`actorTransport` and `actorAddress` are immutable, and *must not* change during the lifetime of the specific actor instance. Equality as well as messaging internals rely on this guarantee.

A `distributed actor` and extensions on it are the only places where `distributed func` declarations are allowed. This is because in order to implement a distributed function, a transport and identity (actor address) are necessary.

It is not possible to declare free `distributed` functions since it would be unclear what this would mean.

It is possible for a distributed actor to have non-distributed functions as well. They are callable only from two contexts: the actor itself (by `self.nonDistributedFunction()`), and from within an `maybeRemoteActor.whenLocalActor { $0.nonDistributedFunction() }` which will be discussed in ["Known to be local" distributed actors](#known-to-be-local-distributed-actors), although the need for this should be relatively rare.

It is not allowed to define global actors which are distributed actors. If enough use-cases for this exist, we may loosen up this restriction, however generally this is not seen as a strong use-case, and it is possible to add this capability in a source and binary compatible way in the future if necessary.

#### Distributed Actor Protocols

In some situations it may be impossible to share the implementation of a distributed actor (the `distributed actor` definition) between "server" and "client". We can imagine a situation where we want to offer users of our system easy access to it using distributed actors, however we do not want to share our internal implementation thereof. This works similarly to how one might want to publish API definitions, but not the actual API implementations. Other RPC runtimes solve this by externalizing the protocol definition into external interface description languages (IDLs), such as `.proto` files in the case of gRPC.

With Swift, we already have a great way to define protocols... protocols!

Distributed actor protocols, i.e. protocols which also conform to `DistributedActor`, are allowed to define distributed functions and can only be implemented by declaring a `distributed actor` conforming to such protocol.

For example, it is legal to define the following distributed actor protocol:

```swift
protocol Greeter: DistributedActor {
  distributed func greet(name: String) throws -> String
}
```

Such protocol can only define distributed functions and as it is strictly designated to define the distributed API of a distributed actor. 

> **Note:** (This feature is pending implementation, and requires synthesis of a "proxy instance"). A proxy instance for a distributed actor protocol can never implement non-distributed functions, however, since the proxy is always _remote_. It is impossible to invoke non-local instances of a proxy actor instantiated via the distributed protocol, as it will never be local, and thus `whenLocalActor { ...}` will never run (which would have allowed calling such un-implemented functions).

### Distributed Actor Initializers

#### Local `DistributedActor` initializer

All distributed actors automatically synthesize a required initializer that accepts an `ActorTransport`. This initializer is _special_ and must not be overridden, and must be called into by all other initializers of such distributed actor. The synthesized initializer boils down to the following:

```swift
distributed actor Greeter {
  /* ~~~ synthesized ~~~ */
  @derived let actorTransport: ActorTransport
  @derived @actorIndependent public let address: ActorAddress

  @derived init(transport actorTransport: ActorTransport) {
    self.actorTransport = actorTransport
    self.address = actorTransport.assignAddress(Self.self)
    // TODO: register specific instance with transport (!!!)
  }
  /* === synthesized === */
}
```

Customization of this initializer itself is _not_ allowed, however it is permittable to define other initializers which accept additional parameters etc. They all must eventually invoke this transport initializer. It is special because of the binding of the actor instance and the transport. Thanks to this guarantee, the actor transport _always_ knows about all instances it was asked to manage. This is important as it allows us to trust that any `resolve(address:)` performed by the transport will correctly yield the appropriate actor reference, or throw.

Some actor types may be designed with the assumption of a specific transport. It is possible to either check and `throw`/`fatalError` in such cases in an auxiliary initializer.

In which case all instances of such actor will use the `BestTransport`. This is useful when a project uses only a single transport, and never uses multiple transports within the same project.

While we warn against abusing this pattern as it makes it impossible to e.g. test code in a distributed-yet-on-the-same-host setting, which can be very useful in writing tests for distributed algorithms. However we acknowledge that the simplification from not having to always pass the transport to all distributed initializers is a nice win in smaller and not-so-distributed projects, e.g. if a project only ever uses distributed actors to communicate with it's daemon co-process using XPC.

#### Resolve Initializer

A special "resolve initializer" is synthesized for distributed actors. It is not implementable manually, and invokes internal runtime functionality for allocating "proxy" actors which is not possible to achieve in any other way.

A resolve initializer takes the shape of `init(resolve: ActorAddress, using: ActorTransport)`:

```swift
distributed actor Greeter {
  /* ~~~ synthesized ~~~ */
  @derived required 
  init(resolve address: ActorAddress, using transport: ActorTransport) throws {
    switch try await transport.resolve(address, as: Self.self) {
    case .instance(let instance):
      self = instance
    case .proxy:
      self = <<make proxy to 'address' over 'transport'>>
    }
  }
  /* === synthesized === */
}
```

A resolve may throw when the transport decides that it cannot resolve the passed in address. A common example of a transport throwing would be if the address is for some protocol `unknown://` while the transport only can resolve `known://` actor addresses.

The resolve initializer and related resolve function on the `ActorTransport` are _not_ `async` because they must be able to be invoked from decoding values, and the `Codable` infrastructure is not async-ready just yet. Also, for most use-cases they need not be asynchronous as the resolve is usually implemented well enough using local knowledge. In the future we might want to introduce an asynchronous variant of resolving actors which would simplify implementing transports as actors themselves, as well as enable more complicated resolve processes.

A transport may decide to return a "dead reference" instead of throwing when resolution fails. Such "dead reference" sometimes is useful when debugging lifecycle races, and allows to log messages intended for the already-dead  actor, rather than failing resolution, making it harder to debug the specific timing with respect to other messages which were meant to, but never had a chance to be delivered to their (dead) recipient. This pattern is entirely optional and most transports are expected to throw upon failing to resolve an address. The pattern has proven very useful in runtimes such as Akka which always employ this strategy when resolving actor addresses, so we wanted to acknowlage that it is possible to implement it using our transport proposal if necessary.

A resolve initializer may transparently create an instance if it decides it is the right thing to do. This is how concepts like "virtual actors" may be implemented: we never actively create an actor instance, but its creation and lifecycle is managed for us by some server-side component with which the transport communicates. Virtual actors and their specific semantics are outside of the scope of this proposal, but remain an important potential future direction of these APIs.

##### Resolve initializer for Distributed Actor protocols

> This feature may be scoped out from the initial implementation and followed up on soon after. 
>
> It is a cruicial feature to enable distributed server/client style applications, such as e.g. commercial cloud offerings offering services as a distributed actor protocol for clients to consume.

And a "client" side application, even without knowledge of how the distributed actor is implemented on the "backend" may resolve it as follows:

```swift
let address: ActorAddress = ... // known to point at a remote `Greeter`
let greeter: Greeter = try Greeter(resolve: address, using: someTransport)

let greeting = try await greeter.greet("Alice")
assert(greeting == "Hello, Alice!")
```

Such a resolved reference (i.e., `greeter`) *should* be a remote actor, since there is no local implementation the transport can invent to implement this protocol. We could imagine some transports using source generation and other tricks to fulfil this requirement, so this isn't stated as a must, however in any normal usage scenario the returned reference would be remote or the resolve should throw.

In other words, thanks to Swift's expressive protocols and isolation-checking rules applied to distributed functions and actors, we are able to use protocols as the interface description necessary to share functionality with other parties, even without sharing our implementations. There is no need to step out of the Swift language to define and share distributed system APIs with each other.

##### Distributed Actor "Proxy" instance allocation

Creating a proxy for an actor type is done using a special `init(resolve:using:)` initializer of a distributed actor. Internally, it invokes the transport's `resolve` function, which determines if the initializer will return a local instance, or if it should return a "proxy" instance.

```swift
protocol DistributedActor { 
    init(resolve address: ActorAddress, using transport: ActorTransport) throws { 
    	// ... synthesized ...
    }
}

protocol ActorTransport { 
  /// Resolve a local or remote actor address to a real actor instance, or throw if unable to.
  /// The returned value is either a local actor or proxy to a remote actor.
  func resolve<Act>(address: ActorAddress, as actorType: Act.Type) throws -> ResolvedDistributedActor<Act>
		where Act: DistributedActor
}
```

Implementing the resolve function by returning `.resolved(instance)` allows the transport to return known local actors it is aware of. Otherwise, if it intends to proxy messages to this actor through itself it should return `.proxy`, instructing the constructor to only construct a partial "proxy" instance using the address and transport. The transport may also chose to throw, in which case the constructor will rethrow the error, e.g. explaining that the passed in address is illegal or malformed.

A proxy instance does not allocate storage for any of the actor defined properties, i.e. it is a simple proxy that only takes as much memory as the transport and actor address take. In other words, a proxy instance is like an empty shell, delegating all calls to the remote incarnation of the actor, all those delegations are performed by transforming calls on the actor into messages, which the transport delivers to the remote actor.

The `resolve` function is intentionally not asynchronous, in order to invoke it from inside `decode` implementations, as they may need to decode actor addresses into actor references.

To see this in action, consider:

```swift
distributed actor Greeter { ... }
```

Which can be used to create a proxy, using the following invocation:

```swift
let greeter = try Greeter(address: someAddress, using: clusterTransport)
```

Some transports may choose to never throw if encountering an "unknown" address, and they may instead return a "dead letters" style object, which will log each message sent to such unresolved actor. The specifics of how a resolve works are left up to the transport though, as their semantics depend on the capabilities of the underlying protocols the transport uses.

### Distributed Functions

#### `distributed func` declarations

Distributed functions are a type of function which can be only defined inside a distributed actor (or an extension on such actor). 

They must be marked explicitly for a number of reasons, the first and most important one being that that they are subject to additional type-checking rules (discussed in detail in [Distributed Actor Isolation](#distributed-actor-isolation)). The explicit marking of functions intended to be distributed also helps frameworks which may need to generate sources (to be enabled by the extensible SwiftPM proposal) to support their transports by generating `$distributed_` function bodies. IDEs also benefit from the ability to understand that a specific function is distributed, and may want to color them differently or otherwise indicate that such function may have higher latency and should be used with care.

A distributed actor function is declared as follows:

```swift
distributed actor Greeter { 
  distributed func greet(name: String) async throws -> String { 
    "Hello, \(name)!"
  }
}
```

A distributed function *must* have all its parameters conform to `Codable`. It is legal for a distributed function to have zero parameters. 

All parameter and return value types of a distributed function must conform to `Codable`. 

It is allowed to declare `Void` returning distributed functions. It is up to the `ActorTransport` to decide (and document) the exact timing semantics and guarantees for when an `await` on such call will complete. i.e. some transports (e.g. IPC mechanisms) may actually perform a full roundtrip call before completing the waiting call (i.e. `await call()` would only complete once the callee has been called and completed). Other transports may treat these as uni-directional messages, and _not_ await a complete request/reply cycle before resuming the suspension point of such call. This is beneficial for typical uni-directional, at-most-once delivery semantics calls which sometimes come in handy in distributed systems. Such uni-directional calls also free the transport from having to perform any timeout and failure detection -- uni-directional sends may be implemented as best-effort uni-directional message send (e.g. the message is considered _sent_ once the message is put into a datagram (udp) and flushed, without the need for waiting for an acknowlagement). Having said that, the precise details are up to the `ActorTransport ` to define and document such that their users understand their semantics. 

In the future we may consider some form of `distributed(unidirectional) func` or similar, which could inform both developer and transport code generators what is expected from this function.

All distributed functions must be declared as **`async throws`**  because the transport may need to decide to cancel a call, due to network issues, timeouts, or other transport issues. Errors thrown out of this function may be determined by the `ActorTransport`.

Distributed function **invocations** are transformed by source generated actor transport frameworks into some specific encoding of the **message**. It is up to the `ActorTransport` to determine how to disambiguate. In general, messages are defined by their full signature. For example, the following two declarations are expected to be encoded as different messages, and each cause an invocation of the appropriate function on the receiving (remote) actor:

```swift
distributed func greet(name: String) async throws -> String 
distributed func greet(who name: String) async throws -> String 
```

As with normal Swift code, it is required that the transport framework encodes the invocation of those functions in such way that they don't get "mixed up." To clarify, it is only the first parameter name which matters for the resolution, the second name should not matter, and be ignored by transport frameworks.

#### `distributed func` internals

Developers implement distributed functions the same way as they would any other functions. This is a core gain from this model, as compared to external source generation schemes which force users to implement and provide so-called "stubs". Using the distributed actor model, the types we program with are the same types that can be used as proxies--there is no need for additional intermediate types.

A local (or "real") actor instance is a simple actor which implements all of its async functions by enqueueing the calls to its queue by the `Actor.enqueue(partialTask:)` function, conceptually this could be expressed as:

```swift
// local-actor pseudo-code (!)
func greet(name: String) async -> String { 
  let partialTask = __makeTaskRepresentingThisInvocation()
  self.enqueue(partialTask)
  return await partialTask.get()
}
```

Distributed actors are very much the same, just that instead of enqueueing a task into the local queue, they convert the call into a message and send it over the wire (using whichever transport is active for the actor). Local actors and distributed actors are much more similar to each other than one might think at first - the main difference is the addition of message creation (serialization) and instead of enqueue a transport mechanism is used to dispatch the message. 

The distributed actors design is purposefully detaching the implementation of such transport from the language, as we cannot and will not ship all kinds of different transport implementations as part of the language. Instead, what a distributed function does, is delegate to functions it assumes will exist at compile time, which are to be filled in by external code generation mechanisms (more on that in the coming sections).

```swift
// distributed actor pseudo-code (!)
func greet(name: String) async throws -> String { 
  if __isRemoteDistributedActor(self) {
    await try self.$distributed_greet(self.transport, name: name)
  } else {
    // actual logic
  }
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
  - in the distributed case: a `Codable` message representation to be sent over the wire to the recipient,
- enqueue the message
  - in the local case: on the actor's local mailbox / `queue`
  - in the distributed case: onto the transport, which will send it over the wire

We are not set on a naming scheme for the `$distributed_` functions just yet, and would appreciate naming feedback here, although it should be something simple, since it is what source code generators of transport frameworks will have to generate, so not much type magic can be required from them. We suggest a simple name prefix or similar.

### Distributed Actor Isolation

Distributed actor isolation inherits all the isolation properties of local-only actors, removes one local-only special-case and adds two additional restrictions to the model. In the following sections we will discuss them one-by one to fully grasp why they are necessary.

#### 1. No permissive special case for accessing constant `let` properties

Distributed actors remove the special case that exists for local-only actors, where access is permitted to such actor's properties as long as they are immutable `let` properties.

Local-only actors in Swift make a special case to permit _synchronous_ access to _constant properties_, e.g. `let name: String`. Since these cannot be modified, for the sake of concurrency safety such access is permissible. Such loosening of the actor model is _not_ permissible for distributed actors, because these properties are potentially remote, and any such access would have to be asynchronous and involve networking.

Specifically, the following is permitted under Swift's local-only actors model:

```swift
actor LocalGreeter { 
  let details: String // "constant"
}
LocalGreeter().name // ok
```

yet is illegal when the actor is distributed:

```swift
distributed actor Greeter { 
  let details: String // "constant", yet distributed actor isolated
}
Greeter(transport: ...).details // error: property 'details' is distributed actor-isolated
```

Alternatively, it could be argued that the loosening of the actor model restrictions should be removed entirely.

#### 2. `Codable` parameters and return values

All parameter and return types of a `distributed func` are required to be `Codable`. 

This helps developers do the right thing, and only send types which are intended for serialization. Note however, that a transport may have various limitations on what it allows to be sent -- it is left up to the distributed actor runtime to determine if and how to actually serialize a message. Including what (if any) `Encoder` to use when doing so.

```swift
distributed actor Greeter { 
  distributed func greet(name: String) async throws -> String {
    "Hello, \(name)"
  }
```

#### 3. Distributed functions must be `throws`

Currently, all distributed functions must be declared as `async throws`.

This is because remote calls may fail due to various reasons not present in single-process programming. Connections can fail, transport layer timeouts and heartbeats may signal that the call should be considered failed etc. This is in addition to the called function simply throwing and is mostly determined by the sender of the message independently of the callee which lives on a remote node. Specifically, remote actor transports often employ health checking protocols to ensure the called side is responsive, and if it is determined to be unavailable, all calls to all actors on such node may have to be considered failed.

In today's Swift the best way to express such failures is by the functions throwing an `ActorTransportError`.

This is sadly conflating networking issues with logical errors that a programmer may want to express in their APIs. Swift does not have a different way to model errors other than throws, and it would be a beneficial future direction to explore what "panics" or "soft faults" can look like, an "actor crashing" could clean up all of it's related memory.

Future direction considerations:

- If Swift were to embrace a more let-it-crash approach, which thanks to actors and the full isolation story of them seems achievable, we could consider building into the language a notion of crashing actors and "failures" which are separate from "errors". This is not being considered or pitched in this proposal, but it's worth keeping in mind. Further discussion on this can be found here: [Cleanup callback for fatal Swift errors](https://forums.swift.org/t/stdlib-cleanup-callback-for-fatal-swift-errors/26977)

### Actor Transports

Distributed actors are always associated with a specific transport that handles a specific instance.

A transport a protocol that distributed runtime frameworks implement in order to intercept and implement the messaging performed by a distributed actor.

The protocol is defined as:


```swift
protocol ActorTransport { 
  
  /// Resolve a local or remote actor address to a real actor instance, or throw if unable to.
  /// The returned value is either a local actor or proxy to a remote actor.
  func resolve<Act>(address: ActorAddress, as actorType: Act.Type) 
    throws -> ResolvedDistributedActor<Act>
    where Act: DistributedActor

  /// Create an `ActorAddress` for the passed actor type.
  ///
  /// This function is invoked by an distributed actor during its initialization,
  /// and the returned address value is stored along with it for the time of its
  /// lifetime.
  ///
  /// The address MUST uniquely identify the actor, and allow resolving it.
  /// E.g. if an actor is created under address `addr1` then immediately invoking
  /// `transport.resolve(address: addr1, as: Greeter.self)` MUST return a reference
  /// to the same actor.
  func assignAddress<Act>(forType: Act.Type, onActorCreated: (Act) -> ())  // FIXME: ??? that callback
    -> ActorAddress
    where Act: DistributedActor
  
  /// Invoked automatically by any distributed actor once it is deinitialized.
  /// 
  /// It allows the transport to free up any mappings it has stored from the address
  /// to the specific actor instance.
  ///
  /// The passed in address SHOULD be an address of an actor local to this transport,
  /// however the transport SHOULD NOT crash if an illegal or unknown address is passed.
  func resignAddress(address: ActorAddress)

//  func send<Message>(
//    _ message: Message,
//    to recipient: ActorAddress
//  ) async throws where Message: Codable

//  func request<Request, Reply>(
//    replyType: Reply.Type,
//    _ request: Request,
//    from recipient: ActorAddress
//  ) async throws where Request: Codable, Reply: Codable
}
```

A transport has two main responsibilities:

- creating and resolving actor addresses which are used by the language runtime to construct distributed actor instances,
- perform all message dispatch and handling on behalf of a distributed actor it manages, specifically:
  - for a remote distributed actor reference: 
    - be invoked by the framework's source generated `$distributed_function` implementations with a "Message" representation of the locally invoked function, serialize and dispatch it onto the network or other underlying transport mechanism. 
    - This turns local actor function invocations into messages put on the network.
  - for a local distributed actor instance: 
    - handle all incoming messages on the transport, decode and dispatch them to the appropriate local recipient instance. 
    - This turns incoming network messages into local actor invocations.

The first category can be seen as "lifecycle management" of distributed actors, and is interlinked with the Swift runtime and synthesized code of a distributed actor. The compiler will synthesize code to call out to the transport to implement some of its most crucial functionality:

- When a distributed actor is created using the local initializer (`init(transport:)`) the `transport.assignAddress(...)` function is invoked.
- When the actor is deinitialized, the transport is invoked with `transport.resignAddress(...)` with the terminated actor's address. 
- When creating a distributed actor using the resolve initializer (`init(resolve:using:)`) the Swift runtime invokes `transport.resolve(...)` asking the transport to decide if this address resolves as a local reference, or if a proxy actor should be allocated. Creating a proxy object is a Swift internal feature, and not possible to invoke in any way other than using a resolve initializer.

The second category is "actually sending/receiving the messages" which is highly dependent on the details of the underlying transport. We do not have to impose any API requirements on this piece of a transport actually. Since a distributed actor is intended to be started with a transport, and `$distributed_` functions are source generated by the same framework as the used transport, it can simply downcast the property to `MyTransport` and implement the message sending whichever way it wants. 

This way of dealing with message sending allows frameworks to use their specific data-types, without having to copy back and forth between Swift standard types and whichever types they are using. It would be helpful if we had a shared "bytes" type in the language here, however in general a transport may not even directly operate on bytes, but rather accept a `Codable` representation of the invoked function (e.g. an enum that is `Codable`) and then internally, depending on configuration, pick the appropriate encoder/decoder to use for the specific message (e.g. encoding it using a binary coder rather than JSON etc). By keeping this representation fully opaque to Swift's actor runtime, we also allow plugging in completely different transports, and we could actually invoke gRPC or other endpoints which use completely different serialization formats (e.g. protobuf) rather than the `Codable` mechanism. We don't want to prevent such use-cases from existing, thus opt to keep the "send" functions out of the `ActorTransport` protocol requirements. This is also good, because it won't allow users to "randomly" write `self.transport.send(randomMessage, to: address)` which would circumvent the typesafety experience of using distributed actors.

#### Transporting `Error`s

A transport _may_ attempt to transport errors back to the caller if it is able to encode/decode them, however this is _not encouraged_.

Instead, logic errors should be modelled by returning `Result<User, InvalidPassword>` as it allows for typed handling of such errors as well as automatically enforcing that the returned error type is also `Codable` and thus possible to encode and transport back to the caller. This is a good idea also because it forces developers to consider if an error really should be encoded or not (perhaps it contains large amounts of data, and a different representation of the error would be better suited for the distributed function).

Generally, it is encouraged to separate the "any failure" handling related to transport failures (such as timeouts or network errors), which are represented by the untyped `throws` of a distributed function call, from logical errors (e.g. "invalid password").

The exact error thrown by a distributed function depends on the underlying transport. Generally one should expect some form of `TheTransportError` to be thrown by a distributed function if a transport error occurs--the exact semantics of those throws are intentionally left up to specific transports to document when and how to expect errors to be thrown.

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

### Actor Address

A distributed actor's identity is defined by its `ActorAddress`.

The actor address is a protocol which an `ActorTransport` framework implements and must return when a distributed actor is initialized (see [Initializing local distributed actors](#initializing-local-distributed-actors)).

The `ActorAddress` is automatically used in the synthesized `Equatable` and `Codable` conformances of a distributed actor (see below).

The address can be seen as a form of URI, however we leave it up to transports to specify the exact formats they want to use. We require an address to start with a `transport://` identifier, such that the `address.protocol` can be used to determine which transport should be used for resolving an address if otherwise unknown.

```swift
public struct ActorAddress: Equatable, Codable {
  
  // TODO: just as a string and offer the getters host/port etc
  let address: String
  
  /// Uniquely specifies the actor transport and the protocol used by it.
  ///
  /// E.g. "xpc", "specific-clustering-protocol" etc.
  var `protocol`: String { get }

  // TODO: I guess easier to not define those at all on the protocol
//  var host: String? { get }
//  var port: Int? { get }
//  var path: String? { get }
//
//  /// Unique Identifier of this actor.
//  var uid: UInt64 { get }
}
```

An example address might be `actorid://#121212` if a transport is able to look up actors by plain uid identifiers. Or it might be more complete, like: `netactors://127.0.0.1:7337/workers/worker-1212#23232` which carries a lot of information in the address itself. 

An actor's address is automatically assigned to it at creation time (by the compiler invoking the `ActorTransport`'s `assignAddress` in the synthesized initializers discussed above) and stored in a synthesized `actorAddress` property:

```swift
distributed actor Greeter { 
  /* ~~~ synthesized ~~~ */
  let actorAddress: ActorAddress // initialized by init(transport:) or init(resolve:using:)
  /* === synthesized === */
}
```

Addressess are created by a specific `ActorTransport` which will usually implement them using a `struct` carrying any of the fields that are necessary for it to function. For example, a "Net Actors" framework would implement the protocol like this:

```swift
public struct NetActorAddress: ActorAddress { 
  static var `protocol`: String { "netactors" }
  let host: String // host of this cluster node
  let port: String // port of this cluster node
  // ...
  let uid: UInt64 // unique identifier of this actor
}
```

Again, the exact shape and fields of an actor address are completely up to the `ActorTransport` framework. Some may just need a single `uid`, others might need more host coordinates, such as shown above.

#### (Distributed) Actors are `Equatable` and `Hashable`

Distributed actors are `Equatable` and `Hashable` by their _actor address_.

> This topic has come up in discussions when plain (local) actors were introduced to the Swift Evolution process. With distribution however, we are able to focus on this point yet again, and clarify the confusion surrounding the topic.

Equality of actors _by address_ is tremendously important, because it enables us to "remember" actors in collections, look them up, and compare if an incoming actor reference is one which we have not seen yet, or is a previously known one. For example:

```swift
distributed actor Chatter {} 

distributed actor ChatRoom { 
  var members: Set<Chatter> = []
  
  distributed func join(chatter: Chatter) async throws -> String { 
    if members.insert(chatter).inserted { 
      return "Welcome!"
    } else {
      return "Welcome back!"
    }
  }
}
```

While this example is pretty simplistic, the ability to identify actors using their address is tremendously important for distributed actors. Unlike local-only actors, reference equality (`===`) is _not_ necessarily correct! This is because we need to be able to implement transports which simply return a new proxy instance whenever they are asked to resolve a remote instance, rather than forevermore return the same exact instance for them which would lead to infinite (!) memory growth including forever keeping around instances of remote actors which we never know for sure if they are alive or not, or if they ever will be resolved again. Therefore `ActorAddress` quality is the only reasonable way to implement equality _and_ is a crucial element to keep transport implementations memory efficient.

This is how the `==` and `hash` functions are synthesized by the compiler:

```swift
extension DistributedActor: Equatable, Hashable {  
  @actorIndependent
  public static func == (lhs: Self, rhs: Self) -> Bool {
    lhs.address == rhs.address
  }

  @actorIndependent
  public func hash(into hasher: inout Hasher) {
    self.address.hash(into: &hasher)
  }
}
```

If an implementation provides an equal or hash implementation, the synthesis will be disabled.

#### Distributed Actors are `Codable`

Distributed actors are `Codable` and are represented as their _actor address_  in their encoded form.

In order to be true to the actor model's idea of freely shareable references, regardless of their location (known as the property of [*location transparency*](#location-transparency)), we need to be able to pass distributed actor references to other--potentially remote--distributed actors.

This implies that distributed actors must be `Codable`. However, the way that encoding and decoding is to be implemented differs tremendously for actors and non-actors. Specifically, it does not make sense to serialize the actor's state - it is after all what is isolated from the outside world and from external access.

The `DistributedActor` protocol also conforms to `Codable`. As it does not make sense to encode/decode "the actor", per se, the actor's encoding is specialized to what it actually intends to express: encoding an address, that can be resolved on a remote node, such that the remote node can contact this actor. This is exactly what the `ActorAddress` is used for, and thankfully it is an immutable private property of each actor, so the synthesis of Codable of a distributed actor boils down to encoding its' address:

```swift
extension DistributedActor {
  /* ~~~ synthesized ~~~ */ 
  @actorIndependent
  @derived func encode(to encoder: Encoder) throws { 
    var container = encoder.singleValueContainer()
    try container.encode(self.actorAddress)
  }
  /* === synthesized === */
}
```

Decoding is slightly more involved, because it must be triggered from _within_ an existing transport. This makes sense, since it is the transport's internal logic which will receive the network bytes, form a message and then turn to decode it into a real message representation before it can deliver it to the recipient. 

In order for decoding of an Distributed Actor to work on every layer of such decoding process, a special `CodingUserInfoKey.actorTransport` is used to store the actor transport in the Decoder's `userInfo` property, such that it may be accessed by any decoding step in a deep hierarchy of encoded values. If a distributed actor reference was sent as part of a message, this means that it's `init(from:)` will be invoked with the actor transport present. 

The default synthesized decoding conformance can therefore automatically, without any additional user intervention, decode itself when being decoded from the context of any actor transport. The synthesized initializer looks roughly like this:

```swift
extension DistributedActor {
  /* ~~~ synthesized ~~~ */ 
  @derived init(from decoder: Decoder) throws {
    guard let transport = self.userInfo[.actorTransport] as? ActorTransport else {
      throw DistributedActorDecodingError.missingTransportUserInfo(Self.self)
    }
    let container = try decoder.singleValueContainer()

    let address = try container.decode(ActorAddress.self)
    self = try Self(resolve: address, using: transport)
    // FIXME: hitting the limitation
    // 			  https://forums.swift.org/t/allow-self-x-in-class-convenience-initializers/15924
    // need to use a workaround...
  }
  /* === synthesized === */
}
```

During decoding of such reference, the transport gets called again and shall attempt to `resolve` the actor's address. If it is a local actor known to the transport, it will return its reference. If it is a remote actor, it will return a proxy instance pointing at this address. If the address is illegal or unrecognized by the transport, this operation will throw and fail decoding the message.

To show an example of what this looks like in practice, we might implement an actor cluster transport, where the actor addresses are a form of URIs, uniquely identifying the actor instance, and as such encoding an actor turns it into `"[transport-name]://system@10.0.0.1:7337/Greeter#349785`, or using some other encoding scheme, depending on the transport used.

It is tremendously important to allow passing actor references around, across distributed boundaries, because unlike local-only actors, distributed actors can not rely on passing a closure to another actor to implement "call this later, please" style patterns. We are not, and do not want to, venture into the world of serializing and sending around closures. 

Instead, such patterns must be expressed by passing the `self` of a distributed actor (potentially as a `DistributedActor` bound protocol) to another distributed actor, such that it may invoke it whenever necessary. E.g. publish/subscribe patterns implemented using distributed patterns need this capability:

```swift
distributed actor PubSubPublisher {
  var subs: Set<AnySubscriber<String>> = []
  
  /// Subscribe a new distributed actor/subscriber to this publisher
  distributed func subscribe<S>(subscriber: subscriber)
    where S: SimpleSubscriber, S.Value == String {
    subs.insert(AnySubscriber(subscriber))
  }
  
  /// Emit a value to all subscribed distributed actors
  distributed func emit(_ value: String) async throws { 
    for s in subs {
      try await s.onNext(value)
    }
  }
}
```

```swift
protocol SimpleSubscriber: DistributedActor {
  associatedtype Value: Codable
  func onNext(_ value: Value) async throws
}

distributed actor PubSubSubscriber: Subscriber { 
	typealias Value = String
  
  let publisher: PubSubPublisher = ...
  
  func start() async throws { 
    try await publisher.subscribe(self) // `self` is safely encoded and sent remotely to the publisher
  }
  
  /// Invoked every time the publisher has some value to emit()
  func onNext(_ value: Value) { 
    print("Received \(value) from \(publisher)!")
  }
}
```

The above snippet showcases that with distributed actors, and codable DistributedActor types it is trivial to implement a simple pub-sub style publisher, and of course this ease of development extends to other distributed systems issues. The fundamental building blocks being a natural fit for distribution that "just click" are of tremendous value to the entire programming style with distributed actors. Of course a real implementation would be more sophisticated in its implementation, but it is a joy to look at how distributed actors make mundane and otherwise difficult distributed programming tasks simple and understandable.

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

The `ActorTransport` will be explained in depth in the detailed design sections, in general though the transport is to be considered an actor local value, which is not intended to be shared with other actors (i.e. unmovable). While the `self.transport.address` is possible to be shared with other actors, or even stored by them. It is a common pattern to store a set of known actors or addresses, in order to maintain a "watched" set of actors, which we can use to check incoming addresses against--is this message from a "new" actor, or one which I already have seen and stored in my "watched" set? These kinds of situations come up frequently in distributed system programming, and it is useful to be able to store addresses, with detachment to any actual reference to the real remote actor.

### "Known to be local" distributed actors

Usually programming with distributed actors means assuming that an actor may be remote, and thus only `Codable` parameters/return types may be used with it. This is a sound and resilient model, however it sometimes becomes an annoying limitation when an actor is "known to be local".

This situation sometimes occurs when developing a form of manager actor which always has a single local instance per cluster node, but also is reachable by other nodes as a distributed actor. Since the actor is defined as distributed, we can only send messages which can be encoded to it, however sometimes such APIs have a few specialized local functions which make sense locally, but are never actually sent remotely. We do want to have these local-only messages to be handled by exactly the same actor as the remote messages, to avoid race conditions and accidental complexity from splitting it up into multiple actors.

A specific example of such pattern is the [CASPaxos protocol](https://arxiv.org/abs/1802.07000), which is a popular distributed consensus protocol which performs a distributed compare-and-set operation over a distributed register. It's API accepts a `change` function, which naturally we cannot (and do not want to) serialize and send around the cluster, however the local proposer wants to accept such function:

```swift
public distributed actor Proposer<Value: Codable> { 
  public func change(
    key: String, 
    update: (Value?) throws -> Value
  ) async throws -> Value { /* ... */ }
}
```

Leaving the exact implementation details aside, such APIs sometimes arise and can either be addressed by wrapping e.g. the closure in some `NotActuallyCodable { value in ... }` wrapper (similar to how `UnsafeTransfer` is proposed in raw concurrency isolation and the `ConcurrentValue` design documents) which pretends to be `Codable` but actually crashes if one were to actually try encoding it. 

This is sub-optimal because technically, we can make mistakes and accidentally invoke such functions on an actor that actually was remote, causing the process to crash. Rather we would like to express the assumption directly: "*assuming this actor is local, I want to be able to invoke it using the local actor rules, without the restrictions imposed by the additional distributed actor checking*".

We could offer a function to inspect and perform actions when the actor reference indeed is local like this:

```swift
extension DistributedActor { 
  @actorIndependent
  public func whenLocalActor(
    _ body: (actor Self) async throws -> T,
    whenRemote remoteBody: ((Self) async throws -> T)? /* = nil */
  ) -> (re)async rethrows -> T?
}
```

The `whenRemote` optional body can be used to provide an alternative or default value when the actor actually *is* indeed remote.

Which can be used like this:

```swift
distributed actor Greeter { func tell(_ message: String) { print(message) } }
let greeter: Greeter = Greeter(transport: someTransport)
let greeter2: Greeter = try Greeter(resolve: address, transport: someTransport)

greeter.whenLocalActor { greeterWasLocal in 
  greeterWasLocal.tell("It is local, after all!")
}

assert(greeter == greeter2)
assert(greeter === greeter2)
```

And thanks to the [forward-scaning of trailing closures](https://github.com/apple/swift-evolution/blob/main/proposals/0286-forward-scan-trailing-closures.md#proposed-solution), a convenient syntax to use the whenRemote branch emerges:

```swift
let accessibleOnlyLocally = greeter.whenLocalActor { local in 
  local.localOnly // e.g. some let constant
} whenRemote { remoteAfterAll in
  nil 
}
```

This allows us to keep using the actor as isolation and "linearization" island, keep the distributed protocol implementations simple and not suffer from accidental complexity.

This API should be seen as somewhat of a "backdoor", and APIs should not abuse it without need, however we acknowlage these situations happen in the real world, and would like to offer a clean solution. 

> For comparison, when this situation happens with other runtimes such as Akka the way around it is to throw exceptions when "not intended for remoting" messages are sent. It is possible to determine if an actor is local or remote in such runtimes, and it is used in some low-level orchestration and internal load balancing implementations, e.g. selecting 1/2 of local actors, and moving balancing them off to another node in the cluster etc.

The implementation of the `isLocalActor` function is a trivial check if the `isDistributedActor` flag is set on the actor instance, and therefore does not add any additional storage to the existing actor infrastructure since actors already have such flags property used for other purposes.

### Future Directions

#### Swift Distributed Tracing integration

With the recent release of [Swift Distributed Tracing](https://github.com/apple/swift-distributed-tracing) we made first steps towards distributed tracing becoming native to server-side swift programs. This is not the end-goal however, we want to enable distributed tracing throughout the Swift ecosystem, and by ensuring tracers can natively, and easily inter-op with distributed actors we lay down further ground work for this vision to become reality.

Note that distributed traces also mean the ability to connect traces from devices, http clients, database drivers and last but not least distributed actors into a single visualizable trace, similar to how Instruments is able to show execution time and profiles of local applications. With distributed tracing we have the ability to eventually offer such "profiler like" experiences over multiple actors, processes, and even front-end/back-end systems.

Thanks to [SE-NNNN: **Task Local Values**](https://github.com/apple/swift-evolution/pull/1245) `ActorTransport` authors can utilize all the tracing and instrument infrastructure already provided by the server side work group and instrument their transports to carry necessary trace information.

Specifically, since the distributed actor design ensures that the transport is called in the right place to pick up the appropriate values, and thus can propagate the trace information using whichever networking protocol it is using internally:

```swift
// simplified proxy implementation generated by a transport framework
func $distributed_exampleFunc() async throws {
  var message: MessageRepr = .init(name: "exampleFunc") // whatever representation the framework uses
  
  if let traceID = Task.local(\.traceID) { // pick up tracing `Baggage` or any specific values and carry them
    message.metadata.append("traceID", traceID) // simplified, would utilize `Injector` types from tracing
  }
  
  try await self.actorTransport.send(message, to: self.actorAddress)
}
```

Using this, trace information and other information (see below where we discuss distributed deadlines), is automatically propagated across node boundaries, and allows sophisticated tracing and visualization tools to be built.

If you recall the `makeDinner` example from the [Structured Concurrency proposal](https://github.com/DougGregor/swift-evolution/blob/structured-concurrency/proposals/nnnn-structured-concurrency.md#motivation), we can use tracing to visualize the execution of the variour tasks involed, and we can even have some pieces of it execute on remote hosts. For example, let's say all chopping of vegetables is performed on a remote host by a distributed actor. The entire work can be then automatically visualized as:

```
>-o-o-o----- makeDinner ----------------o---------------x      [15s]
  | | |                     | |         |                  
~~~~~~~~~ ChoppingService ~~|~|~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  \-|-|- chopVegetables-----x |                            [2s]     \
    | |  \- chop -x |         |                        [1s]         | Executed on different host (!)
    | |             \- chop --x                        [1s]         /
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    \-|- marinateMeat -----------x      |                  [3s]
      \- preheatOven -----------------x |                 [10s]
                                        \--cook---------x  [5s]
```

Specific tracing systems may offer actual fancy nice UIs to visualize this rather than ASCII art.

#### Distributed `Deadline` propagation

The concurrency proposals initially also included the concept of deadlines, which co-operate with task cancellation in the task runtime.

Distributed actors as well as Swift Distributed Tracing

> This pattern is well-known and has proven most useful as proven by its wide application in effectively _all_ distributed Go systems, where the `Context` type automatically carries a deadline value, and automatically causes cancellation of a context if the deadline is exceeded.

Example use-case:

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

#### Potential Transport Candidates

While this proposal intentionally does not introduce any specific transport, the obvious reason for introducing this feature is implementing specific actor transports. This proposal would feel incomplete if we would not share our general thoughts about which transports would make sense to be implemented over this mechanism, even if we cannot at this point commit to specifics about their implementations.

It would be very natural, and has been considered and ensured that it will be possible by using these mechanism, to build any of the following transports:

- clustering and messaging protocols for distributed actor systems, e.g. like [Erlang/OTP](https://www.google.com/search?q=erlang) or [Akka Cluster](https://doc.akka.io/docs/akka/current/typed/cluster-concepts.html).
- [inter-process communication](https://en.wikipedia.org/wiki/Inter-process_communication) protocols, e.g. XPC on Apple platforms or shared-memory.
- various other RPC-style protocols, e.g. the standard [XML RPC](http://xmlrpc.com), [JSON RPC](https://www.jsonrpc.org/specification) or custom protocols with similar semantics.
- it should also be possible to communicate with WASM and "Swift in the Browser" using distributed actors and an appropriate websocket transport.

#### Support for `AsyncSequence`

This isn't really something that the language will need much more support for, as it is mostly handled in the `ActorTransport` and serialization layers, however it is worth pointing out here.

It is possible to implement (and we have done so in other runtimes in the past), distributed references to streams, which may be consumed across the network, including the support for flow-control/back-pressure and cancelation.

This would manifest in returning / accepting values conforming to the AsyncSequence or some more specific marker protocol. Distributed actors can then be used as coordinators and "sources" e.g. of metrics or log-lines across multiple nodes -- a pattern we have seen sucessfully applied in other runtimes in the past.

#### Ability to hardcode actors to specific shared transport

We are not sure if this extension pulls its own weight. It depends on the usage patterns that end up most common, however we envision XPC to be a common use case where a single shared transport may be useful... In the current design a transport must always be passed explicitly to a distributed actor during initialization.

We can imagine specific transports, or projects, which know that they only use a specific shared transport in the entire application, and may avoid this initialization boilerplate. This would be possible if we tweak synthesis to allow and respect properties initialized in their field declarations, like this:

```swift
protocol XPCActor: DistributedActor { 
  var actorTransport: ActorTransport { XPCTransport.shared }
}

distributed actor BackgroundDaemon: XPCActor { 
 // NOT synthesized:
 // init(transport:)
 // init(resolve:using:) throws
 
 /* ~~~ synthesized instead ~~~ */
 @derived init()
 @derived init(resolve:) throws
 /* === synthesized instead === */
}
```

This kind of loops back to allowing actor inheritance or not; If we do then the `init(transport:)` initializer must be `required` complicating the model and we may need to rely on synthesis magic.

We also may want to consider if it makes sense to allow the local `init()` to throw or not.

#### Actor Supervision

Since this topic is bound to come up during review of this proposal, let us address it right away.

Supervision trees are a common pattern in actor systems where it is possible to monitor the lifetime of one actor by another, including over the network, and being able to detect network failures etc.

This proposal does not address any of these patterns however it is possible for a specific transport/runtime/framework to implement such mechanisms and add them as protocols, e.g. `DeathWatchable` that actors can adopt and replicate the functionality offered by such systems as OTP or Akka.



## Related Proposals

- **[Swift Concurrency Manifesto](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782#design-sketch-for-interprocess-and-distributed-compute)** - distributed actors as part of the language were first roughly outlined as a general future direction in the Concurrency Manifesto. The approach specified in this proposal takes inspiration from the manifesto, however may be seen as a reboot of the effort. We have invested a significant amount of time, research, prototyping and implementing the approach since, and are confident in the details of the proposed model.

**Pre-requisites**

- [SE-0296: **async/await**](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md) - asynchronous functions are used to express distributed functions,
- [SE-NNNN: **Actors**](https://forums.swift.org/t/pitch-2-actors/44094) - are the basis of this proposal, as it refines them in terms of a distributed actor model by adding a few additional restrictions to the existing isolation rules already defined by this proposal,

**Related**

- [SE-NNNN: **Task Local Values**](https://github.com/apple/swift-evolution/pull/1245) - are accessible to transports, which use task local values to transparently handle request deadlines and implement distributed tracing systems (this may also apply to multi-process instrumentation),
- [SE-0295: **Codable synthesis for enums with associated values**](https://github.com/apple/swift-evolution/blob/main/proposals/0295-codable-synthesis-for-enums-with-associated-values.md) - because distributed functions relying heavily on `Codable` types, and runtimes may want to express entire messages as enums, this proposal would be tremendously helpful to avoid developers having to drop down to manual Codable implementations.

**Follow-ups**

- **SwiftPM Extensible build-tools** – which will enable source-code generators, necessary to fill-in distributed function implementations by specific distributed actor transport frameworks.

**Related work**

- [Swift Cluster Membership](https://www.github.com/apple/swift-cluster-membership) ([blog](https://swift.org/blog/swift-cluster-membership/)) – cluster membership protocols are both natural to express using distributed actors, as well as very useful to implement membership for distributed actor systems.
- [Swift Distributed Tracing](https://github.com/apple/swift-distributed-tracing) – distributed actors are able to automatically and transparently participate in distributed tracing systems and instrumentation, this allows for a "profiler-like" performance debugging experience across entire fleets of servers,



## Alternatives Considered

### Infer that "`distributed func`" automatically is `throws`

Since distributed functions today are required to be throwing, one could ask why not make them throwing automatically?

Firstly, we want to make it simple for readers of code to understand what they can and cannot do in such functions, and having the `throws` explicit is helpful for devleopers encountering a distributed function and actor the first time.

One might argue that, similar to `async` being infered on actor functions _if called from the outside_ throws could be inferred the same way if called externally. However we do not want to weave too much magically infering things into those APIs, especially when in the future we might end up allowing non throwing declarations, if swift were to gain e.g. panics or soft faults which are more natural to model the "network failure" issues that are the reason to _require_ all distributed functions to be throwing.

### Discussion: Why Distributed Actors are better than "just" some RPC library?

While this may be a highly subjective and sensitive topic, we want to tackle the question up-front, so why are distributed actors better than ""just" some RPC library?

The answer lies in the language integration and the mental model developers can work with when working with distributed actors. Swift already embraces actors for its local concurrency programming, and they will be omni-present and become a familiar and useful tool for developers. It is also important to notice that any async function may be technically performing work over the network, and it is up to developers to manage such calls in order to not overwhelm the network, etc. With distributed actors, such calls are more _visible_ because IDEs have the necessary information to underline or otherwise highlight that a function is likely to hit the network and one may need to consider it's latency more than if it was just a local call. IDEs and linters can even use this statically available information to write hints such as "hey, you're doing this distributed actor call in a tight loop - are you sure you want to do that?"

Distributed actors, unlike "raw" RPC frameworks, help developers to think about their distributed applications in terms of a network of collaborating actors, rather than having to think and carefully manage every single serialization call and network connection management between many connected peers - which we envision to be more and more important in the future of device and server programming et al. You may also refer to the [Swift Concurrency Manifesto; Part 4: Improving system architecture](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782#part-4-improving-system-architecture) section on some other ideas on the topic.

This does _not_ mean that we shun RPC style libraries or plain-old HTTP clients and libraries similar to them, which may rather be expressed as non-actor types with asynchronous functions. They still absolutely have their place, and we do not envision distributed actors fully replacing them in all use-cases. We do mean however that extending the actor model to it's natural habitat (networking) will enable developers to build some kinds of interactive multi-peer/multi-node systems far more naturally than each time having to re-invent a similar abstraction layer, never quite reaching the integration smoothness as language provided integration points such as distributed actors can offer.

### Special Actor spawning APIs

One of the verbose bits of this API is that a distributed actor must be created or resolved from a specific transport. This makes sense and is _not_ accidental, but rather inherent complexity – because Swift applications often interact over various transports: within the host (e.g. a phone or mac) communicating via XPC with other processes, while at the same time communicating with server side components etc. It is important that it is clear and understandable what transport is used for what actor at construction.

#### Explicit `spawn(transport)` keyword-based API

Rather than provide the specialized `init(transport)` that every distributed actor must invoke, we could solve this using "compiler magic" (which we are usually trying to avoid), and introduce a form of `spawn` keyword. 

This goes against the current Actor proposal. Actors do not currently need to be prefixed with any special keywords when creating them. I.e. a local actor is simply created by constructing it: `Greeter()` rather than `spawn Greeter()` or similar.

In theory, we could require that a distributed actor must be spawned by `spawn(transport) Greeter()` and we would be able to hook up all internals of the distributed actor this way. Local actors could be spawned using `spawn Greeter()`. 

This would be fairly consistent and leaves a natural extension point for additional actor configuration in the spawn function (e.g. configuring an actor's executor at _spawn time_ could be done this way as well). However it is not clear if the core team would want to introduce yet another keyword for actor spawning, and specialize it this way. The burden of adding yet another keyword for this feature may be too high and not exactly worth it, as it only moves around where the transport/configuration passed: from their natural location in actor constructors, to special magical syntax.

#### Global eagerly initialized transport

One way to avoid having to pass a transport to every distributed actor on creation, would be to use some global state to register the transport to be used by all distributed actors in the process. This _seems_ like a good idea at first, but actually is a terrible idea - based on experience from moving an actor system implementation from global to non-global state over many years (during the Akka 1 to 2 series transition, as well as years of migrating off global state and problems caused by the Play framework).

The idea is similar in spirit to what SSWG projects do with bootstraping logging, metrics, and tracing systems: 

```swift
GlobalActorTransport.bootstrap(SpecificTransport())
// ... 
let greeter = DistributedGreeter()
```

While _at first glance_ this seems nice, the negative implications of such global state are numerous:

- It is hard to know by browsing the code what transport the greeter will use,
  - if a transport were passed in via constructor (as implemented by this proposal) it is simpler to understand where the messages will be sent, e.g. via XPC, or some networking mechanism.
- The system would have to crash the actor creation if no transport is bootstrapped before the greeter is initialized.
- Since global state is involved, all actor spawns would need to take a lock when obtaining the actor reference. We would prefer to avoid such locking in the core mechanisms of the proposal.
- It encourages global state and makes testing harder; such bootstrap can only be called __once__ per process, making testing annoying. For example, one may implement an in-process transport for testing distributed systems locally; or simply configure different actors using different "nodes" even though they run in the same process. 

This global pattern discourages good patterns, about managing where and how actors are spawned and kept, thus we are not inclined to accepting any form of global transport.

#### Directly adopt Akka-style Actors References `ActorRef<Message>`

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

The additional use of the keyword `distributed` in `distributed actor` and `distributed func` applies more restrictive requirements to the use of such an actor, however this only applies to new code, as such no existing code is impacted.

Marking an actor as distributed when it previously was not is potentially source-breaking, as it adds additional type checking requirements to the type.

## Effect on ABI stability

This proposal is ABI additive.

**TODO:** The "proxy actor"... ???

## Effect on API resilience

**TODO:**

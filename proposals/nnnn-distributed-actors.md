# Distributed Actors

* Proposal: [SE-NNNN](NNNN-distributed-actors.md)
* Authors: [Konrad 'ktoso' Malawski](https://github.com/ktoso), [Dario Rexin](https://github.com/drexin), [Doug Gregor](https://github.com/DougGregor), [Tomer Doron](https://github.com/tomerd)
* Review Manager: TBD
* Status: **Implementation in progress**
* Implementation: 
  * Partially available in [recent `main` toolchain snapshots](https://swift.org/download/#snapshots) behind the `-enable-experimental-distributed` feature flag. 
  * This flag also implicitly enables `-enable-experimental-concurrency`.

## Table of Contents

<!--ts-->
* [Distributed Actors](#distributed-actors)
   * [Table of Contents](#table-of-contents)
   * [Introduction](#introduction)
   * [Motivation](#motivation)
   * [Proposed solution](#proposed-solution)
      * [Distributed actors](#distributed-actors-1)
      * [Distributed functions](#distributed-functions)
      * [Distributed actor transports](#distributed-actor-transports)
      * [Fundamental principle: Location Transparency](#fundamental-principle-location-transparency)
   * [Detailed design](#detailed-design)
      * [Distributed Actors](#distributed-actors-2)
         * [The DistributedActor protocol](#the-distributedactor-protocol)
            * [DistributedActor is not an Actor](#distributedactor-is-not-an-actor)
            * [Optional: The AnyActor marker protocol](#optional-the-anyactor-marker-protocol)
         * [Progressive Disclosure towards Distributed Actors](#progressive-disclosure-towards-distributed-actors)
      * [Distributed Actor initialization](#distributed-actor-initialization)
         * [Local initializers](#local-initializers)
         * [Resolve function](#resolve-function)
      * [Distributed Functions](#distributed-functions-1)
         * [distributed func declarations](#distributed-func-declarations)
         * [Distributed function parameters and return values](#distributed-function-parameters-and-return-values)
         * [Distributed functions are implicitly async throws when called cross-actor](#distributed-functions-are-implicitly-async-throws-when-called-cross-actor)
      * [Distributed Actor Isolation](#distributed-actor-isolation)
         * [No permissive special case for accessing constant let properties](#no-permissive-special-case-for-accessing-constant-let-properties)
         * [Distributed functions and protocol conformances](#distributed-functions-and-protocol-conformances)
         * [nonisolated members](#nonisolated-members)
      * [Actor Transports](#actor-transports)
         * [ActorTransport functions](#actortransport-functions)
         * [Transporting Errors](#transporting-errors)
      * [Actor Identity](#actor-identity)
         * [Distributed Actors are Identifiable](#distributed-actors-are-identifiable)
         * [Distributed Actors are Equatable and Hashable](#distributed-actors-are-equatable-and-hashable)
         * [Distributed Actors are Codable](#distributed-actors-are-codable)
      * ["Known to be local" distributed actors](#known-to-be-local-distributed-actors)
   * [Runtime implementation details](#runtime-implementation-details)
         * [Remote distributed actor instance allocation](#remote-distributed-actor-instance-allocation)
         * [distributed func internals](#distributed-func-internals)
   * [Future Directions](#future-directions)
         * [Resolving DistributedActor bound protocols](#resolving-distributedactor-bound-protocols)
         * [Synthesis of _remote and well-defined Envelope&lt;Message&gt; functions](#synthesis-of-_remote-and-well-defined-envelopemessage-functions)
         * [Support for AsyncSequence](#support-for-asyncsequence)
         * [Ability to hardcode actors to specific shared transport](#ability-to-hardcode-actors-to-specific-shared-transport)
         * [Actor Supervision](#actor-supervision)
      * [Related Work](#related-work)
         * [Swift Distributed Tracing integration](#swift-distributed-tracing-integration)
         * [Distributed Deadline propagation](#distributed-deadline-propagation)
         * [Potential Transport Candidates](#potential-transport-candidates)
   * [Related Proposals](#related-proposals)
   * [Alternatives Considered](#alternatives-considered)
      * [Discussion: Why Distributed Actors are better than "just" some RPC library?](#discussion-why-distributed-actors-are-better-than-just-some-rpc-library)
      * [Special Actor spawning APIs](#special-actor-spawning-apis)
         * [Explicit spawn(transport) keyword-based API](#explicit-spawntransport-keyword-based-api)
         * [Global eagerly initialized transport](#global-eagerly-initialized-transport)
         * [Directly adopt Akka-style Actors References ActorRef&lt;Message&gt;](#directly-adopt-akka-style-actors-references-actorrefmessage)
   * [Acknowledgments &amp; Prior Art](#acknowledgments--prior-art)
   * [Source compatibility](#source-compatibility)
   * [Effect on ABI stability](#effect-on-abi-stability)
   * [Effect on API resilience](#effect-on-api-resilience)

<!--te-->


## Introduction

Thanks to the recent introduction of [actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md) in Swift 5.5, developers gained the ability to express their concurrent programs in a new and natural way. 

Actors are a fantastic, foundational, building block for highly concurrent and scalable systems. Actors enable developers to focus on their problem domain, rather than having to micro manage every single function call with regard to its thread-safety. State isolated by actors can only be interacted with "through" the enclosing actor, which ensures proper synchronization. This also means that such isolated state, may not actually be locally available, and as far as the caller of such function is concerned, there is not much difference "where" the computation takes place.

This is one of the core strengths of the [actor model](https://en.wikipedia.org/wiki/Actor_model): it applies equally well to concurrent and distributed systems. 

This proposal introduces *distributed actors*, which allow developers to take full advantage of the general actor model of computation. Distributed actors allow developers to scale their actor systems beyond single node/device systems, without having to learn many new concepts, but rather, naturally extending what they already know about actors to the distributed setting.

Distributed actors introduce the necessary type system guardrails, as well as runtime hooks along with an extensible transport mechanism. This proposal focuses on the language integration pieces, and explains where a transport library author would interact with such a system to build a fully capable distributed actor runtime. The proposal does not go in depth about transport internals and design considerations, other than those required by the model.

**TODO: Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)**

After reading the proposal, we also recommend having a look at the [Related Proposals](#related-proposals), which may be useful to understand the big picture and how multiple Swift evolution proposals and libraries come together to support this proposal.



## Motivation

Most of the systems we write nowadays, whether we want it or not, are distributed.


For example:
- They might have multiple processes with the parent process using some IPC (inter-process communication) mechanism. 
- They might be part of a single (clustered) service, or a multi-service backend system that uses various networking technologies for communication between the nodes. 
- They might be composed of a client and server side applications, where the client needs to constantly interact with the server component to keep the application up to date. The same may apply to interactive, networked applications, such as games, chat or media applications.

These use cases all vary significantly and have very different underlying transports and mechanisms that enable them. Their implementations are also tremendously different. However, the general concept of wanting to communicate with non-local _identifiable_ entities is common in all of them. 

Distributed actors provide a general abstraction that extends the notion of an identifiable actor beyond the scope of a single process. By abstracting away the communication transport from the conceptual patterns associated with distributed actors, they enable application code to focus on their business, while providing a common and elegant pattern to solving the networking issues such applications would have solved in an ad-hoc manner otherwise.

This proposal _does not_ define any specific runtime. It is designed in such a way that various, first and third-party, transport implementations may be offered and co-exist even in the same process if necessary (e.g. utilizing some distributed actors for cross-process communication, while utilizing others to communicate with a server backend).

## Proposed solution

### Distributed actors

This proposal introduces the `distributed` contextual keyword, which may be used in conjunction with actor definitions (`distributed actor`), as well as `distributed func` declarations within such actors.

Distributed actors are very similar to their local counterparts. They provide the same state isolation guarantees as local actors with some additional restrictions. E.g. synchronous access to a remote property would not make sense. This means that everything that is true about an `actor` in general is also true for distributed actors.

When adding the `distributed` modifier to an existing `actor` the first thing we notice is that a distributed actor has stronger isolation requirements than plain local actors. Specifically, it is not possible to invoke plain (async or not) functions on the distributed actor anymore, and we'll need to mark functions that are accessible on the distributed actor as `distributed func`.

For example, we could implement a `Player` actor (similar to player objects as seen in the [SwiftShot](https://developer.apple.com/documentation/arkit/swiftshot_creating_a_game_for_augmented_reality) WWDC18 sample app), like this:

```swift
distributed actor Player {
  let name: String
  var score: Int
}
```

So far this behaves the same as a local-only actor, we cannot access `score` directly because the properties are "actor isolated". This is the same as with local-only actors where mutable state is isolated by the actor. Distributed actors however must also isolate immutable state, because the state itself may not even exist locally. Therefore, accessing `name` is also illegal on a distributed actor type, and would fail at compile time as shown below:

```swift
let player: Player
player.score // error: distributed actor-isolated property 'score' can only be referenced inside the distributed actor
player.name // error: distributed actor-isolated property 'name' can only be referenced inside the distributed actor
```

It is illegal to declare `nonisolated` _stored_ properties on distributed actors. The exact semantics of `nonisolated` will be discussed later on.

It is allowed to declare `static` properties and functions on distributed actors and–as they are completely outside of the distributed actor instance–they are legal to access from any context. They always refer to the local processes value of the static property or function. Usually these can be used for constants useful for the distributed actor itself, or users of it, e.g. like names or other identifiers.

```swift
distributed actor Player { 
  static let namePrefix = "PLAYER"
  static func makeName(name: String) -> String { 
    return "\(Player.namePrefix):\(name)"
  }
}

Player.namePrefix // ok
Player.makeName("Alice") // ok
```



### Distributed functions

A distributed function is declared by prefixing the `func` keyword with the `distributed` contextual keyword, like this:

```swift
distributed actor Worker { 
  func actuallyDoTheWork(task: TaskID) {
    // ... 
  }
  distributed func work() {
    actuallyToTheWork(.first)
    actuallyToTheWork(.second)
  }
}
```

Distributed functions are the only type of function or property that is "cross-actor" (see [cross-actor references and Sendable types](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md#cross-actor-references-and-sendable-types) in the actors proposal) callable on a distributed actor type.

A distributed function can only be declared in the context of a distributed actor. Attempts to define a distributed function outside of a distributed actor will result in a compile time error:

```swift
struct SomeNotActorStruct {
  distributed func nope() async throws -> Int { 42 } 
  // error: 'distributed' function can only be declared within 'distributed actor'
  // fixit: replace 'struct' with 'distributed actor'
  // fixit: remove 'distributed' from 'distributed func'
}
```

Distributed functions, when called cross-actor, are *implicitly* asynchronous *and* throwing. This follows precendent from the actor proposal, in which synchronous functions inside actors, when called cross-actor, are implicitly asynchronous. The distributed nature of distributed functions however, also implies that any such call may fail, and therefore must be assumed to be throwing:

```swift
distributed actor Worker { 
  distributed func work() -> Results { ... }
  
  func inside() {
    self.work() // calls inside the actor are always fine, and no implicit effects are added
  }
}

func outside(worker: Worker) {
  worker.inside() // error: cannot invoke non-distribured function 'inside()' on distributed actor 'Worker'
  worker.work() // error: 
}
```

It is not allowed to declare `static` `distributed` functions, because static functions are not isolated to the actor they are defined on, and as such can never be remote:

```swift
distributed actor Worker { 
  static distributed func illegal() {} 
  // error: 'distributed' functions cannot be 'static'
}
```

### Distributed actor transports

Distributed actors and functions are largely a type system and compiler feature, providing the necessary isolation and guardrails for such distributed actor to be correct and safe to use. In order for such an actor to actually perform any remote communication, it is necessary to provide an implementation of the `ActorTransport` protocol.

While distributed actors, on purpose, abstract away the "how" of their underlying messaging runtime, the actor transport is where the implementation must be defined. A transport must be provided to every distributed actor instantiation. By default the `init(transport:)` is synthesized (and the `init()` default initializer is _not_ synthesized for distributed actors).

```swift
distributed actor Worker {}

Worker() 
// error: missing argument for parameter 'transport' in call
// note: 'init(transport:)' declared here
//     internal init(transport: ActorTransport)
//              ^

let transport = SomeActorTransport()
let worker = Worker(transport: transport)
```

An actor transport can take advantage of any networking, cross process, or even in-memory approach it wants to, as long as it implements the protocol correctly. Transports may offer varying amounts of message send reliability, or other guarantees, and it is important that while we focus on the ability for actors to use any transport, it is not wrong for an application to be aware of what transports it will be used with, and e.g. attempt to limit message sizes to small messages in case the underlying transport has some inherent limitations around those.

End users of transports interact with them frequently, but not very deeply, as a transport must be passed to any instantiation of a distributed actor. Following that however, all other messaging interactions are done transparently by various hooks in the runtime. 

A typical distributed actor creation therefore can look like this:

```swift
let transport: ActorTransport = // WebSocketActorTransport(...)
                                // ClusterActorTransport(...)
                                // InterProcessActorTransport(...)

// create local instance, on given transport:
let local: Greeter = Greeter(transport: transport)
let greeting: String = try await local.greet(name: "Alice")
```

The second way of creating distributed actors is by resolving a potentially remote identity, this is done using the `resolve(id:using:)` function:

```swift
// resolve remote instance, using provided transport:
let maybeRemoteID: AnyActorIdentity = ...
let greeter: Greeter = try Greeter.resolve(id: maybeRemoteID, using: transport)
let greeting: String = try await greeter.greet(name: "Alice")
```

The returned greeter may be a remote or local instance, depending on what the passed in identity was pointing at. Resolving a local instance is how incoming messages are delivered to specific actors, the transport preforms a lookup using a freshly deserialized identity, and if an actor is located, delivers messages (function invocations) to it.

All distributed actors have the implicit nonisolated `actorTransport` and `id` property, which is initialized by the local initializer or resolve function automatically:

```swift
worker.actorTransport // ok; synthesized, nonisolated property
worker.id // ok; synthesized, nonisolated property
```

### Fundamental principle: Location Transparency

To fully understand and embrace the design of distributed actors it is useful to first discuss the concept of [location transparency](https://en.wikipedia.org/wiki/Location_transparency#:~:text=In%20computer%20networks%2C%20location%20transparency,their%20actual%20location.), as it is the foundational principle this design is built upon. This technique is usually explained as follows:

> In computer networks, location transparency is the use of *names* to identify network resources, rather than their *actual location*.

In our context, this means that distributed actors are uniquely identifiable in the network/system/cluster by their `ActorIdentity` which is assigned and managed by a specific `ActorTransport`. This identifier is sufficient to uniquely identify and locate the specific actor in the system, regardless of its location.

This, in combination with the principle that it is generally not possible to statically determine if a distributed actor is local or remote, allows us to fully embrace location transparency in the programming model. Developers should focus on getting their actor interactions correct, without focusing too much on _where exactly_ the actor is running. Static isolation checking rules in the model enforce this, and help developers not to violate this principle. 

> We offer _dynamic_ ways to check and peek into a local actor if it is _known to be local_, and we'll discuss this entry point in detail in ["Known to be local" distributed actors](#-known-to-be-local--distributed-actors), however their use should be rare and limited to special use-cases such as testing.

When an actor is declared using the `distributed` keyword (`distributed actor Greeter {}`), it is referred to as a "distributed actor". At runtime, references to distributed actors can be either "local" or "remote":

- **local** `distributed actor` references
  - which are semantically the same as non-distributed `actor`s at runtime.
- **remote** `distributed actor` references
  - which can be thought of as "proxy" objects, which merely point at a remote actor, identified by their `.id`. Such objects do not have any storage allocated for the actor declarations stored properties. Distributed functions on such instances are implemented by serializing and sending asynchronous messages over the underlying transport.

In other words, given the following snippet of code, we never know if the `greeter` passed to the `greet(who:)` function is local or remote. We merely operate on the information that it is a _distributed actor_, and therefore _may_ be remote:

```swift
distributed actor Greeter {
  distributed func hello() async throws
}

func greet(who greeter: Greeter) { // maybe remote, maybe local -- we don't care
  try await greeter.hello()
}
```

It is not _statically_ possible to determine if the actor is local or remote. This is hugely beneficial, as it allows us to write code independent of the location of the actors. We can write a complex distributed systems algorithm and test it locally. Deploying it to a cluster is merely a configuration and deployment change, without any additional code changes.

*Location Transparency* enables distributed actors to be used across various transports without changing code using them, be balanced between nodes once capacity of a cluster changes, be passivated when not in use and many more advanced patterns such as "virtual actor" style systems as popularized by Orleans and Akka's cluster sharding.

## Detailed design

### Distributed Actors

Distributed actors are declared using the `distributed actor` keywords, similar to local-only actors which are declared using only the `actor` keyword.

Similar to local-only actors which automatically conform to the `Actor` protocol, a type declared as `distributed actor` implicitly conforms to the `DistributedActor` protocol. The distributed modifier cannot be applied to any other type declaration other than `actor`, doing so results in a compile-time error:

```swift
distributed class ClassNope {} // error: 'distributed' can only be applied to 'actor' definitions
distributed struct StructNope {} // error: 'distributed' modifier cannot be applied to this declaration
distributed enum EnumNope {} // error: 'distributed' modifier cannot be applied to this declaration
```

A `distributed actor` and extensions on it are the only places where `distributed func` declarations are allowed. This is because in order to implement a distributed function, a transport and identity (actor address) are necessary.

It is not possible to declare free `distributed` functions since it would be unclear what this would mean.

It is possible for a distributed actor to have non-distributed functions as well. They are callable only from two contexts: the actor itself (by `self.nonDistributedFunction()`), and from within an `maybeRemoteActor.whenLocalActor { $0.nonDistributedFunction() }` which will be discussed in ["Known to be local" distributed actors](#known-to-be-local-distributed-actors), although the need for this should be relatively rare.

It is not allowed to define global actors which are distributed actors. If enough use-cases for this exist, we may loosen up this restriction, however generally this is not seen as a strong use-case, and it is possible to add this capability in a source and binary compatible way in the future if necessary.

#### The `DistributedActor` protocol

Similar to how any `actor` type automatically conforms to the `Actor` protocol, and any other kinds of types are banned from manually conforming to the `Actor` protocol, any `distributed actor` automatically conforms to the `DistributedActor` protocol.

The `DistributedActor` bears similarity to the `Actor` protocol and is defined as: 

```swift
public protocol DistributedActor: AnyActor, Sendable, Codable, Identifiable {

  // << Discussed in detail in "Resolve function" >>
  static func resolve<Identity>(id identity: Identity, using transport: ActorTransport) 
    throws -> Self
    where Identity: ActorIdentity

  // << Discussed in detail in "Actor Transports" >>
  var transport: ActorTransport { get }

  // << Discussed in detail in "Actor Identity" >> 
  var id: AnyActorIdentity { get }
}
```

It is not possible to declare any other type (struct, actor, class, ...) and make it conform to the `DistributedActor` protocol manually, for the same reasons as doing so is illegal for the `Actor` protocol: such a type would be missing additional type-checking restrictions and synthesized pieces which are necessary for distributed actors to function properly.

```swift
actor ActorNope: DistributedActor {
  // error: non-distributed actor type 'ActorNope' cannot conform to the 'DistributedActor' protocol
  // fixit: insert 'distributed' before 'actor'
}

class ClassNope: DistributedActor {
  // error: non-actor type 'ClassNope' cannot conform to the 'Actor' protocol
  // fixit: replace 'class' with 'distributed actor'
}
// (similarly for enums and structs)
```

The `DistributedActor` protocol includes a few more conformances which will be covered in depth in their own dedicated sections, as we discuss the importance of the [actor identity](#actor-identity) property.

The two property requirements (`transport` and `id`) are automatically implemented by the compiler. They are derived from specific calls to the underlying transport during the actor's initialization. They are immutable and will never change during the lifetime of the specific actor instance. Equality as well as messaging internals rely on this guarantee.


##### `DistributedActor` is not an `Actor`

It is crucial for correctness that neither `Actor` implies `DistributedActor`, nor the other way around. Such relationship would cause soundness issues in the model, e.g. like this:

```swift
// NOT PROPOSED; Illustrates a soundness issue if `DistributedActor: Actor` was proposed
//
// Assume that:
protocol DistributedActor: Actor { ... } // NOT proposed

extension Actor {
  func f() -> SomethingSendable { ... }
}
// and then...
func g<A: Actor>(a: A) async {
  print(await a.f())
}
// and then...
distributed actor MA {
}
func h(ma: MA) async {
  await g(ma) // BUG! allowed because a distributed actor is an Actor, but can't actually work at runtime
}
```

The core of the issue illustrated above stems from the incompatibility of the types semantics of "definitely local" vs. "maybe local, maybe remote". The semantics of a distributed actor to carry this "*maybe*" are crucial for building systems using distributed actors, and are not something we want to sacrifice (it is the foundation of location transparency), as such the two types are incompatible and converting between them seemingly with a sub-typing relationship would cause bugs and crashes at runtime.

The proposal addresses this simply by acknowlaging that DistributedActor does _not_ inherit from Actor, and we believe this is in practice a very much workable model.

##### Optional: The `AnyActor` marker protocol

We could introduce a lightweight marker `@_marker protocol AnyActor` which is inherited by both `Actor` and `DistributedActor`. 

```swift
@_marker protocol AnyActor {} 
protocol Actor: AnyActor {} 
protocol DistributedActor: AnyActor {}
```

The utility of this protocol is fairly limited in terms of extending it, because it has absolutely no protocol requirements. However, it is useful to express type requirements where we'd like to express the requirement that a type be implemented using any actor type, such as:

```swift
protocol Scientist { 
  func research() async throws -> Publication
}

func <S: AnyActor & Scientist>researchAny(scientist: S) async throws -> Publication {
  try await scientist.research()
}
```

> **Note:** Indeed, the utility of this marker protocol is rather limited... however we feel it would be nice, and relatively cost-free to introduce this type as a _marker_ protocol because of the zero runtime overhead of it, and the added logical binding of actors in a common type hierarchy. 
>
> The authors of the proposal could be convienced either way about this type though, and we welcome feedback from the community about it.

#### Progressive Disclosure towards Distributed Actors

The introduction of `distributed actor` is purely incremental, and does not imply any changes to the local programming model as defined by `actor`. However, once developers understand "share nothing" and "all communications are done through asynchronous functions (messages)", it is relatively simple to map the same understanding onto the distributed setting.

None of the distributed systems aspects of distributed actors leaks through to local-only actors. Developers who do not wish to use distributed actors may simply ignore them.

Developers who first encounter `distributed actor` in some API beyond their control can more easily learn about it if they have already seen actors in other parts of their programs, since the same mental model applies to distributed as well as local-only actors. The big difference is the inclusion of serialization and networking, but it can be quickly understood with the general "it will be slower than a local call" intuition. This is no different from having _some_ asynchronous functions performing "very heavy" work (like sending HTTP requests by calling `httpClient.post(<file upload>)`) while some other asynchronous functions are relatively fast--developers always need to reason about _what_ a function does in any case to understand performance characteristics.

Swift's Distributed Actors help because we can explicitly mark such network interaction heavy objects as `distributed actor`s, and therefore we know that distributed functions are going to use e.g. networking, so invoking them repeatedly in loops may not be the best idea. Xcode and other IDEs can make use of this static information to e.g. highlight distributed actor functions, helping developers understand where exactly networking costs are to be expected.

### Distributed Actor initialization

#### Local initializers

All `distributed actors` have a synthesized required designated initializer that accepts an `ActorTransport`. 

Every instantiation of a local distributed actor must call this initializer, which in turn performs lifecycle interactions with the transport, initializing the actor's `id` and `transport` properties.

> **Note:** We could potentially discuss if rather we would want to store `ActorIdentity` which would keep a reference to the transport it was created from. The general idea of an address and transport though remain crucial to the entire design, regardless if we decide to store them slightly differently or not.

This initializer contains synthesized lifecycle interactions with the passed in transport. A simplified outline of the synthesized initializer code is shown below:

```swift
distributed actor Greeter {
  /* ~~~ synthesized ~~~ */
  nonisolated var transport: ActorTransport { ... }
  nonisolated var id identity: ActorIdentity { ... }

  init(transport transport: ActorTransport) {
    self.transport = transport
    self.address = transport.assignAddress(Self.self)
  }
  /* === end of synthesized === */
}
```

Currently, overriding of this initializer itself is _not_ allowed. It is possible to define other initializers, however they must always delegate to this constructor, because of the important actor lifecycle tasks it performs:

```swift
distributed actor Bad {
  init() {} // forgot to call init(transport:)
  // error: 'distributed actor' initializer 'init(y:transport:)' must (directly or indirectly) delegate to 'init(transport:)'
  
  init(transport: ActorTransport) {}
  // Currently error: 'distributed actor' local-initializer 'init(transport:)' cannot be implemented explicitly
}
```

```swift
distributed actor OK1 {
  let name: String
  
  init(name: String) { // ok
    self.name = name
    self.init(transport: HardcodedTransport.global) // ok, but not recommended
  }
  
  init(name: String, transport: ActorTransport) {
    self.name = name
    self.init(transport: transport)
  }
}

distributed actor OK2 {
  
  init(y: Int, transport: ActorTransport) { // ok
    self.init(transport: transport)
  }

  init(x: Int, y: Int, transport: ActorTransport) {
    // ok, since we do delegate to init(transport) *eventually*
    self.init(y: y, transport: transport)
  }
}
```

> **Note:** We could potentially lift the restriction, and allow end-could be lifted in which case the necessary lifecycle hooks would be injected before and actor the user-defined initializer function body. The only reason we don't do this today is because it is hard to implement and make Definite Initialization happy about this. It is understood that it would be nice to allow implementing their own `init(transport:)`.

Thanks to the guarantee that the `init(transport:)` will _always_ be called when creating a distributed actor instance and the only other method of distributed actor initialization also ensures those properties are initialized properly, we are able to offer the `transport` and `id` properties as `nonisolated` members of any distributed actor. This is important, because those properties enable us to implement certain crucial protocol requirements using nonisolated functions, as we'll discuss in [Distributed functions and protocol conformances](#distributed-functions-and-protocol-conformances).

#### Resolve function

So far, we have not seen a way to create a "remote" distributed actor reference. This is handled by a special "resolve initializer" is synthesized for distributed actors. It is not manually implementable, and invokes internal runtime functionality for allocating "proxy" actors which is not possible to achieve in any other way.

A resolve initializer takes the shape of `init(resolve: ActorAddress, using: ActorTransport)`:

```swift
distributed actor Greeter {  
  /* ~~~ synthesized ~~~ */
  static func resolve(id identity: ActorIdentity, using transport: ActorTransport) throws -> Self {
    switch try await transport.resolve(identity, as: Self.self) {
    case .instance(let instance):
      return instance
    case .proxy:
      return <<make proxy to 'identity' over 'transport'>>
    }
  }
  /* === synthesized === */
}
```

A resolve may throw when the transport decides that it cannot resolve the passed in identity. A common example of a transport throwing in a `resolve` call would be if the `identity` is for some protocol `unknown://...` while the transport only can resolve `known://...` actor identities.

The resolve initializer and related resolve function on the `ActorTransport` are _not_ `async` because they must be able to be invoked from decoding values, and the `Codable` infrastructure is not async-ready just yet. Also, for most use-cases they need not be asynchronous as the resolve is usually implemented well enough using local knowledge. In the future we might want to introduce an asynchronous variant of resolving actors which would simplify implementing transports as actors themselves, as well as enable more complicated resolve processes.

A transport may decide to return a "dead letters" reference, which is a pattern in where instead of throwing, we return a reference that will log all incoming calls as so-called dead letters, meaning that we know that those messages will never arrive at their designated recipient. This concept is can be useful in debugging actor lifecycles, where we accidentally didn't keep the actor alive as long as we hoped etc. It is up to each specific transport to document and implement either behavior.

A resolve initializer may transparently create an instance if it decides it is the right thing to do. This is how concepts like "virtual actors" may be implemented: we never actively create an actor instance, but its creation and lifecycle is managed for us by some server-side component with which the transport communicates. Virtual actors and their specific semantics are outside of the scope of this proposal, but remain an important potential future direction of these APIs.

### Distributed Functions

#### `distributed func` declarations

Distributed functions are a type of function which can be only defined inside a distributed actor (or an extension on such actor), and any attempt of defining one outside of a `distributed actor` (or an extension of such) is a compile-time error:

```swift
distributed actor DA {
  distributed func greet() -> String { ... } // ok
}

extension DA {
  distributed func hola() -> String { ... } // ok
}

struct/class/enum/actor NotDistActor {
  distributed func nope() -> Int { ... } 
  // error: 'distributed' function can only be declared within 'distributed actor'
}
```

Distributed functions must be marked explicitly for a number of reasons, the first and most important one being that that they are subject to additional type-checking rules (discussed in detail in [Distributed Functions](#distributed-functions) and [Distributed Actor Isolation](#distributed-actor-isolation)). IDEs also benefit from the ability to understand that a specific function is distributed, and may want to color them differently or otherwise indicate that such function may have higher latency and should be used with care.

Similar to normal functions defined on actors, a distributed function has an implicitly `isolated` self parameter, meaning that they are able to refer to all internal state of an actor. Calling a distributed function on a local actor works effecitvely the same as calling any actor function on a local-only actor, if necessary an actor-hop will be emitted.

#### Distributed function parameters and return values

In addition to the usual `Sendable` parameter and return type requirements of actor functions, distributed actor functions also require their parameters and return values to conform to `Codable`. This is because any call of a distributed function may potentially need to be serialized, and `Codable` is our mechanism of doing so. 

Our greeter example naturally fulfills these requirements, because it only used primitive types which already conform to `Codable`, such as `String` and `Int`:

```swift
distributed actor Greeter { 
  distributed func greet(
    name: String, // ok: String is Codable
    age: Int      // ok: Int is Codable
  ) -> String {   // ok: String is Codable
    "Hello, \(name)! Seems you're \(age) years old."
  }
}
```

If we were to try to return a non-Codable type, such as a `Greeting` struct we just created, we'll get a compiler error informing us that the Greeting type must be made to conform to `Codable` as well:

```swift
struct Greeting { ... }

distributed actor Greeter {
  distributed func greet(name: String) -> Greeting { ... }
  // error: distributed function result type 'NotCodableValue' does not conform to 'Codable'
  // fixit: add 'Codable' conformance to 'Greeting'
  
  distributed func receive(_ greeting: Greeting) { ... }
  // error: distributed function parameter 'notCodable' of type 'NotCodableValue' does not conform to 'Codable'
  // fixit: add 'Codable' conformance to 'Greeting'
}
```

Once we make `Greeting` codable both those functions would compile and work fine.

The specific encoder/decoder choice is left up to the specific transport that the actor was created with. It is legal for a distributed actor transport to impose certain limitations, e.g. on message size or other arbitrary runtime checks when forming and sending remote messages. Distributed actor users may need to consult the documentation of such transport to understand those limitations. It is a non-goal to abstract away such limitations in the language model. There always will be real-world limitations that transports are limited by, and they are free to express and require them however they see fit (including throwing if e.g. the serialized message would be too large etc.)

#### Distributed functions are implicitly `async throws` when called cross-actor

Similarily to how any `actor` function is *implicitly* asynchronous if called from outside the actor (sometimes called a "cross-actor call"), distributed functions are implicitly asynchronous _and throwing_. This is because a distributed function represents a potentially remote call, and any remote call may fail due to various reasons not present in single-process programming. Connections can fail, transport layer timeouts and heartbeats may signal that the call should be considered failed etc. 

This implicit throwing behavior is in addition to whether or not the function itself is declared as throwing. The below snippet shows the implicit effects applied to distributed functions when called from the outside:

```swift
distributed actor Greeter {
  func englishGreeting() -> String { "Hello!" }
  func japaneseGreeting() throws -> String { "こんにちは！" }
  func germanGreeting() async -> String { "Servus!" }
  func polishGreeting() async throws -> String { "Cześć!" }
  
  func inside() async throws { 
    _ = self.englishGreeting() // ok
    _ = try self.japaneseGreeting() // ok
    _ = await self.germanGreeting() // ok
    _ = try await self.polishGreeting() // ok
  }
}

func outside(greeter: Greeter) async throws { 
  _ = try await greeter.englishGreeting()  // ok, implicit `async` and implicit `throws`
  _ = try await greeter.japaneseGreeting() // ok, implicit `async` and explicit `throws`
  _ = try await greeter.germanGreeting()   // ok, explicit `async` and implicit `throws`
  _ = try await greeter.polishGreeting()   // ok, explicit `async` and explicit `throws`
}
```

The same implicit rules apply for `nonisolated` distributed functions, because they are effectively functions outside of the actors isolation domain.

The type of errors thrown if the remote communication fail are up to the transport, however it is recommended that all errors thrown by the _transport_ rather than the actual remote function conform to the `ActorTransportError` protocol, which helps determine the reason of a call failing. We will discuss errors in more detail in their dedicated [Transporting Errors](#transporting-errors) section of this proposal.

> **Potential future direction:** If Swift were to embrace a more let-it-crash approach, which thanks to actors and the full isolation story of them seems achievable, we could consider building into the language a notion of crashing actors and "failures" which are separate from "errors". This is not being considered or pitched in this proposal, but it's worth keeping in mind. Further discussion on this can be found here: [Cleanup callback for fatal Swift errors](https://forums.swift.org/t/stdlib-cleanup-callback-for-fatal-swift-errors/26977)

### Distributed Actor Isolation

Distributed actor isolation inherits all the isolation properties of local-only actors, removes a few one local-only special-case and adds two additional restrictions to the model. In the following sections we will discuss them one-by one to fully grasp why they are necessary.

#### No permissive special case for accessing constant `let` properties

Distributed actors remove the special case that exists for local-only actors, where access is permitted to such actor's properties as long as they are immutable `let` properties.

Local-only actors in Swift make a special case to permit _synchronous_ access to _constant properties_, e.g. `let name: String`, since they are known to be safe to access and cannot be modified either so for the sake of concurrency safety, such access is permissible. Such loosening of the actor model is _not_ permissible for distributed actors, because these properties must are potentially remote, and any such access would have to be asynchronous and involve networking.

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

#### Distributed functions and protocol conformances

It is legal to declare a `DistributedActor` bound protocol, and to declare `distributed` functions inside it. In fact, this is a common use-case and will eventually allow developers to vend their APIs as plain protocols, without having to share the underlying `distributed actor` implementation. Such protocols may be defined as:

```swift
protocol DistributedFunctionality: DistributedActor {
  distributed func dist() -> String
  distributed func distAsync() async -> String
  distributed func distThrows() throws -> String
  distributed func distAsyncThrows() async throws -> String
}

distributed actor DF: DistributedFunctionality { 
  // ... implements all of the above functions ...
  
  func local() async throws {
    _ = self.dist() // ok
    _ = await self.distAsync() // ok
    _ = try self.distThrows() // ok
    _ = try await self.distAsyncThrows() // ok
  }
}

func outsideAny<AnyDF: DistributedFunctionality>(df: AnyDF) async throws {
  _ = try await self.dist() // ok
  _ = try await self.distAsync() // ok
  _ = try await self.distThrows() // ok
  _ = try await self.distAsyncThrows() // ok
}
```

As showcased in `outsideAny()` it is possible to invoke distributed functions on a generic or even existential type bound to a distributed actor protocol. The behavior of such invocations is as one might expect: if the actual passed actor is local, the actual function implementation is invoked, if it is remote a message is formed and sent to the remote distributed actor.

It is also legal to declare non-distributed functions and other protocol requirements in such protocol, however as usual with any distributed actor such will _not_ be possible to be invoked on the distributed actor itself. They would be possible to call cross-actor when using the "known to be local" escape hatch which will be discussed later on.

```  swift
protocol DistributedAndNonisolatedLocalFunctionality: DistributedActor {
  func makeName() -> String // must be implemented as `nonisolated`
  distributed func generate() -> String
}

// e.g. the generate function may offer a default implementation in terms of the local funcs
extension DistributedAndNonisolatedLocalFunctionality {
  distributed func generate() -> String { 
    "Default\(self.makeName())"
  }
}
```

A conforming distributed actor would be able to implement such protocol as follows:

```swift
distributed actor DAL: DistributedAndNonisolatedLocalFunctionality {
  nonisolated makeName() -> String { "SomeName" }
  
  // uses default generate() impl, 
  // or may provide one here by implementing `distributed func generate() -> String` here.
}
```

Given such a definition of the `DAL` actor and above protocols, the following would be legal/illegal invocations of the appropriate functions:

```swift
let dal: DAL
dal.makeName() // error: only 'distributed' functions can be called from outside the distributed actor
try await dal.generate() // ok!
```

The semantics of the `makeName()` witness are the same as discussed in [SE-0313: Improved control over actor isolation: Protocol conformances](https://github.com/apple/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md#protocol-conformances) - it must be nonisolated in order to be able to conform to the protocol requirement. As they are not distributed functions, the same rules apply as if it were a plain old normal actor. The only difference being, when one is allowed to invoke them cross-actor (only when using the `whenLocal(...)` escape hatch).

#### `nonisolated` members

In order to be able to implement a few yet very useful protocols for distributed actor usage within the context of e.g. collections, we need to be able to implement functions which are "independent" of their enclosing distributed actor's nature.

As we will discuss in detail in the following sections, distributed actors are for example `Equatable` by default, because there is exactly _one_ correct way of implementing this and a few other protocols on a distributed actor type.

The `nonisolated` serves the same purpose and mechanically works the same way as on normal actors - it effectively means that the implicit `self` parameter of any such function or computed property, is `nonisolated` which can be understood as the function being "outside" of the actor, and therefore no actor hop needs to be emitted to invoke such functions. This allows us to implement synchronous protocol requirements, such as Codable, Hashable and others. Refer to [SE-0313: Improved control over actor isolation](https://github.com/apple/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md#protocol-conformances) for more details on actors and protocol conformances.

Specifically, the Equatable and Hashable implementations have only _one_ meaningfully correct implementation given any distributed actor, which is utilizing the actors address to check for equality or compute the hash code:

```swift
protocol DistributedActor {
  nonisolated var actorAddress: ActorAddress { get }
  // ...
}

distributed actor Worker {} 

extension Worker: Equatable {
  @inlinable
  public static func ==(lhs: Self, rhs: Self) -> Bool {
    lhs.actorAddress == rhs.actorAddress
  }
}

extension Worker: Hashable { 
  nonisolated public func hash(into hasher: inout Hasher) {
    self.actorAddress.hash(into: &hasher)
  }
}
```

While it is not possible to declare stored properties of distributed actors as `nonisolated`, the address and transport are implemented using special computed properties, which know where in memory those fields are stored for every distributed actor, even if it is a remote instance which does not otherwise allocate memory for any of its declared stored properties.

Unlike local-only actors, distributed actors *cannot* declare stored properties as `nonisolated`, however nonisolated functions, computed properties and subscripts are all supported. As any nonisolated function though, they may not refer to any isolated state of the actor.

This would invite a model in the isolation checking, where we would allow–when dealing with a remote actor reference–access to properties which do not exist in memory. This is because a remote reference has allocated _exactly_ as much memory for the object as is necessary to store the address and transport fields, and no storage is allocated for any of the other stored properties of a remote reference, because the actor's state _does not exist_ locally, and we would not be able to invent any valid values for it. Therefore, it must not be possible to declare stored properties as nonisolated.

This also means that it is possible to access the address and transport cross-actor, even though they are not distributed, and these accessess will not have any implicit effects applied to them:

```swift
distributed actor Worker {}
let worker: Worker = ... 
worker.id   // ok
worker.transport // ok
```

### Actor Transports

Distributed actors are always associated with a specific transport that handles a specific instance.

A transport a protocol that distributed runtime frameworks implement in order to take intercept and implement the messaging performed by a distributed actor.

The protocol is defined as:


```swift

public protocol ActorTransport: Sendable {

  // ==== ---------------------------------------------------------------------
  // - MARK: Resolving actors by identity

  func decodeIdentity(from decoder: Decoder) throws -> AnyActorIdentity

  func resolve<Act>(_ identity: Act.ID, as actorType: Act.Type) throws -> ActorResolved<Act>
      where Act: DistributedActor

  // ==== ---------------------------------------------------------------------
  // - MARK: Actor Lifecycle

  func assignIdentity<Act>(_ actorType: Act.Type) -> Act.ID
      where Act: DistributedActor

  func actorReady<Act>(_ actor: Act) 
      where Act: DistributedActor

  func resignIdentity(_ id: AnyActorIdentity)

}
```

A transport has two main responsibilities:

- creating and resolving actor addresses which are used by the language runtime to construct distributed actor instances,
- perform all message dispatch and handling on behalf of a distributed actor it manages, specifically:
  - for a remote distributed actor reference: 
    - be invoked by the framework's source generated `$distributed_function` implementations with a "Message" representation of the locally invoked function, serialize and dispatch it onto the network or other underlying transport mechanism. 
    - This turns local actor function invocations into messages put on the network.
  - for a local distributed actor instance: 
    - handle all incoming messages on the transport, decode and dispatch them to the apropriate local recipient instance. 
    - This turns incoming network messages into local actor invocations.

The first category can be seen as "lifecycle management" of distributed actors, and is interlinked with the Swift runtime and synthesized code of a distributed actor. The compiler will synthesize code to call out to the transport to implement some of its most crucial functionality:

- When a distributed actor is created using the local initializer (`init(transport:)`) the `transport.assignAddress(...)` function is invoked.
- When the actor is deinitialized, the transport is invoked with `transport.resignAddress(...)` with the terminated actor's address. 
- When creating a distributed actor using the resolve initializer (`resolve(id:using:)`) the Swift runtime invokes `transport.resolve(...)` asking the transport to decide if this address resolves as a local reference, or if a proxy actor should be allocated. Creating a proxy object is a Swift internal feature, and not possible to invoke in any way other than using a resolve initializer.

The second category is "actually sending/receiving the messages" which is highly dependent on the details of the underlying transport. We do not have to impose any API requirements on this piece of a transport actually. Since a distributed actor is intended to be started with a transport, and `$distributed_` functions are source generated by the same framework as the used transport, it can downcast the property to `MyTransport` and implement the message sending whichever way it wants. 

This way of dealing with message sending allows frameworks to use their specific data-types, without having to copy back and forth between Swift standard types and whichever types they are using. It would be helpful if we had a shared "bytes" type in the language here, however in general a transport may not even directly operate on bytes, but rather accept a `Codable` representation of the invoked function (e.g. an enum that is `Codable`) and then internally, depending on configuration, pick the appropriate encoder/decoder to use for the specific message (e.g. encoding it using a binary coder rather than JSON etc). By keeping this representation fully opaque to Swift's actor runtime, we also allow plugging in completely different transports, and we could actually invoke gRPC or other endpoints which use completely different serialization formats (e.g. protobuf) rather than the `Codable` mechanism. We don't want to prevent such use-cases from existing, thus opt to keep the "send" functions out of the `ActorTransport` protocol requirements. This is also good, because it won't allow users to "randomly" write `self.transpot.send(randomMessage, to: id)` which would circumvent the type-safety experience of using distributed actors.

#### ActorTransport functions

The transport's functions are closely tied to the lifecycle of a distributed actor. Specific transport calls are synthesized into every distributed actor declaration, e.g. interacting with it upon initialization and deinitialization.

An actor transport can be implemented by conforming to the `ActorTransport` protocol:

 ```swift
 
 public protocol ActorTransport: Sendable {
 
   // ==== ---------------------------------------------------------------------
   // - MARK: Resolving actors by identity
   /// Upon decoding of a `AnyActorIdentity` this function will be called
   /// on the transport that is set un the Decoder's `userInfo` at decoding time.
   /// This user info is to be set by the transport, as it receives messages and
   /// attempts decoding them. This way, messages received on a specific transport
   /// (which may be representing a connection, or general transport mechanism)
   /// is able to utilize local knowledge and type information about what kind of
   /// identity to attempt to decode.
   func decodeIdentity(from decoder: Decoder) throws -> AnyActorIdentity
 
   /// Resolve a local or remote actor address to a real actor instance, or throw if unable to.
   /// The returned value is either a local actor or proxy to a remote actor.
   ///
   /// Resolving an actor is called when a specific distributed actors `init(from:)`
   /// decoding initializer is invoked. Once the actor's identity is deserialized
   /// using the `decodeIdentity(from:)` call, it is fed into this function, which
   /// is responsible for resolving the identity to a remote or local actor reference.
   ///
   /// If the resolve fails, meaning that it cannot locate a local actor managed for
   /// this identity, managed by this transport, nor can a remote actor reference
   /// be created for this identity on this transport, then this function must throw.
   ///
   /// If this function returns correctly, the returned actor reference is immediately
   /// usable. It may not necessarily imply the strict *existence* of a remote actor
   /// the identity was pointing towards, e.g. when a remote system allocates actors
   /// lazily as they are first time messaged to, however this should not be a concern
   /// of the sending side.
   ///
   /// Detecting liveness of such remote actors shall be offered / by transport libraries
   /// by other means, such as "watching an actor for termination" or similar.
   func resolve<Act>(_ identity: Act.ID, as actorType: Act.Type) throws -> ActorResolved<Act>
       where Act: DistributedActor
 
   // ==== ---------------------------------------------------------------------
   // - MARK: Actor Lifecycle
   /// Create an `ActorIdentity` for the passed actor type.
   ///
   /// This function is invoked by an distributed actor during its initialization,
   /// and the returned address value is stored along with it for the time of its
   /// lifetime.
   ///
   /// The address MUST uniquely identify the actor, and allow resolving it.
   /// E.g. if an actor is created under address `addr1` then immediately invoking
   /// `transport.resolve(address: addr1, as: Greeter.self)` MUST return a reference
   /// to the same actor.
   // FIXME: make it Act.ID needs changes in AST gen
   func assignIdentity<Act>(_ actorType: Act.Type) -> AnyActorIdentity
       where Act: DistributedActor
 //  func assignIdentity<Act>(_ actorType: Act.Type) -> Act.ID
 //      where Act: DistributedActor
   func actorReady<Act>(_ actor: Act) where Act: DistributedActor
 
   /// Called during actor deinit/destroy.
   func resignIdentity(_ id: AnyActorIdentity)
 
 }
 
 public enum ActorResolved<Act: DistributedActor> {
   case resolved(Act)
   case makeProxy
 }
 ```



#### Transporting `Error`s

A transport _may_ attempt to transport errors back to the caller if it is able to encode/decode them, however this is _not encouraged_.

Instead, logic errors should be modelled by returning `Result<User, InvalidPassword>` as it allows for typed handling of such errors as well as automatically enforcing that the returned error type is also `Codable` and thus possible to encode and transport back to the caller. This is a good idea also because it forces developers to consider if an error really should be encoded or not (perhaps it contains large amounts of data, and a different representation of the error would be better suited for the distributed function).

Generally, it is encouraged to separate the "any failure" handling related to transport failures (such as timeouts or network errors), which are represented by the untyped `throws` of a distributed function call, from logical errors which (e.g. "invalid password").

The exact errors thrown by a distributed function depends on the underlying transport. Generally one should expect some form of `TheTransportError` to be thrown by a distributed function if transport errors occur--the exact semantics of those throws are intentionally left up to specific transports to document when and how to expect errors to be thrown.

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

### Actor Identity

A distributed actor's identity is defined by its `ActorIdentity`. The property `var id: AnyActorIdentity { get }` is implemented for every distributed actor 

The actor address is a protocol which an `ActorTransport` framework implements and must return when a distributed actor is initialized (see [Initializing local distributed actors](#initializing-local-distributed-actors)).

The address can be seen as a form of URI, however we leave it up to transports to specify the exact formats they want to use. We require an address to start with a `transport://` identifier, such that the `address.protocol` can be used to determine which transport should be used for resolving an address if otherwise unknown.

```swift
// TODO: move to protocol (!), just "ID" or ActorTransport.ActorIdentity
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

An actor's address is automatically assigned to it at creation time (by the compiler invoking the `ActorTransport`'s `assignAddress` in the synthesized initializers discussed above) and stored in a synthesized `id` property:

```swift
distributed actor Greeter { 
  /* ~~~ synthesized ~~~ */
  nonisolated var actorAddress: ActorAddress { ... } // returns value created during init or resolve
  /* === synthesized === */
}
```

Identities are created by a specific `ActorTransport` which will usually implement them using a `struct` carrying any of the fields that are necessary for it to serialize and resolve it later. For example, a "Net Actors" framework would implement the protocol like this:

```swift
/// Example of specific actor identity, e.g. encoding it as host+port and unique ID of an actor.
public struct NetActorAddress: ActorIdentity { 
  let proto: String // protocol of this cluster node
  let host: String // host of this cluster node
  let port: UInt32 // port of this cluster node
  let uid: UInt64 // unique identifier of this actor
}
```

Again, the exact shape and fields of an actor address are completely up to the `ActorTransport` framework. Some may just need a single `uid`, others might need more host coordinates, such as shown above.

#### Distributed Actors are `Identifiable`

Another natural candidate for default conformance is the [`Identifiable` protocol](https://developer.apple.com/documentation/swift/identifiable) which is used to provide a _stable (logical) identity_, as this is exactly what a distributed actor's identity provides, we propose to conform to this protocol by default.

The `Identifiable` protocol is useful for any type which has a *stable identity*, and in the case of a distributed actor there is a single correct way to conform to this protocol: by implementing the `id` requirement using the actor's address, therefore this conformance is provided as part of the standard library:

```swift
extension DistributedActor: Identifiable { 
  nonisolated var id: AnyActorIdentity { /*... synthesized ...*/ }
}
```

The identity is a value generated by the transport upon the actor's instantiation, and is stored automatically by any distributed actor. It is nonisolated, which means that it can be accessed synchronously, and even though that it is not a distributed function, it can always be accessed on any distributed actor. 

The identity can be largely viewed as an opaque object from the perspective of the runtime, and users of distributed actors. However, for users it may provide useful debug information since, depending on the transport, it may include a full network address of the referred to actor, or some other logical name that can help identify the specific instance this reference is pointing at in the case of a remote actor reference.

Because the identity is `nonisolated` we can use it to implement all kinds of other synchronous protocol requirements, as we will discuss next.

#### Distributed Actors are `Equatable` and `Hashable`

Distributed actors automatically get synthesized `Equatable` and `Hashable` conformances.

Equality of actors _by actor identity_ is tremendously important, because it enables us to "remember" actors in collections, look them up, and compare if an incoming actor reference is one which we have not seen yet, or is a previously known one. It is an important piece of the location transparency focused design that we are proposing. For example:

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

While this example is pretty simple, the ability to identify actors using their address is tremendously important for distributed actors. 

Unlike local-only actors, reference equality (`===`) is very likely incorrect! This is because we need to be able to implement transports which simply return a new proxy instance whenever they are asked to resolve a remote instance, rather than forevermore return the same exact instance for them which would lead to infinite (!) memory growth including forever keeping around instances of remote actors which we never know for sure if they are alive or not, or if they ever will be resolved again. Therefore `ActorAddress` quality is the only sane way to implement equality _and_ is a crucial element to keep transport implementations memory efficient.

This is how the `==` and `hash` functions are synthesized by the compiler:

```swift
extension Chatter: Hashable {  
  public static func == (lhs: Self, rhs: Self) -> Bool {
    lhs.id == rhs.id
  }

  nonisolated public func hash(into hasher: inout Hasher) {
    self.id.hash(into: &hasher)
  }
}
```

If a distributed actor type provides either an `==(lhs:rhs:)` or `hash(into:)` implementation, the synthesis will be disabled.

We, purposefully, do not implement these protocols directly on the `DistributedActor` protocol, because we want to allow such protocols be used as existentials in the near future. If we conformed the protocol itself to `Hashable` we would not be able to store other distributed actor protocols as existentials in variables, which is an essential use-case we want to support in order to allow for resolving such protocols, without knowing their implementation type at runtime. This use-case is discussed in depth in [Future Directions: Resolving `DistributedActor` bound protocols](#resolving-distributedactor-bound-protocols)).

#### Distributed Actors are `Codable`

Distributed actors are `Codable` and are represented as their _actor identity_ in their encoded form.

In order to be true to the actor model's idea of freely shareable references, regardless of their location (known as the property of [*location transparency*](#location-transparency)), we need to be able to pass distributed actor references to other--potentially remote--distributed actors.

This implies that distributed actors must be `Codable`. However, the way that encoding and decoding is to be implemented differs tremendously for actors and non-actors. Specifically, it does not make sense to serialize the actor's state - it is after all what is isolated from the outside world and from external access.

The `DistributedActor` protocol also conforms to `Codable`. As it does not make sense to encode/decode "the actor", per se, the actor's encoding is specialized to what it actually intends to express: encoding an address, that can be resolved on a remote node, such that the remote node can contact this actor. This is exactly what the `ActorAddress` is used for, and thankfully it is an immutable private property of each actor, so the synthesis of Codable of a distributed actor boils down to encoding its' address:

```swift
distributed actor DA {
  nonisolated var id identity: ActorIdentity { ... }
  nonisolated var x: Int {}
  nonisolated let xx: Int // ban
  
  let yy: Int // ok
  
  nonisolated func bar() {
    self.x
    self.xx // BAD
    
    self.yy // bad, because not distributed
    // ask "is the BASE of this member access known to be isolated"
  }
  da.bar()
}

let da: DA 
da.address // ok


// no need for "assume local", just hop to the DA and then it is isolated 
await da.whenLocal { (localDA: isolated DA) in 
  localDA.xx // OK?
}

extension DistributedActor: Codable {
  /* ~~~ synthesized ~~~ */ 
  nonisoalted 
  @derived func encode(to encoder: Encoder) throws { 
    var container = encoder.singleValueContainer()
    try container.encode(self.actorAddress)
  }
  /* === synthesized === */
}
```

Decoding is slightly more involved, because it must be triggered from _within_ an existing transport. This makes sense, since it is the transport's internal logic which will receive the network bytes, form a message and then turn to decode it into a real message representation before it can deliver it to the recipient. 

In order for decoding of an Distributed Actor to work on every layer of such decoding process, a special `CodingUserInfoKey.transport` is used to store the actor transport in the Decoder's `userInfo` property, such that it may be accessed by any decoding step in a deep hierarchy of encoded values. If a distributed actor reference was sent as part of a message, this means that it's `init(from:)` will be invoked with the actor transport present. 

The default synthesized decoding conformance can therefore automatically, without any additional user intervention, decode itself when being decoded from the context of any actor transport. The synthesized initializer looks roughly like this:

```swift
extension DistributedActor {
  /* ~~~ synthesized ~~~ */ 
  @derived init(from decoder: Decoder) throws {
    guard let transport = self.userInfo[.transport] as? ActorTransport else {
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

The above snippet is showcases that with distributed actors, and codable DistributedActor types it is trivial to implement a simple pub-sub style publisher, and of course this ease of development extends to other distributed systems issues. The fundamental building blocks being a natural fit for distribution that "just click" are of tremendous value to the entire programming style with distributed actors. Of course a real implementation would be more sophisticated in its implementation, but it is a joy to look at how distributed actors make mundane and otherwise difficult distributed programming tasks simple and understandable.

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

This situation sometimes ocurs when developing a form of manager actor which always has a single local instance per cluster node, but also is reachable by other nodes as a distributed actor. Since the actor is defined as distributed, we can only send messages which can be encoded to it, however sometimes such APIs have a few specialized local functions which make sense locally, but are never actually sent remotely. We do want to have these local-only messages to be handled by exactly the same actor as the remote messages, to avoid race conditions and accidental complexity from splitting it up into multiple actors.

A specific example of such pattern is the [CASPaxos protocol](https://arxiv.org/abs/1802.07000), which is a popular distributed consensus protocol which performs a distributed compare-and-set operation over a distributed register. It's API accepts a `change` function, which naturally we cannot (and do not want to) serialize and send around the cluster, however the local proposer wants to accept such function:

```swift
distributed actor Proposer<Value: Codable> { 
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
  @discardableResult
  nonisolated func whenLocal(
    _ body: (actor Self) async throws -> T
  ) (re)async rethrows -> T?
  
  nonisolated func whenLocal(
    _ body: (actor Self) async throws -> T,
    else whenRemote (Self) async throws -> T
  ) (re)async rethrows -> T
}
```

Which can be used like this:

```swift
distributed actor Greeter { func tell(_ message: String) { print(message) } }
let greeter: Greeter = Greeter(transport: someTransport)
let greeter2: Greeter = try Greeter(resolve: address, transport: someTransport)

greeter.whenLocal { greeterWasLocal in 
  greeterWasLocal.tell("It is local, after all!")
}

let location = greeter.whenLocal { _ in
  "was local"
} else { 
  "was remote, after all!"
}

assert(greeter == greeter2)
assert(greeter === greeter2)
```

This allows us to keep using the actor as isolation and "linearization" island, keep the distributed protocol implementations simple and not suffer from accidental complexity.

This API should be seen as somewhat of a "backdoor", and APIs should not abuse it without need, however we acknowlage these situations happen in the real world, and would like to offer a clean solution. 

> For comparison, when this situation happens with other runtimes such as Akka the way around it is to throw exceptions when "not intended for remoting" messages are sent. It is possible to determine if an actor is local or remote in such runtimes, and it is used in some low-level orchestration and internal load balancing implementations, e.g. selecting 1/2 of local actors, and moving balancing them off to another node in the cluster etc.

The implementation of the `isLocalActor` function is a trivial check if the `isDistributedActor` flag is set on the actor instance, and therefore does not add any additional storage to the existing actor infrastructure since actors already have such flags property used for other purposes.

## Runtime implementation details

This section of the proposal discusses some of the runtime internal details of how distributed functions and remote references are implemented. While not part of the semantic model of the proposal, it is crucial for the implementation approach to fit well into Swift's runtime.

#### Remote `distributed actor` instance allocation

Creating a proxy for an actor type is done using a special `resolve(id:using:)` factory function of any distributed actor. Internally, it invokes the transport's `resolve` function, which determines if the identity resolves to a known local actor managed by this transport, a remote actor which this transport is able to communicate with, or the identity cannot be resolved and the resolve will throw:

```swift
protocol DistributedActor { 
    static func resolve<ID>(identity: ActorIdentity, using transport: ActorTransport) 
      throws -> Self 
      where ID: ActorIdentity { 
    	// ... synthesized ...
    }
}

protocol ActorTransport { 
  /// Resolve a local or remote actor address to a real actor instance, or throw if unable to.
  /// The returned value is either a local actor or proxy to a remote actor.
  func resolve<Act>(id identity: ActorIdentity, as actorType: Act.Type) 
      throws -> ResolvedDistributedActor<Act>
	  where Act: DistributedActor
}

enum ResolvedDistributedActor<Act: DistributedActor> { 
  case resolved(instance: Act)
  case makeProxy
}
```

This function can only be invoked on specific actor types–as usual with static functions on protocols–and serves as a factory function for actor proxies of given specific type.

Implementing the resolve function by returning `.resolved(instance)` allows the transport to return known local actors it is aware of. Otherwise, if it intends to proxy messages to this actor through itself it should return `.proxy`, instructing the constructor to only construct a partial "proxy" instance using the address and transport. The transport may also chose to throw, in which case the constructor will rethrow the error, e.g. explaining that the passed in address is illegal or malformed.

The `resolve` function is intentionally not asynchronous, in order to invoke it from inside `decode` implementations, as they may need to decode actor addresses into actor references.

To see this in action, consider:

```swift
distributed actor Greeter { ... }
```

Given an `ActorAddress`, `Greeter.resolve` can be used to create a proxy to the remote actor:

```swift
let greeter = try Greeter.resolve(id: someIdentity, using: someTransport)
```

The specifics of how a `resolve` works are left up to the transport, as their semantics depend on the capabilities of the underlying protocols the transport uses.

Implementations of resolve should generally not perform heavy operations and should be viewed similar to initializers -- quickly return the object, without causing side effects or other unexpected behavior.

#### `distributed func` internals

> **Note:** It is a future direction to stop relying on end-users or source-generators having to fill in the `_remote_` function implementations. However we will only do so once we've gained practical experience with a few more transport implementations. This would happen before the feature is released from its experimental mode however.

Developers implement distributed functions the same way as they would any other functions. This is a core gain from this model, as compared to external source generation schemes which force users to implement and provide so-called "stubs". Using the distributed actor model, the types we program with are the same types that can be used as proxies--there is no need for additional intermediate types.

A local `distributed actor` instance is a plain-old `actor` instance. If we simplify the general idea what an actor call actually is, it boils down to enqueueing a representation of the call to the actor's executor. The executor then eventually schedules the task/actor and the call is invoked. We could think about it as if it did the following for each cross-actors call:

```swift
// local-actor PSEUDO-CODE (does not directly represent actual runtime implementation!)
func greet(name: String) async -> String { 
  let job = __makeTaskRepresentingThisInvocation()
  self.serialExecutor.enqueue(job)
  return await job.get() // get the job's result
}
```

Remote actors are very similar to this, but instead of enqueueing a `job` into their `serialExecutor`, they convert the call into a `message` (rather than `job`) and pass it to their `transport` (rather than `Executor`) for processing. In that sense, local and remote actors are very similar, each needs to turn an invocation into something the underlying runtime can handle, and then pass it to the appropriate runtime.

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

The function at compile time will generate where the `$distributed_[name]([params])` function is assumed to be provided by _someone_. In reality these functions will be provided by code generators which are able to turn the function calls (name + parameters) into `Codable` messages and dispatch such message onto the `transport` via `transport.send(id:message:expectingReply:)`.

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



## Future Directions

#### Resolving DistributedActor bound protocols

In some situations it may be undesirable or impossible to share the implementation of a distributed actor (the `distributed actor` definition) between "server" and "client". 

We can imagine a situation where we want to offer users of our system easy access to it using distributed actors, however we do not want to share our internal implementation thereof. This works similarly to how one might want to publish API definitions, but not the actual API implementations. Other RPC runtimes solve this by externalizing the protocol definition into external interface description languages (IDLs), such as `.proto` files in the case of gRPC.

With Swift, we already have a great way to define protocols... protocols!

Distributed actor protocols, i.e. protocols which also conform to `DistributedActor`, are allowed to define distributed functions and can only be implemented by declaring a `distributed actor` conforming to such protocol.

For example, it is legal to define the following distributed actor protocol:

```swift
protocol Greeter: DistributedActor {
  distributed func greet(name: String) throws -> String
}
```

And a "client" side application, even without knowledge of how the distributed actor is implemented on the "backend" may resolve it as follows:

```swift
let id: ActorIdentity = ... // known to point at a remote `Greeter`
let greeter: Greeter = try Greeter.resolve(id: id, using: someTransport)

let greeting = try await greeter.greet("Alice")
assert(greeting == "Hello, Alice!")
```

Such a resolved reference (i.e., `greeter`) should be a remote actor, since there is no local implementation the transport can invent to implement this protocol. We could imagine some transports using source generation and other tricks to fulfil this requirement, so this isn't stated as a MUST, however in any normal usage scenario the returned reference would be remote or the resolve should throw.

In other words, thanks to Swift's expressive protocols and isolation-checking rules applied to distributed functions and actors, we are able to use protocols as the interface description necessary to share functionality with other parties, even without sharing out implementations. There is no need to step out of the Swift language to define and share distributed system APIs with eachother.

> TODO: This is not implemented yet, and a bit more tricky however unlocks amazing use cases for when client/server are not the same team or organization.

#### Synthesis of `_remote` and well-defined `Envelope<Message>` functions

Currently the proposal relies on "someone", be it a SwiftPM plugin performing source code generation, or a developer implementing specific `_remote_` function counterparts for each distributed function for a transport to receive apropriate message representations of each function invocation. 

While this is sub-optimal, it is not a road-block for the first iteration of this proposal. It allows us to explore and iterate on specific requirements from various transport implementations without having to bake their assumptions into the compiler right away.

Once we have collected enough experience from real transport implementations we will be able to remove the requirement to "fill in" remote function implementations by end users and instead synthesize them. This change will likely take the shape of introducing a common "envelope" type, and adding a `send(envelope: Envelope)` requirement to the `ActorTransport` protocol, as such, we expect this piece of work would be best done before stabilizing the distributed feature.

A rough sketch of what would need to be synthesized by the compiler is shown below:

```swift
distributed actor Greeter { 
  distributed func greet(name: String) -> String
}

// *mockup - not complete design yet*
protocol ActorTransport {
  func send<Message: Codable>(envelope: Envelope) async throws 
}

// ------- synthesized --------
extension Greeter { 
  func $remote_greet(name: String) async throws -> String { 
    var envelope: Envelope<$GreetMessage> = Envelope($GreetMessage(name: name))
    envelope.recipient = self.id
    // ... 
    return try await self.transport.send(envelope)
  }
}
// ---- end of synthesized ----
```

The difficulty in this is mostly in the fact that we would be committing forever to the envelope format and how the messages are synthesized and encoded. 

This must be thought though with consideration for numerous transports, and only once we're confident the strategy serves all transports we're interested will we be able to commit to a synthesis strategy here. Until then, we want to explore and learn about the various transport specific complications as we implement them using source generators and/or hand implemented `_remote_` functions.

#### Support for `AsyncSequence`

This isn't really something that the language will need much more support for, as it is mostly handled in the `ActorTransport` and serialization layers, however it is worth pointing out here.

It is possible to implement (and we have done so in [other runtimes](https://doc.akka.io/docs/akka/current/stream/stream-refs.html) in the past), distributed references to streams, which may be consumed across the network, including the support for flow-control/back-pressure and cancellation.

This would manifest in returning / accepting values conforming to the AsyncSequence or some more specific marker protocol. Distributed actors can then be used as coordinators and "sources" e.g. of metrics or log-lines across multiple nodes -- a pattern we have seen successfully applied in other runtimes in the past.

#### Ability to hardcode actors to specific shared transport

In this potential extension we would allow either requiring a specific type of transport be used by adopting distributed actors, or outright provide a shared transport instance for certain distributed actors.

Specifically, it may be useful for some transports which offer special features that only they can implement (and perhaps a test "in memory" transport), to require that all distributed actors conforming to `FancyDistributedActor` should require `FancyActorTransport`:

```swift
protocol FancyDistributedActor: DistributedActor { 
  typealias ActorTransportRequirement = FancyActorTransport
}
```

This would affect the generated initializer and related functions, by changing the type used for the transport transport parameter and storage:

```swift
distributed actor FancyGreeter: FancyDistributedActor { 
  // var transport: FancyActorTransport { ... }
  // init(transport: FancyActorTransport) { ... }
}
```

We can also imagine specific transports, or projects, which know that they only use a specific shared transport in the entire application, and may avoid this initialization boilerplate. This would be possible if we tweak synthesis to allow and respect properties initialized in their field declarations, like this:

```swift
protocol SpecificDistributedActor: DistributedActor { 
  var transport: ActorTransport { SpecificTransport.shared }
}

distributed actor SpecificDaemon: SpecificActor { 
  // var transport: SpecificTransport { ... }
  
  // NOT synthesized: init(transport:)
 
  /* ~~~ synthesized instead ~~~ */
  init()
  static func resolve(id:) throws
  /* === synthesized instead === */
}
```

#### Actor Supervision

Actor supervision is a powerful and crucial technique for distributed actor systems. It is a pattern well-known from other runtimes such as Erlang and Akka, where it is referred to "linking" or "watching" respectively.

Our goal with the `distributed actor` language feature is to allow enough flexibility such that such features may be implemented in transport libraries. Specific semantics on failure notification may differ depending on transports, and while a generalization over them can be definitely very useful (and we may end up providing one), allowing specific transports to offer specific failure handling mechanisms is called for as well.

A rough sketch of this is shown below. An actor can "watch" other actors if the transport supports it, and if such remote actor–or the node on which it was hosted–crashes, we would be notified in the `actorTerminated` function and may react to it, e.g. by removing any such crashed workers from our internal collection of workers:

```swift
@available(SwiftStdlib 5.5, *)
distributed actor Observer {
  let watch: DeathWatch!
  var workers: [Worker] = []

  init(…) { 
    // … 
    watch = DeathWatch(self)
  }

  func add(worker: Worker) {
    workers[worker.id] = watch(worker)
  }

  func actorTerminated(identity: AnyActorIdentity) {
    print(“oh no, \(identity) has terminated!”)
    workers.remove(identity)
  } 
}
```





### Related Work

#### Swift Distributed Tracing integration

> This future direction does *not* impact the compiler pieces of the proposal, and is implementable completely in `ActorTransport` implementations, however we want to call it out nevertheless, because the shape of the proposal and task-local values have been designed to support this long-term use case in mind.

With the recent release of [Swift Distributed Tracing](https://github.com/apple/swift-distributed-tracing) we made first steps towards distributed tracing becoming native to server-side swift programs. This is not the end-goal however, we want to enable distributed tracing throughout the Swift ecosystem, and by ensuring tracers can natively, and easily inter-op with distributed actors we lay down further ground work for this vision to become reality.

Note that distributed traces also mean the ability to connect traces from devices, http clients, database drivers and last but not least distributed actors into a single visualizable trace, similar to how Instruments is able to show execution time and profiles of local applications. With distributed tracing we have the ability to eventually offer such "profiler like" experiences over multiple actors, processes, and even front-end/back-end systems.

Thanks to [SE-NNNN: **Task Local Values**](https://github.com/apple/swift-evolution/pull/1245) `ActorTransport` authors can utilize all the tracing and instrument infrastructure already provided by the server side work group and instrument their transports to carry necessary trace information.

Specifically, since the distributed actor design ensures that the transport is called in the right place to pick up the apropriate values, and thus can propagate the trace information using whichever networking protocol it is using internally:

```swift
// simplified proxy implementation generated by a transport framework
func $distributed_exampleFunc() async throws {
  var message: MessageRepr = .init(name: "exampleFunc") // whatever representation the framework uses
  
  if let traceID = Task.local(\.traceID) { // pick up tracing `Baggage` or any specific values and carry them
    message.metadata.append("traceID", traceID) // simplified, would utilize `Injector` types from tracing
  }
  
  try await self.transport.send(message, to: self.id)
}
```

Using this, trace information and other information (see below where we discuss distributed deadlines), is automatically propagated across node boundaries, and allows sophisticated tracing and visualization tools to be built.

If you recall the `makeDinner` example from the 

Such distributed dinner cooking can then be visualized as:

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
- it should also be possible to communicate with WASM and "Swift in the Browser" using distributed actors and an apropriate websocket transport.

## Related Proposals

- **[Swift Concurrency Manifesto](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782#design-sketch-for-interprocess-and-distributed-compute)** - distributed actors as part of the language were first roughly outlined as a general future direction in the Concurrency Manifesto. The approach specified in this proposal takes inspiration from the manifesto, however may be seen as a reboot of the effort. We have invested a significant amount of time, research, prototypinig and implementing the approach since, and are confident in the details of the proposed model.

**Pre-requisites**

- [SE-0296: **async/await**](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md) - asynchronous functions are used to express distributed functions,
- [SE-0306: **Actors**](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md) - are the basis of this proposal, as it refines them in terms of a distributed actor model by adding a few additional restrictions to the existing isolation rules already defined by this proposal,

**Related**

- [SE-0311: **Task Local Values**](https://github.com/apple/swift-evolution/blob/main/proposals/0311-task-locals.md) - are accessible to transports, which use task local values to transparently handle request deadlines and implement distributed tracing systems (this may also apply to multi-process Instruments instrumentations),
- [SE-0295: **Codable synthesis for enums with associated values**](https://github.com/apple/swift-evolution/blob/main/proposals/0295-codable-synthesis-for-enums-with-associated-values.md) - because distributed functions relying heavily on `Codable` types, and runtimes may want to express entire messages as enums, this proposal would be tremendously helpful to avoid developers having from ever dropping down to manual Codable implementations.

**Follow-ups**

- [SE-0303: **SwiftPM Extensible build-tools**](https://github.com/apple/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md) – which will enable source-code generators, necessary to fill-in distributed function implementations by specific distributed actor transport frameworks.

**Related work**

- [Swift Cluster Membership](https://www.github.com/apple/swift-cluster-membership) ([blog](https://swift.org/blog/swift-cluster-membership/)) – cluster membership protocols are both natural to express using distributed actors, as well as very useful to implement membership for distributed actor systems.
- [Swift Distributed Tracing](https://github.com/apple/swift-distributed-tracing) – distributed actors are able to automatically and transparently participate in distributed tracing systems and instrumentation, this allows for a "profiler-like" performance debugging experience across entire fleets of servers,



## Alternatives Considered

### Discussion: Why Distributed Actors are better than "just" some RPC library?

While this may be a highly subjective and sensitive topic, we want to tackle the question up-front, so why are distributed actors better than ""just" some RPC library?

The answer lies in the language integration and the mental model developers can work with when working with distributed actors. Swift already embraces actors for its local concurrency programming, and they will be omni-present and become a familiar and useful tool for developers. It is also important to notice that any aync function may be technically performing work over network, and it is up to developers to manage such calls in order to not overwhelm the network etc. With distributed actors, such calls are more _visible_ because IDEs have the necessary information to e.g. underline or otherwise hightlight that a function is likely to hit the network and one may need to consider it's latency more, than if it was just a local call. IDEs and linters can even use this statically available information to write hints such as "hey, you're doing this distributed actor call in a tight loop - are you sure you want to do that?"

Distributed actors, unlike "raw" RPC frameworks, help developers to think about their distributed applications in terms of a network of collaborating actors, rather than having to think and carefully manage every single serialization call and network connection management between many connected peers - which we envision to be more and more important in the future of device and server programming et al. You may also refer to the [Swift Concurrency Manifesto; Part 4: Improving system architecture](https://gist.github.com/lattner/31ed37682ef1576b16bca1432ea9f782#part-4-improving-system-architecture) section on some other ideas on the topic.

This does _not_ mean that we shun RPC style libraries or plain-old HTTP clients and libraries similar to them, which may rather be expressed as non-actor types with asynchronous functions. They still absolutely have their place, and we do not envision distributed actors fully replacing them in all use-cases. We do mean however that extending the actor model to it's natural habitat (networking) will enable developers to build some kinds of interactive multi-peer/multi-node systems far more naturally than each time having to re-invent a similar abstraction layer, never quite reaching the integration smoothness as language provided integration points such as distributed actors can offer.

### Special Actor spawning APIs

One of the verbose bits of this API is that a distributed actor must be created or resolved from a specific transport. This makes sense and is _not_ accidental, but rather inherent complexity – because Swift applications often interact over various transports: within the host (e.g. a phone or mac) communicating via XPC with other processes, while at the same time communicating with server side components etc. It is important that it is clear and understandable what transport is used for what actor at construction.

#### Explicit `spawn(transport)` keyword-based API

The `spawn` word is often used in various actor runtimes

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

#### Directly adopt Akka-style Actors References `ActorRef<Message>`

Theoretically, distributed actors could be implemented as just a library, same as Akka does on the JVM. However this would be undermining both the distributed actors value proposition as well as the actual local-only actors provided by the language.

First, a quick refresher how Akka models (distributed) actors: There are two API varieties, the untyped "classic" APIs, and the current (stable since a few years) "typed" APIs. 

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

## Acknowledgments & Prior Art

We would like to acknowlage the prior art in the space of distributed actor systems which have inspired our design and thinking over the years. Most notably we would like to thank the Akka and Orleans projects, each showing independent innovation in their respective ecosystems and implementation approaches. 

We would also like to acknowlage the Erlang BEAM runtime and Elixir language for a more modern take built upon the on the same foundations. In some ways, Swift's distributed actors are not much unlike the gen_server style processes available on those platforms. While we operate in very different runtime environments and need to take different tradeoffs at times, both have been an inspiration and very useful resource when comparing to the more Akka or Orleans style actor designs.

## Source compatibility

This change is purely additive to the source language. 

The additional use of the keyword `distributed` in `distributed actor` and `distributed func` applies more restrictive requirements to the use of such an actor, however this only applies to new code, as such no existing code is impacted.

Marking an actor as distributed when it previously was not is potentially source-breaking, as it adds additional type checking requirements to the type.

## Effect on ABI stability

This proposal is ABI additive.

**TODO:** ???

## Effect on API resilience

**TODO:** ???

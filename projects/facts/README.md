# Facts

An open-source project about streaming and sourcing facts for backend applications.

## Premise

Building effective backend applications is a pain. It takes a lot of effort to design and implement workflows that complete without the risk of data loss or duplicate side effects.

Event-driven architectures help, but they're not trivial to build, and there's no consensus on how do this.

What's needed is an open-source project that facilitates writing distributed backend applications by letting users focus on the business logic and the data model, rather than on retries, timeouts, messaging, etc.

## Goals

In no particular oder:

1. Opinionated and effective structure.
    1. Facts as the fundamental unit of persistent data.
        1. A fact is an immutable piece of information that's uniquely defined by its fields.
        2. Facts can reference other facts as predecessors, establishing causality.
        3. If a fact B succeeds a fact A, and a fact C also succeeds A, then B and C happened concurrently.
        4. If two facts B and C don't share an ancestor, we cannot infer any causality between the two.
        5. Overall, this introduces partial causality and ordering.
        6. Facts can be generated in a fully decentralized fashion, without the need to have some of their fields populated by a centralized system.
            1. Traditional events (like in event-streaming or event-sourcing), on the other hand, need a sequence number that only a centralized system can provide.
    2. Commands, events, and queries.
        1. Queries return information aggregated from historical facts to users.
        2. Commands initiate actions that result in new facts, can be rejected, and can return the outcome of their processing to users.
        3. Events inform the system of external facts. Events cannot be rejected if authorized, as they're historical records.
    3. Service choreography.
        1. The entire application logic is a chain of reactions to facts.
        2. No commands, queries, or events between internal nodes. Commands, queries, and events are ingresses to the system only.

2. Streamlined declarative controls.
    1. The tool should make it easy to declare the data model and the application logic.
    2. Users shouldn't concern themselves with transport protocols, routing, retries, horizontal scaling, partitioning, data storage, etc.
    3. Backward and forward compatibility with newer data format schemata should be supported as a first-class citizen.

3. Nodes should be heavily templated:
    1. Command endpoints.
        1. Receive commands from outside the system, and publishes them for processing.
        2. Inform the caller of validation problems and of the outcome of the commands.
    2. Query endpoints.
        1. Allow users to request information aggregated from historical facts.
        2. Materialize their own data from observing the facts.
    3. Event endpoints.
        1. Allow external systems to inform the application of facts that happened outside the system.
    4. Fact processors.
        1. React to specific types of facts, by recording successor facts.
        2. Represent the bulk of the application logic.
    5. Fact observers.
        1. React to specific types of facts by informing external systems.

4. Support for multiple programming languages. Users should be allowed to express their application logic in more than 1 programming language.
    1. At the very least, we should support JVM languages, Python, C and C++, JavaScript and TypeScript, Golang, and ideally Rust.

5. Support for multiple data formats.
    1. While Thrift and Avro are great data formats, we cannot discount the possibility of emerging new formats in the future.
    2. We should allow users to configure the tool with the data format they want to use.
    3. It sounds reasonable that the tool would allow for different parts of the topology to use different data formats. This would also help with switching from a data format to another in an incremental way.

6. Support for third-party nodes and sub-systems.
    1. We should allow users to leverage nodes and sub-topologies developed by third-parties.
    2. This means we cannot assume a trusted security context for the entire topology.
    3. Nodes should be sandboxed and isolated, so that a misbehaving node cannot compromise the security of the whole system.
    4. ACLs should control what a node can and cannot do e.g., consuming and publishing specific types of data, communicating to external systems, storing and reading data, etc.

7. In-order, exactly-once processing.
    1. Facts should be processed in order, exactly once.
    2. This requires a combination of keyed routing and idempotency.
    3. Facts are pretty idempotent on their own, as they are immutable and fully defined by their fields.
    4. Keyed routing means that facts should be routed for processing to processors based on their key.
    5. A fact key is a business attribute that groups facts based on their ordered processing requirements.
        1. A bank must process transactions in order, within a bank account, or otherwise end up making wrong decisions.
        2. In this case, the various transactions modelled as facts would be keyed based on the account they are relevant to.
    6. It's imperative that all facts keyed with a key are uniquely processed by the same physical instance of a fact processor (in general, processors can scale in terms of number of instances).
    7. Multi-keyed facts should be supported (there are multiple potential solutions, to be discussed).

8. Built-in observability and auditability.
    1. Invocations should be modelled as facts.
    2. Commands, queries, and events are all invocations.
    3. Instances of facts processing (in processors or observers) are also invocations.
    4. Invocations should contain a context including the actor, a timestamp, tracing information, client origin, etc.
    5. Business facts should reference the invocations that generated them as predecessors.
    6. Users should be able to explore the facts based on invocations, including:
        1. All facts generated by an invocation.
        2. All facts and invocations generated by an actor or an authentication token.
        3. All changes made during a session.
    7. Logging should also be available to users and nodes as a primitive.

9. Efficiency.
    1. Facts should be stored efficiently.
        1. If two facts reference the same fact, then that fact shouldn't be stored more than once in the system.
        2. This includes business facts, invocations, etc.
    2. Storage should allow users to trade-off cost and performance.
        1. Tiered storage should be able to store various facts in a variety of ways, based on their temporal relevance and frequency of access.
        2. Low-cost cold storage should be an option for those facts that must be kept for audit purposes, but won't likely be processed as part of new workflows.
    3. The propagation method that makes facts available to processors and observers shouldn't introduce excessive latency.

10. Scalability.
    1. The system should scale horizontally to support large-scale deployments with sustained throughput of commands, queries, and events.
    2. The tool should be an option for a large retain bank with tens of millions of users, each initiating invocations tens of times a day.

## Notes/Thoughts

1. There is something vaguely similar, although not open-source, called [Rama](https://redplanetlabs.com/).
2. A graph database would be an interesting storage primitive for historical facts.
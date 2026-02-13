# Intro

We want to build evolvable systems and therefore we should not only focus on release new future but to maintain the complexity of the software behind them stable.

There are many way to help like Chunking and Hierarchization but we focus on Pattern Languages like Domain-Driven Design.


## Architecturally-evident code

Another important concept is Architecturally Evident Code is code whose structure, naming, and dependency rules make the system’s architecture and boundaries immediately understandable from the code itself—not just from class names, but from how the code is organized and what it is allowed to depend on.

Architecturally-evident code has two complementary aspects: extensional and intentional.

![Architecturally evident code](images/architecturally-evident-code-exstensional-intentional.png)

### Extensional

Extensional is about business components, packages, and class names.
The architecture is visible from the code structure and naming.

Example:

```text
ordering.domain.Customer
ordering.domain.CustomerRepository
ordering.api.CustomerController
```

From this, we understand there is an Ordering module and a Customer concept in the domain.

This is architecture that humans can see by reading the code.

### Intentional

Intentional is about conventions and rules for the code.
It defines how the architecture should be respected.

Examples of rules:

* Domain does not depend on infrastructure
* Controllers do not access entities directly
* Repositories handle aggregates

These rules can be checked with validation tools.

### Issues

If we add “Adapter” or a technology name, it can blur the extensional meaning if the technical name dominates the domain meaning.

```text
CustomerRepository
JpaCustomerRepository
```

Here the domain concept (Customer) should remain primary, while “Jpa” only indicates the implementation.

Also, if we only see `CustomerRepository`, we might guess Customer is an aggregate, but this is not explicit.

Another issue is validation: it is hard to verify whether a repository is implemented correctly. For example, if a repository loads an aggregate that contains full object references to other aggregates instead of only their IDs, this may violate aggregate boundaries, but it is not always easy for a tool to detect.

This shows that naming alone is not enough; architectural rules and aggregate boundaries need explicit definitions and validation rules.


### Solutions

We can validate intentional rules with tools like ArchUnit.

Additionally, we can make concepts explicit in code (for example, marking aggregates explicitly) so they are visible to both humans and tools.

The tools the help doing this is jMolecules.

# jMolecules

Here is your notes with those two points integrated, keeping your content and adding only the requested clarifications.

---

# jMolecules

jMolecules provides JARs (DDD, events, etc.) that contain interfaces and annotations to help enforce intentional architectural rules in code.

![jMolecules](images/jmolecules-jar.png)

With jMolecules:

* Code is more expressive (for example, you can immediately see if a class is an aggregate without checking for a repository)
* Rules can be verified
* Documentation can be generated and stays in sync with code

You can choose to work with interfaces or annotations.

* Interfaces are more powerful because the compiler helps enforce correctness
* Annotations are lighter but rely more on external validation tools
* Annotations are less compiler-enforced, but not inherently weaker if combined with validation tools

Example: Aggregate interfaces force you to specify the aggregate type and its ID type. If the ID type does not implement the required interface, the code does not compile.

```java
public class Order 
  extends AbstractAggregateRoot<Order> 
  implements AggregateRoot<Order, OrderIdentifier> {

    private final List<LineItem> lineItems = new ArrayList<>();

    public static class OrderIdentifier 
        implements Identifier { }

    static class LineItem 
        implements Entity<Order, LineItemId> { }
}
```

A downside is that when using interfaces, the jMolecules JAR must be on the classpath for compilation because your code directly depends on its types. With annotations, compile-time coupling is lower since they are metadata and do not shape your type hierarchy.

In practice, annotation usage also usually requires the dependency to compile (you still reference the annotation types). The real difference is about **compile-time enforcement vs. metadata-based validation**, not the mere presence of the JAR.

Another tool is JQAssistant. It is more powerful than ArchUnit but also more complex. It scans your code, Git history, and commits. You can define rules and generate reports.

Example: scanning commits to identify which classes change most often to find hotspots.

In Visual Studio Code (and planned for IntelliJ IDEA), there is a logical structure view where classes can be grouped by types. 

You can also define your own types and groupings.

Example:

```java
@Stereotype(groups = "restbucks")
```
You can create logical groups (e.g., all DTOs).

You can also define hierarchies. By annotating classes with architectural stereotypes (e.g., hexagonal architecture), you can see them nested logically (e.g., adapters containing DTO types).

# Eliminate Boilerplate

We want to have a separation between the domain model and infrastructure models (API DTOs and JPA entities). If we do this, then we have a lot of boilerplate code.

Does it make sense to have this split? Usually, if we own the application (in modern team setups with microservices), we also own the database and we do not want differences between the two (JPA entity and domain model), because a difference is logically wrong and also too difficult to maintain.

![Legacy vs Modern applications ownership](images/legacy-vs-modern-application-ownership.png)

On the API side instead, we need this split and mapping. We cannot change the API model as we want because we have consumers.

If we go with domain model = JPA entity, then we will in any case have many annotations on our class (e.g., relationships, no-argument constructors, etc.). But if we consider the Order already as an aggregate, many of these annotations become conceptually redundant because they are there by design if it is an aggregate (e.g., it must have an id).

```java
@Entity
@NoArgsConstructor(force = true)
@EqualsAndHashCode(of = "id")
@Table(name = "SAMPLE_ORDER")
@Getter
public class Order {

    private final @EmbeddedId OrderId id;

    @OneToMany(cascade = CascadeType.ALL, …)
    private List<LineItem> lineItems;

    private CustomerId customerId;

    public Order(CustomerId customerId) {
        this.id = OrderId.of(UUID.randomUUID());
        this.customerId = customerId;
    }

    @Value
    @RequiredArgsConstructor(staticName = "of")
    @NoArgsConstructor(force = true)
    public static class OrderId implements Serializable {
        private static final long serialVersionUID = …;
        private final UUID orderId;
    }
}
```

Byte Buddy is a library to manipulate bytecode of compiled classes. With the jMolecules Byte Buddy plugin, we can use jMolecules interfaces like before, and then Byte Buddy will generate the boilerplate annotations for us based on jMolecules rules.

```java
@Table(name = "SAMPLE_ORDER")
@Getter
public class Order implements AggregateRoot<Order, OrderId> {

    private final OrderId id;
    private List<LineItem> lineItems;
    private Association<Customer, CustomerId> customer;

    public Order(Customer customer) {
        this.id = OrderId.of(UUID.randomUUID());
        this.customer = Association.forAggregate(customer);
    }

    @Value(staticConstructor = "of")
    public static class OrderId implements Identifier {
        private final UUID orderId;
    }
}
```

In the IDE logs, however, you can see everything that is happening:

```text
[INFO] □─ example.order.Order
[INFO] ├─ JPA - Adding @j.p.Entity.
[INFO] ├─ JPA - Adding default constructor.
[INFO] ├─ JPA - Adding nullability verification using new callback methods.
[INFO] ├─ JPA - Defaulting id mapping to @j.p.EmbeddedId().
[INFO] ├─ JPA - Defaulting lineItems mapping to @j.p.JoinColumn(…).
[INFO] ├─ JPA - Defaulting lineItems mapping to @j.p.OneToMany(…).
[INFO] ├─ Spring Data JPA - Implementing o.s.d.d.Persistable<e.o.Order>
```

There are many integrations, for example Jackson for serialization and deserialization. For identifiers, this allows them to be serialized as a single value and not a nested object.

jMolecules provides a command-line tool to scaffold classes for you, for example for Spring Modulith modules. If you create an aggregate, it can also generate the repository and unit and integration tests (jMolecules scaffolding).

# Spring Modulith

## Motivation

jMolecules removes the need to structure your application using packages named after the chosen architectural model (such as adapters or repositories). Overall, this can be considered wrong because it adds a high-level abstraction at a lower level (Java packages).

**So what is the right way to split the application? An architectural-level split is not ideal because it is too high level. Should it be based only on the domain?**

If we group repositories together and controllers together to achieve decoupling (as learned from architectural standards like Onion or Hexagonal), we create low cohesion between elements (for example, controllers for orders and customers). This also causes a chain reaction when making changes in the code. Decoupling has its cost.

![Chain reaction with decoupling but low cohesion](images/decupling-low-cohesion-changes-chain.png)

![Classic layers low cohesion](images/classic-low-cohesion.png)

![Onion layers low cohesion](images/onion-low-cohesion.png)

Cohesion is coupling in the right places and is also a result of good modularization.

![Chain reaction high cohesion](images/high-cohesion-changes-chain.png)

We want to create cohesion in modules. If we design only for decoupling, we can end up without cohesion.


## Modularity

Onion architecture (or any similar architecture) is a technical separation, not a domain separation. Spring Modulith pushes toward domain decomposition. However, the best solution is not to cut the domain, but to split the onion into multiple modules.

[THIS IS JUST A RANDOM QUESTION TO REFLECT ON: Does it make sense to split your app so that you can change the persistence type or framework? Is this really possible? How much does it affect daily work?]

![Normal Onion Architecture](images/onion-architecture.png)

The idea is to create slices or packages per domain and then use jMolecules to enrich classes with the correct types. This also allows making public only what is supposed to be accessed. If a slice or domain becomes too big, it can be further split into subpackages (for example, order.web for controllers, DTOs, mappings, etc.).

The important point is to have higher cohesion at the domain level. Inside, subpackages can still be used depending on the module’s complexity. Be flexible on a package-by-package basis.

![Normal Onion Architecture Slices](images/onion-architecture-domain-slices.png)

Modules can be connected by exposing only certain parts of the code to reduce coupling and increase control. Usually, these connections should be in the application layer.

To configure Spring Modulith, you import the module in the POM, and it enforces a certain structure on subfolders. Normally, without explicit exceptions defined by the developer, there cannot be interactions between subpackages at a higher level.

![Packages separation](images/modulith-packages.png)

If you have a monolith and want to transform it into a modulith, you can customize which packages are considered modules (with annotations or programmatically, for example using `ApplicationModuleDetectionStrategy`).

## Testing

You can verify Spring Modulith rules (package rules) with a simple test:

```java
var modules = ApplicationModules.of(MyApplication.class);
modules.verify(…);
```

Spring Boot tests provide annotations to load only a given horizontal slice into the test context.

![Spring Test horizontal slice](images/spring-test-horizontal-slice.png)

However, with Modulith, you should be able to load only a specific module in the test context. `@ApplicationModuleTest` bootstraps only the necessary modules.

![Spring Modulith Test vertical slice](images/spring-modulith-test-vertical-slice.png)

If you need classes from other modules, you can use `@MockitoBean` to mock them. You can also configure Spring Modulith to bootstrap additional required modules with `@ApplicationModuleTest(mode = ALL_DEPENDENCIES)`.

Spring Modulith’s JUnit integration can detect which module you changed and run tests only for modules that depend on the changed one (based on commits to the main branch). You can also do this in CI pipelines by calculating the delta with respect to the last successful build.

![Spring Modulith Test vertical slice](images/spring-modulith-junit-integration.png)

## Documentation

The higher the level you go (for example, company-wide), the less the documentation changes because it is more abstract. At the code level, it changes very often, and we do not want to maintain it manually.

![Documentation layers](images/documentation-layers.png)

Spring Modulith can generate documentation that explains each module and their interactions or dependencies.

![Modules documentation](images/modulith-packages.png)

You can also see the jMolecules stereotypes. If you generate Javadoc as well, you can link it to the classes in the Spring Modulith documentation to get more details.

![Modules documentation with jMolecules stereotypes](images/documentation-stereotypes-jmolecules.png)


## Communication

How do modules interact? There are three possibilities:

* Via Spring beans, using direct class calls in a synchronous way
* Via `@EventListener`, which is synchronous
* Via an application module listener in an asynchronous way


### Transactional communication

What if we have a transaction that encompasses multiple modules?

![Transactional](images/transactional.png)

We can use normal Java beans, but this requires the additional modules to be started and included in tests.

![Transactional with Java Beans](images/java-beans-transactional.png)

We can instead use events so that we no longer depend directly on other services. This allows cleaner tests that focus on a single module and only verify that a message is published, without checking how other modules react.

![Transactional with Events](images/events-transactional.png)

In this way, even if an exception is thrown in the consuming module, the transaction is maintained.

What if we have multiple listeners? Does it also roll back database interactions in other modules? If they use the same resource, yes. But what if another resource is used, such as SMTP? This can be a problem because it can lock the database for a long time with respect to the database transaction. Also, what if an email is sent and then the transaction rolls back?

![Transactional with multiple consumers](images/transactional-with-different-resources.png)

We can switch from `@EventListener` to `@TransactionalEventListener`. This indicates that the publisher is in a transaction and waits until the transaction commits before sending the event to the consumer. The disadvantage is that the checkout takes longer because we wait for the email to be sent, and the database transaction may remain open even after commit.

![Transactional with TransactionalEventListener](images/transactionaleventlistener.png)


If we make the listener `@Async`, it completes immediately because handling is done on another thread, and the commit or end is immediate.

![Transactional with Async TransactionalEventListener](images/asynctransactionaleventlistener.png)

What happens if SMTP fails or its module goes down? Spring Modulith uses something similar to the outbox pattern (within the transaction) for events, ensuring that the listener must succeed before the event is removed from the outbox table. Spring Modulith does not automatically resubmit the event, but APIs are available to retrieve this information from the database and retry as needed.

The outbox pattern processes events in order and does not move to the next element until the previous one succeeds, guaranteeing ordering. In Spring Modulith, it is not exactly the same. (An example of a real outbox for Spring Boot is Namastack Outbox.) It is possible to switch from the Spring Modulith “outbox” to a real one.

### Externalize

We can externalize messaging with `@Externalized` and use various technologies such as Kafka. This also comes with success and failure handling similar to the outbox pattern.

## Testing

We can also test these interactions by calling a public interface of a module and waiting for a specific event to be published using the Scenario API. In this way, we do not need to care whether the event is asynchronous or synchronous. This also works with multiple interacting modules (for example, calling an input in module A and waiting for an event published by module B).

```java
@ApplicationModuleTest
class CheckoutTests {

    private final Checkout checkout;

    @Test
    void completesOrder(Scenario scenario) {
        var order = new Order(…);

        scenario.stimulate(() -> checkout.complete(order))
                .andWaitForEventOfType(InventoryUpdated.class)
                .matchingMappedValue(InventoryUpdated::getOrderId, order.getId())
                .toArrive();
    }
}
```


## Observability

We can have distributed tracing in Spring Modulith similar to what we have in a microservices system. It enhances input and output interfaces with Micrometer. We can also customize what is published as metrics using `Bean ModulithEventMetricsCustomizer`.


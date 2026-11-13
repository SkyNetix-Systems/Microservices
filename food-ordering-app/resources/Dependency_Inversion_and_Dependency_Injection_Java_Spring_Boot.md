# Dependency Inversion Principle (DIP) and Dependency Injection (DI)

## Detailed Java and Spring Boot Guide

## 1. Why these concepts matter

A Java class often needs another class to perform work. For example,
`OrderService` needs a payment capability, `FlightBookingService` needs
seat inventory, and `NotificationService` needs an email or SMS sender.

The design question is: should a business class create and depend
directly on a specific implementation, or should it depend on a contract
and receive an implementation from outside?

Two related concepts help answer this:

-   **Dependency Inversion Principle (DIP)** is a SOLID design
    principle. It guides the direction of dependencies.
-   **Dependency Injection (DI)** is a technique for supplying an object
    with the dependencies it needs.

They are related, but they are not the same.

# Part A --- Dependency Inversion Principle

## 2. What is DIP?

DIP is the **D** in SOLID. It has two central rules:

1.  High-level modules should not depend directly on low-level modules.
    Both should depend on abstractions.
2.  Abstractions should not depend on details. Details should depend on
    abstractions.

  -----------------------------------------------------------------------------
  Term                    Meaning                 Example
  ----------------------- ----------------------- -----------------------------
  High-level module       Business policy or      `FlightBookingService`
                          application logic       

  Low-level module        Technical               `KafkaEventPublisher`
                          implementation          

  Abstraction             Contract describing a   `FlightEventPublisher`
                          capability              

  Detail                  Concrete implementation `KafkaFlightEventPublisher`
  -----------------------------------------------------------------------------

A business service should express what capability it needs, rather than
being permanently tied to one technical provider.

## 3. Real-world analogy

Imagine a hotel that needs to arrange airport transportation. If its
booking process is permanently tied to one taxi company, changing
providers requires changing the booking process.

Instead, the hotel can depend on a `TransportProvider` contract. A taxi
company, shuttle service, or other provider can implement that contract.
The hotel booking logic depends on the capability, not on a particular
provider.

An interface often represents this kind of contract in Java.

## 4. Tightly coupled code --- without DIP

``` java
public class StripePaymentGateway {
    public void charge(double amount) {
        System.out.println("Charging through Stripe: " + amount);
    }
}
```

``` java
public class OrderService {
    private final StripePaymentGateway paymentGateway =
            new StripePaymentGateway();

    public void placeOrder(double amount) {
        paymentGateway.charge(amount);
        System.out.println("Order placed");
    }
}
```

### Problems

-   `OrderService` is directly tied to Stripe.
-   Replacing Stripe with another provider requires changing the
    service.
-   Unit testing the service independently is harder.
-   Business logic knows about an infrastructure detail.
-   The service creates and controls its own dependency.

## 5. Apply DIP using an interface

First define an abstraction:

``` java
public interface PaymentGateway {
    void charge(double amount);
}
```

Implement it with Stripe:

``` java
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Charging through Stripe: " + amount);
    }
}
```

Another provider can implement the same contract:

``` java
public class PayPalPaymentGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Charging through PayPal: " + amount);
    }
}
```

Now the business service depends on the abstraction:

``` java
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void placeOrder(double amount) {
        paymentGateway.charge(amount);
        System.out.println("Order placed");
    }
}
```

### What changed?

-   `OrderService` depends on `PaymentGateway`, not Stripe.
-   Stripe and PayPal both implement the contract.
-   The implementation can be supplied from outside.
-   The business service does not need to know which provider it
    receives.

This is the central idea of Dependency Inversion.

## 6. Dependency direction

### Before DIP

``` text
OrderService
    |
    | directly depends on
    v
StripePaymentGateway
```

### After DIP

``` text
              PaymentGateway
                (interface)
                /        \
               /          \
              v            v
       OrderService   StripePaymentGateway
                      PayPalPaymentGateway
```

`OrderService` uses the abstraction. Concrete implementations realize
the abstraction. Application configuration chooses which implementation
to supply.

## 7. United Airlines microservices example

Suppose a Flight Operations service needs to publish a flight-disruption
event.

A business service directly using Kafka APIs is coupled to Kafka:

``` java
public class FlightDisruptionService {
    private final KafkaTemplate<String, String> kafkaTemplate;

    public FlightDisruptionService(
            KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void reportDisruption(String flightNumber) {
        kafkaTemplate.send(
            "flight-disruption-events",
            flightNumber,
            "Flight disrupted"
        );
    }
}
```

This example is simplified; a real `KafkaTemplate` needs Spring Kafka
configuration.

Introduce an application-level port:

``` java
public interface FlightEventPublisher {
    void publishDisruption(FlightDisruptionEvent event);
}
```

A simple event record:

``` java
public record FlightDisruptionEvent(String flightNumber) {}
```

The business service depends on the port:

``` java
public class FlightDisruptionService {
    private final FlightEventPublisher eventPublisher;

    public FlightDisruptionService(FlightEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void reportDisruption(String flightNumber) {
        var event = new FlightDisruptionEvent(flightNumber);
        eventPublisher.publishDisruption(event);
    }
}
```

Kafka is hidden behind an adapter:

``` java
@Component
public class KafkaFlightEventPublisher
        implements FlightEventPublisher {

    private final KafkaTemplate<String, FlightDisruptionEvent> kafkaTemplate;

    public KafkaFlightEventPublisher(
            KafkaTemplate<String, FlightDisruptionEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    @Override
    public void publishDisruption(FlightDisruptionEvent event) {
        kafkaTemplate.send(
            "flight-disruption-events",
            event.flightNumber(),
            event
        );
    }
}
```

In this design:

-   `FlightDisruptionService` contains application/business logic.
-   `FlightEventPublisher` is an abstraction or port.
-   `KafkaFlightEventPublisher` is an infrastructure adapter.
-   Kafka is a technical implementation detail behind the adapter.

This approach is common in Hexagonal Architecture (Ports and Adapters)
and Clean Architecture.

**Important:** DIP does not mean business logic can never use a
framework. It means important business policies should not be
unnecessarily controlled by concrete technical details.

# Part B --- Dependency Injection

## 8. What is DI?

Dependency Injection is a technique in which an object receives the
objects it depends on from an external source, rather than creating
those dependencies itself.

### Without DI

``` java
public class OrderService {
    private final PaymentGateway paymentGateway =
            new StripePaymentGateway();
}
```

The service creates its own dependency.

### With DI

``` java
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

The service declares what it needs. Another component supplies it.

That supplier might be: - Spring's IoC container. - A manually written
factory. - Application configuration code. - A unit test that passes a
mock or fake.

DI is not limited to Spring. Spring automates dependency injection and
bean management.

## 9. Three common types of DI

### 9.1 Constructor injection

Dependencies are passed through the constructor.

``` java
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void placeOrder(double amount) {
        paymentGateway.charge(amount);
    }
}
```

Advantages: - Required dependencies are explicit. - `final` fields
support immutability. - Unit testing is straightforward. - The object
cannot be constructed without its required dependency. - Too many
dependencies become visible as a design smell.

**Recommended for required dependencies in Spring applications.**

### 9.2 Setter injection

Dependencies are supplied through setter methods.

``` java
public class NotificationService {
    private MessageSender messageSender;

    public void setMessageSender(MessageSender messageSender) {
        this.messageSender = messageSender;
    }

    public void notifyCustomer(String message) {
        messageSender.send(message);
    }
}
```

Useful when a dependency is optional or can legitimately be changed
after construction. The object may exist before the dependency has been
configured, so guard against use before configuration.

### 9.3 Field injection

The framework injects directly into a field.

``` java
@Service
public class OrderService {
    @Autowired
    private PaymentGateway paymentGateway;
}
```

It is concise, but generally not preferred for application code because
dependencies are less visible, fields cannot be `final`, and plain unit
tests cannot easily construct the class with its dependency. Prefer
constructor injection for required dependencies.

## 10. What is IoC?

IoC means **Inversion of Control**. It is the broader idea of moving
responsibility for object creation and wiring from application classes
to an external container or framework.

Without a container:

``` java
PaymentGateway gateway = new StripePaymentGateway();
OrderService service = new OrderService(gateway);
```

With Spring, the application declares components and Spring creates and
wires them:

``` java
@Component
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Payment processed: " + amount);
    }
}
```

``` java
@Service
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

Relationship: - IoC is the broader principle of handing control to an
external mechanism. - DI is one way to implement IoC. - Spring's IoC
container creates, configures, wires, and manages beans.

# Part C --- DI in Spring Boot

## 11. Complete Spring Boot example

### Step 1: Define the abstraction

``` java
public interface PaymentGateway {
    void charge(double amount);
}
```

### Step 2: Implement the abstraction

``` java
import org.springframework.stereotype.Component;

@Component
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Stripe charged: " + amount);
    }
}
```

`@Component` makes the class a candidate for Spring bean registration
when component scanning includes its package.

### Step 3: Inject it into a service

``` java
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void placeOrder(double amount) {
        paymentGateway.charge(amount);
        System.out.println("Order placed successfully");
    }
}
```

`@Service` marks the class as a service component. Spring registers it
as a bean and resolves its constructor dependency.

### Step 4: Inject the service into a REST controller

``` java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public String placeOrder(@RequestParam double amount) {
        orderService.placeOrder(amount);
        return "Order placed";
    }
}
```

### Step 5: What Spring does at startup

At a high level:

1.  Scans configured packages for components.
2.  Registers bean definitions.
3.  Creates beans according to their scope and lifecycle.
4.  Inspects constructor dependencies.
5.  Finds a matching `PaymentGateway` bean.
6.  Supplies it to `OrderService`.
7.  Supplies `OrderService` to `OrderController`.

The real Spring lifecycle includes more details, but this is the
essential wiring flow.

### Step 6: Test the endpoint

With the application running:

``` http
POST http://localhost:8080/api/orders?amount=2500
```

Expected response:

``` text
Order placed
```

Expected application log:

``` text
Stripe charged: 2500.0
Order placed successfully
```

## 12. Multiple implementations: `@Primary` and `@Qualifier`

Suppose two beans implement `PaymentGateway`:

``` java
@Component
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Stripe: " + amount);
    }
}
```

``` java
@Component
public class PayPalPaymentGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("PayPal: " + amount);
    }
}
```

Spring now has two candidates for a single `PaymentGateway` constructor
parameter. It needs a selection rule.

### Option 1: `@Primary`

``` java
@Component
@Primary
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Stripe: " + amount);
    }
}
```

`@Primary` makes Stripe the default candidate when multiple beans match
and no more specific qualifier is supplied.

### Option 2: `@Qualifier`

``` java
@Service
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(
            @Qualifier("payPalPaymentGateway")
            PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

`@Qualifier` narrows the candidate by bean name or qualifier value.

### Option 3: Inject all implementations

``` java
@Service
public class PaymentService {
    private final List<PaymentGateway> paymentGateways;

    public PaymentService(List<PaymentGateway> paymentGateways) {
        this.paymentGateways = paymentGateways;
    }
}
```

Spring can inject all matching beans as a collection. The application
can choose one according to business rules.

# Part D --- DIP vs DI

## 13. Differences

  -----------------------------------------------------------------------
  DIP                                 DI
  ----------------------------------- -----------------------------------
  A SOLID design principle            A technique for supplying
                                      dependencies

  Focuses on dependency direction     Focuses on object construction and
                                      wiring

  High-level and low-level modules    An object receives its
  depend on abstractions              collaborators from outside

  Reduces coupling between business   Avoids making an object create all
  policy and implementation details   its own dependencies

  Can be applied without Spring       Can be used with or without Spring

  Example: `OrderService` depends on  Example: Spring passes
  `PaymentGateway`                    `StripePaymentGateway` into
                                      `OrderService`
  -----------------------------------------------------------------------

### How they work together

DIP defines the desired dependency structure. DI is one way to assemble
objects that follow that structure.

You can use DI without properly applying DIP. For example, injecting a
concrete `StripePaymentGateway` into `OrderService` is DI, but the
service remains directly coupled to Stripe.

You can also apply DIP without a DI framework by manually constructing
and passing dependencies.

# Part E --- Unit Testing

## 14. Why DI helps testing

Because a class receives dependencies from outside, a test can pass a
mock or fake instead of connecting to a real external system.

Example using JUnit 5 and Mockito:

``` java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private PaymentGateway paymentGateway;

    private OrderService orderService;

    @BeforeEach
    void setUp() {
        orderService = new OrderService(paymentGateway);
    }

    @Test
    void shouldChargePaymentWhenOrderIsPlaced() {
        orderService.placeOrder(2500);

        verify(paymentGateway).charge(2500);
    }
}
```

The test does not need to connect to Stripe.

Benefits: 1. Faster unit tests. 2. No external payment network required.
3. More predictable results. 4. Easier testing of success and failure
cases. 5. Business logic can be tested independently of infrastructure.

A mock verifies interactions; a fake can provide a lightweight working
implementation. Choose the test double that fits the test.

# Part F --- Mistakes and Best Practices

## 15. Common mistakes

### Mistake 1: Treating DIP and DI as synonyms

DIP is a principle; DI is a technique.

### Mistake 2: Injecting a concrete class everywhere

This is DI:

``` java
public OrderService(StripePaymentGateway gateway) {
    this.gateway = gateway;
}
```

But the service still depends on a concrete provider. If
substitutability is important, depend on an abstraction such as
`PaymentGateway`.

### Mistake 3: Creating dependencies inside business services

Avoid:

``` java
private final PaymentGateway gateway = new StripePaymentGateway();
```

Prefer constructor injection.

### Mistake 4: Using field injection by default

Prefer constructor injection for required dependencies.

### Mistake 5: Creating interfaces for every class automatically

An interface is useful when it represents a meaningful boundary,
capability, variation point, or contract. Creating interfaces without a
design benefit can add unnecessary complexity.

### Mistake 6: Assuming DI automatically creates a loosely coupled architecture

DI can inject a concrete class just as easily as an interface. Good
boundaries and appropriate abstractions are still required.

## 16. Best practices

-   Prefer constructor injection for required dependencies.
-   Use `private final` fields where possible.
-   Depend on meaningful abstractions at architectural boundaries.
-   Keep business logic independent of infrastructure APIs where
    practical.
-   Use `@Primary` or `@Qualifier` when multiple beans implement the
    same interface.
-   Review classes with too many dependencies; they may have too many
    responsibilities.
-   Use mocks or fakes to test business logic in isolation.
-   Keep object creation and application wiring in the
    composition/configuration layer.
-   Do not introduce interfaces merely to satisfy a rule; introduce them
    when they improve the design.

# Part G --- Interview Preparation

## 17. What is Dependency Inversion Principle?

**Sample answer:**

Dependency Inversion Principle is the D in SOLID. It states that
high-level modules should not depend directly on low-level modules; both
should depend on abstractions. Abstractions should not depend on
implementation details. For example, an `OrderService` should depend on
a `PaymentGateway` interface rather than directly on
`StripePaymentGateway`. This reduces coupling and makes implementations
easier to replace and test.

## 18. What is Dependency Injection?

**Sample answer:**

Dependency Injection is a technique in which an object receives its
required dependencies from an external source rather than creating those
dependencies itself. In Spring Boot, the IoC container creates and
manages beans and injects dependencies, commonly through constructors.
Constructor injection makes required dependencies explicit and improves
testability.

## 19. Are DIP and DI the same?

**Sample answer:**

No. DIP is a design principle that guides how modules should depend on
abstractions. DI is a technique for providing dependencies from outside
an object. Spring uses DI to wire beans, and DI can help implement a
design that follows DIP. However, DI alone does not guarantee DIP.

## 20. Which injection type is recommended in Spring Boot?

**Sample answer:**

Constructor injection is generally recommended for required
dependencies. It makes dependencies explicit, supports immutable `final`
fields, simplifies unit testing, and ensures the object cannot be
constructed without its required collaborators.

## 21. What is the difference between IoC and DI?

**Sample answer:**

Inversion of Control is the broader concept of transferring control of
object creation or other responsibilities to an external framework or
container. Dependency Injection is one way to achieve IoC by supplying
dependencies to an object. Spring's IoC container manages beans and
injects their dependencies.

## 22. Quick revision table

  -----------------------------------------------------------------------
  Concept                             One-line memory aid
  ----------------------------------- -----------------------------------
  DIP                                 Depend on abstractions, not
                                      concrete implementation details.

  DI                                  Receive dependencies from outside
                                      instead of creating them
                                      internally.

  IoC                                 Transfer control of object creation
                                      and wiring to an external
                                      container.

  Spring IoC Container                Creates, configures, wires, and
                                      manages Spring beans.

  Constructor Injection               Supplies required dependencies
                                      through a constructor.

  Setter Injection                    Supplies or changes a dependency
                                      through a setter.

  Field Injection                     Injects directly into a field,
                                      usually through framework
                                      reflection.

  `@Component`                        Registers a class as a component
                                      candidate for Spring bean creation.

  `@Service`                          Marks a service-layer component for
                                      Spring.

  `@Autowired`                        Requests dependency injection;
                                      often unnecessary on a single
                                      constructor.

  `@Primary`                          Selects a default candidate when
                                      multiple beans match.

  `@Qualifier`                        Specifies which matching bean
                                      should be injected.
  -----------------------------------------------------------------------

## 23. Final takeaway

Remember the concepts in this order:

1.  **DIP:** Design the business service to depend on a meaningful
    abstraction.
2.  **DI:** Supply a suitable implementation from outside the service.
3.  **IoC:** Let Spring manage object creation and dependency wiring in
    a Spring Boot application.

For example, `OrderService` depends on `PaymentGateway`, and Spring
injects `StripePaymentGateway`.

That is how Dependency Inversion and Dependency Injection work together
in a typical Java Spring Boot application.

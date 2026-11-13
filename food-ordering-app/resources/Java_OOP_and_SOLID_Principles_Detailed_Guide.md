# Object-Oriented Programming (OOP) Concepts and SOLID Principles

## A Detailed Java Guide with Practical Examples

This guide explains the core OOP concepts and the five SOLID design
principles using Java examples. It is intended for Java backend
developers preparing for Spring Boot development and Java Architect
interviews.

------------------------------------------------------------------------

# Part 1 --- Object-Oriented Programming

## 1. What is Object-Oriented Programming?

Object-Oriented Programming (OOP) is a programming approach in which
software is designed around **objects**. An object combines:

-   **State** --- the data or attributes it holds.
-   **Behavior** --- the operations or methods it can perform.

For example, a `Flight` object may hold a flight number, origin,
destination, and status. It may also provide operations such as delay,
cancel, or update status.

``` java
public class Flight {
    private String flightNumber;
    private String origin;
    private String destination;
    private String status;

    public void cancel() {
        this.status = "CANCELLED";
    }

    public String getStatus() {
        return status;
    }
}
```

OOP helps organize code into reusable, maintainable units that model
business concepts.

## 2. Class and Object

### 2.1 Class

A class is a blueprint or definition that describes the fields and
behavior its objects can have.

``` java
public class Passenger {
    private String name;
    private String passportNumber;

    public Passenger(String name, String passportNumber) {
        this.name = name;
        this.passportNumber = passportNumber;
    }

    public void checkIn() {
        System.out.println(name + " checked in");
    }
}
```

### 2.2 Object

An object is a runtime instance of a class.

``` java
Passenger passenger =
        new Passenger("Amit", "P123456");

passenger.checkIn();
```

Here: - `Passenger` is the class. - `passenger` is a reference
variable. - `new Passenger(...)` creates an object. - The object has its
own state.

### 2.3 Class vs Object

  Class                        Object
  ---------------------------- ---------------------------------
  Blueprint or definition      Runtime instance
  Defines fields and methods   Holds actual field values
  Declared in source code      Created at runtime
  Example: `Passenger`         Example: one passenger instance

### 2.4 Where are objects stored?

In typical Java execution, an object created with `new` is allocated on
the heap, while local variables and references are commonly associated
with a thread's stack frame. JVM optimizations can change physical
allocation details, so this is a useful conceptual model rather than an
absolute implementation guarantee.

------------------------------------------------------------------------

## 3. The Four Main Pillars of OOP

The four commonly taught pillars are:

1.  Encapsulation
2.  Abstraction
3.  Inheritance
4.  Polymorphism

## 4. Encapsulation

### 4.1 What is encapsulation?

Encapsulation means bundling data and the methods that operate on that
data inside a class, while controlling access to the object's internal
state.

In Java, encapsulation is commonly implemented using: - `private`
fields. - Public methods that expose safe operations. - Validation
inside methods or constructors. - Avoiding unrestricted direct
modification of fields.

### 4.2 Without encapsulation

``` java
public class BankAccount {
    public double balance;
}
```

Usage:

``` java
BankAccount account = new BankAccount();
account.balance = -50000; // Invalid state is possible
```

Any caller can directly change the balance, even to an invalid value.

### 4.3 With encapsulation

``` java
public class BankAccount {
    private double balance;

    public BankAccount(double openingBalance) {
        if (openingBalance < 0) {
            throw new IllegalArgumentException(
                    "Opening balance cannot be negative");
        }
        this.balance = openingBalance;
    }

    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException(
                    "Deposit must be positive");
        }
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0 || amount > balance) {
            throw new IllegalArgumentException(
                    "Invalid withdrawal amount");
        }
        balance -= amount;
    }

    public double getBalance() {
        return balance;
    }
}
```

### 4.4 Benefits of encapsulation

-   Protects object state.
-   Prevents invalid values.
-   Centralizes validation and business rules.
-   Reduces dependency on internal implementation.
-   Makes refactoring easier.
-   Improves maintainability.

### 4.5 Important interview point

Encapsulation is not simply writing getters and setters for every field.
If a setter allows callers to place an object into an invalid state, the
class has not achieved strong encapsulation.

Prefer meaningful business operations such as `deposit()` and
`withdraw()` over unrestricted setters such as `setBalance()`.

------------------------------------------------------------------------

## 5. Abstraction

### 5.1 What is abstraction?

Abstraction means exposing essential behavior while hiding unnecessary
implementation details.

A caller should know what an object can do without needing to know every
internal step.

For example, a passenger can request a seat reservation without knowing
how the inventory database, locking, or messaging system works.

### 5.2 Abstraction using an interface

``` java
public interface SeatReservation {
    void reserve(String flightNumber, String seatNumber);
}
```

Implementation:

``` java
public class SeatReservationService
        implements SeatReservation {

    @Override
    public void reserve(String flightNumber, String seatNumber) {
        System.out.println(
                "Reserving " + seatNumber + " on " + flightNumber);
    }
}
```

Client:

``` java
public class BookingService {
    private final SeatReservation reservation;

    public BookingService(SeatReservation reservation) {
        this.reservation = reservation;
    }

    public void bookSeat() {
        reservation.reserve("UA245", "12A");
    }
}
```

The client uses the contract and does not need to know the
implementation details.

### 5.3 Abstraction using an abstract class

``` java
public abstract class Notification {
    public abstract void send(String message);

    public void log(String message) {
        System.out.println("Notification: " + message);
    }
}
```

Concrete implementation:

``` java
public class EmailNotification extends Notification {
    @Override
    public void send(String message) {
        log(message);
        System.out.println("Sending email: " + message);
    }
}
```

### 5.4 Interface vs abstract class

  -----------------------------------------------------------------------
  Interface                           Abstract class
  ----------------------------------- -----------------------------------
  Defines a contract/capability       Provides a common base and partial
                                      implementation

  A class can implement multiple      A class can extend only one class
  interfaces                          

  Fields are implicitly               Can have instance fields and state
  `public static final` constants     

  Methods can be abstract, default,   Can have abstract and concrete
  static, and private (subject to     methods
  Java rules)                         

  No ordinary instance constructor    Can have constructors

  Useful for capabilities and         Useful for shared state or common
  boundaries                          implementation
  -----------------------------------------------------------------------

An interface can have default and static methods. Modern Java also
permits private interface methods to help share implementation between
interface methods.

### 5.5 When should you use each?

Use an interface when: - Multiple unrelated classes need to provide the
same capability. - You want to define an application boundary or port. -
Implementations may vary. - You want clients to depend on a contract.

Use an abstract class when: - Related classes share state or common
implementation. - You need a protected base implementation. - The
subclasses have a genuine "is-a" relationship.

------------------------------------------------------------------------

## 6. Inheritance

### 6.1 What is inheritance?

Inheritance allows one class (child/subclass) to acquire accessible
fields and methods from another class (parent/superclass), and to extend
or specialize its behavior.

It represents an **is-a** relationship.

### 6.2 Java example

``` java
public class Employee {
    protected String name;

    public Employee(String name) {
        this.name = name;
    }

    public void work() {
        System.out.println(name + " is working");
    }
}
```

``` java
public class Pilot extends Employee {
    public Pilot(String name) {
        super(name);
    }

    public void flyAircraft() {
        System.out.println(name + " is flying");
    }
}
```

Usage:

``` java
Pilot pilot = new Pilot("Rahul");
pilot.work();
pilot.flyAircraft();
```

`Pilot` inherits the accessible `work()` behavior and adds
`flyAircraft()`.

### 6.3 Types of inheritance

Common conceptual forms:

**Single inheritance**

``` text
Animal
  |
  v
Dog
```

**Multilevel inheritance**

``` text
Animal
  |
  v
Mammal
  |
  v
Dog
```

**Hierarchical inheritance**

``` text
       Animal
       /    \
      v      v
     Dog    Cat
```

Java classes support single inheritance of classes. A class cannot
extend two classes.

### 6.4 Does Java support multiple inheritance?

Java does not allow a class to extend multiple classes:

``` java
// Not valid Java
class C extends A, B { }
```

Java does support implementing multiple interfaces:

``` java
interface Flyable {
    void fly();
}

interface Trackable {
    void track();
}

public class Aircraft implements Flyable, Trackable {
    @Override
    public void fly() {
        System.out.println("Aircraft flying");
    }

    @Override
    public void track() {
        System.out.println("Aircraft tracked");
    }
}
```

This allows a class to implement multiple capabilities without
inheriting state and implementation from multiple classes.

### 6.5 Benefits and risks of inheritance

Benefits: - Reuse common behavior. - Express a genuine subtype
relationship. - Support polymorphism. - Share a common contract or base
implementation.

Risks: - Deep hierarchies become difficult to understand. - Changes in a
superclass can affect subclasses. - A child may inherit behavior it
cannot safely support. - Inheritance can create tight coupling.

Use inheritance when the subtype relationship is valid. Do not use it
only to avoid writing a few lines of code.

------------------------------------------------------------------------

## 7. Polymorphism

### 7.1 What is polymorphism?

Polymorphism means "many forms." A common parent type or interface can
refer to objects of different concrete classes, and the appropriate
implementation can be selected at runtime.

### 7.2 Runtime polymorphism (method overriding)

``` java
public interface NotificationSender {
    void send(String message);
}
```

``` java
public class EmailSender implements NotificationSender {
    @Override
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}
```

``` java
public class SmsSender implements NotificationSender {
    @Override
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}
```

Client:

``` java
public class NotificationService {
    public void notify(
            NotificationSender sender,
            String message) {
        sender.send(message);
    }
}
```

Usage:

``` java
NotificationService service = new NotificationService();

NotificationSender sender = new EmailSender();
service.notify(sender, "Flight delayed");

sender = new SmsSender();
service.notify(sender, "Flight delayed");
```

The variable type is `NotificationSender`, but the runtime object
determines which overridden `send()` implementation executes.

This is runtime polymorphism through dynamic method dispatch.

### 7.3 Compile-time polymorphism (method overloading)

Method overloading means defining multiple methods with the same name
but different parameter lists.

``` java
public class FareCalculator {

    public double calculateFare(double baseFare) {
        return baseFare;
    }

    public double calculateFare(
            double baseFare,
            double tax) {
        return baseFare + tax;
    }

    public double calculateFare(
            double baseFare,
            double tax,
            double discount) {
        return baseFare + tax - discount;
    }
}
```

The compiler selects the matching overload based on the method arguments
and compile-time types.

**Important:** Changing only the return type does not create a valid
overload.

### 7.4 Overloading vs overriding

  -----------------------------------------------------------------------
  Overloading                         Overriding
  ----------------------------------- -----------------------------------
  Same method name, different         Subclass provides a new
  parameter list                      implementation of an inherited
                                      instance method

  Usually within the same class, but  Requires inheritance or interface
  can occur across inherited methods  implementation

  Resolved at compile time            Dynamic dispatch for instance
                                      methods at runtime

  Return type alone cannot            Return type must be same or
  distinguish overloads               covariant

  `@Override` is not used             `@Override` is recommended
  -----------------------------------------------------------------------

### 7.5 Important Java overriding rules

-   The overriding method cannot reduce visibility.
-   A checked exception cannot be broadened beyond what the overridden
    method allows.
-   Static methods are hidden, not overridden.
-   Private methods are not overridden.
-   Final methods cannot be overridden.
-   Covariant return types are permitted.
-   Constructors are not inherited or overridden.

------------------------------------------------------------------------

# Part 2 --- Other Important Object-Oriented Relationships and Java Concepts

## 8. Association, Aggregation, and Composition

These concepts describe relationships between objects.

### 8.1 Association

Association means two objects are connected or interact with each other.

Example: a passenger books a flight.

``` java
public class Passenger {
    private String name;

    public Passenger(String name) {
        this.name = name;
    }
}

public class FlightBooking {
    private Passenger passenger;

    public FlightBooking(Passenger passenger) {
        this.passenger = passenger;
    }
}
```

The booking refers to a passenger. Association by itself does not define
ownership or lifecycle.

### 8.2 Aggregation

Aggregation is a weaker whole-part relationship. The contained object
can exist independently of the container.

Example: a department has employees, but employees can exist
independently of a particular department.

``` java
import java.util.List;

public class Department {
    private final List<Employee> employees;

    public Department(List<Employee> employees) {
        this.employees = employees;
    }
}
```

The department receives employee objects created elsewhere.

### 8.3 Composition

Composition is a stronger whole-part relationship. The containing object
owns the lifecycle of its parts as part of its model.

Example: an order owns its order-line items.

``` java
import java.util.ArrayList;
import java.util.List;

public class Order {
    private final List<OrderLine> lines = new ArrayList<>();

    public void addLine(String product, int quantity) {
        lines.add(new OrderLine(product, quantity));
    }

    private static class OrderLine {
        private final String product;
        private final int quantity;

        private OrderLine(String product, int quantity) {
            this.product = product;
            this.quantity = quantity;
        }
    }
}
```

The `Order` controls creation and access to its lines in this simplified
example.

### 8.4 Comparison

  -----------------------------------------------------------------------
  Relationship            Meaning                 Lifecycle/ownership
  ----------------------- ----------------------- -----------------------
  Association             Objects know or use     No ownership implied
                          each other              

  Aggregation             Weak whole-part         Parts can exist
                          relationship            independently

  Composition             Strong whole-part       Whole controls the
                          relationship            parts' lifecycle in the
                                                  model
  -----------------------------------------------------------------------

These are modeling concepts. Java does not have special `aggregation` or
`composition` keywords; they are expressed through object references and
lifecycle design.

------------------------------------------------------------------------

## 9. Constructor

A constructor initializes a new object.

``` java
public class Passenger {
    private final String name;

    public Passenger(String name) {
        this.name = name;
    }
}
```

Key points: - Constructor name must match the class name. - A
constructor has no return type, not even `void`. - Constructors can be
overloaded. - Constructors are not inherited. - If no constructor is
declared, Java provides a default no-argument constructor. - If any
constructor is declared, Java does not automatically provide that
default constructor.

## 10. `this` and `super`

### `this`

`this` refers to the current object.

``` java
public class Passenger {
    private String name;

    public Passenger(String name) {
        this.name = name;
    }
}
```

### `super`

`super` refers to the superclass portion of the current object and can
be used to call superclass constructors or accessible superclass
methods.

``` java
public class Person {
    public Person(String name) {
        System.out.println("Person: " + name);
    }
}

public class Passenger extends Person {
    public Passenger(String name) {
        super(name);
    }
}
```

A superclass constructor call must be the first statement in a
constructor when explicitly written using `super(...)` in traditional
Java constructor syntax.

## 11. `static`

A static field or method belongs to the class rather than to an
individual instance.

``` java
public class Flight {
    private static int totalFlights;

    public Flight() {
        totalFlights++;
    }

    public static int getTotalFlights() {
        return totalFlights;
    }
}
```

Use static for class-level state or behavior that does not require a
particular object. Be careful with mutable static state because it is
shared and can complicate concurrency and testing.

## 12. `final`

`final` has different meanings depending on where it is used.

``` java
final class SecurityPolicy {
    // Cannot be subclassed
}
```

``` java
public class Booking {
    private final String bookingId;

    public Booking(String bookingId) {
        this.bookingId = bookingId;
    }
}
```

-   Final class: cannot be extended.
-   Final method: cannot be overridden.
-   Final variable: can be assigned only once.
-   A final object reference cannot point to a different object, but the
    referenced object may still be mutable.

## 13. Access modifiers

  -------------------------------------------------------------------------------------
  Modifier              Same class   Same package    Subclass in         Other packages
                                                         another 
                                                         package 
  ----------------- -------------- -------------- -------------- ----------------------
  `private`                    Yes             No             No                     No

  package-private              Yes            Yes     No, except                     No
  (no modifier)                                          through 
                                                  package access 
                                                           rules 

  `protected`                  Yes            Yes   Yes, subject      No general access
                                                    to protected 
                                                    access rules 

  `public`                     Yes            Yes            Yes        Yes, subject to
                                                                   module/accessibility
                                                                                  rules
  -------------------------------------------------------------------------------------

Use the least permissive visibility that supports the required design.

------------------------------------------------------------------------

# Part 3 --- SOLID Principles

## 14. What is SOLID?

SOLID is a group of five object-oriented design principles that help
developers build software that is easier to understand, extend, test,
and maintain.

  Letter   Principle
  -------- ---------------------------------
  S        Single Responsibility Principle
  O        Open/Closed Principle
  L        Liskov Substitution Principle
  I        Interface Segregation Principle
  D        Dependency Inversion Principle

SOLID principles are design guidelines, not rigid laws. Apply them to
reduce real design problems rather than adding abstractions without
purpose.

------------------------------------------------------------------------

## 15. S --- Single Responsibility Principle (SRP)

### Definition

A class should have one responsibility and one reason to change.

A practical interpretation is that a class should be focused on one
cohesive area of responsibility.

### Bad design

``` java
public class FlightBookingService {

    public void bookFlight() {
        // Booking business logic
    }

    public void generateTicketPdf() {
        // PDF generation
    }

    public void sendEmail() {
        // Email delivery
    }

    public void saveToDatabase() {
        // Persistence
    }
}
```

This class handles booking, document generation, notification, and
persistence. Different business or technical changes may force changes
to the same class.

### Better design

``` java
public class FlightBookingService {
    public void bookFlight() {
        // Booking business logic
    }
}
```

``` java
public class TicketGenerator {
    public void generatePdf() {
        // PDF generation
    }
}
```

``` java
public class EmailNotificationService {
    public void sendEmail() {
        // Email delivery
    }
}
```

``` java
public class BookingRepository {
    public void save() {
        // Persistence
    }
}
```

### Benefits

-   Easier maintenance.
-   Easier unit testing.
-   Clearer responsibilities.
-   Reduced impact of changes.
-   Better separation of business logic and infrastructure.

### Interview answer

SRP states that a class should have one responsibility and one reason to
change. For example, a booking service should handle booking rules,
while PDF generation, notification, and persistence should be handled by
focused collaborators.

------------------------------------------------------------------------

## 16. O --- Open/Closed Principle (OCP)

### Definition

Software entities such as classes, modules, and methods should be **open
for extension but closed for modification**.

This means we should be able to add new behavior without repeatedly
changing stable, existing code.

### Bad design

``` java
public class NotificationService {

    public void send(String type, String message) {
        if (type.equals("EMAIL")) {
            System.out.println("Sending email: " + message);
        } else if (type.equals("SMS")) {
            System.out.println("Sending SMS: " + message);
        } else if (type.equals("PUSH")) {
            System.out.println("Sending push notification: " + message);
        }
    }
}
```

Every new notification type requires changing the existing method. The
conditional logic grows over time.

### Better design using polymorphism

``` java
public interface NotificationSender {
    void send(String message);
}
```

``` java
public class EmailSender implements NotificationSender {
    @Override
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}
```

``` java
public class SmsSender implements NotificationSender {
    @Override
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}
```

``` java
public class PushSender implements NotificationSender {
    @Override
    public void send(String message) {
        System.out.println("Push: " + message);
    }
}
```

``` java
public class NotificationService {
    public void notify(
            NotificationSender sender,
            String message) {
        sender.send(message);
    }
}
```

To add WhatsApp notifications, create another implementation:

``` java
public class WhatsAppSender implements NotificationSender {
    @Override
    public void send(String message) {
        System.out.println("WhatsApp: " + message);
    }
}
```

The core `NotificationService` does not need to change.

### Benefits

-   New functionality can be added with less risk.
-   Stable code changes less often.
-   Polymorphism replaces growing conditional logic.
-   Implementations can be tested independently.

### Interview answer

OCP means a component should be open for extension but closed for
modification. We can use interfaces and polymorphism so that new
behavior is introduced through new implementations rather than
repeatedly modifying stable business logic.

------------------------------------------------------------------------

## 17. L --- Liskov Substitution Principle (LSP)

### Definition

Objects of a subtype should be usable wherever objects of the base type
are expected without breaking the correctness of the program.

A subtype must honor the behavioral expectations of its parent type or
interface.

### Bad design example

``` java
public class Bird {
    public void fly() {
        System.out.println("Bird is flying");
    }
}
```

``` java
public class Sparrow extends Bird {
    @Override
    public void fly() {
        System.out.println("Sparrow is flying");
    }
}
```

``` java
public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException(
                "Penguins cannot fly");
    }
}
```

A caller expecting every `Bird` to fly can fail when given a `Penguin`.

The base abstraction promises more than every subtype can provide.

### Better design

Separate the general bird concept from the flying capability:

``` java
public abstract class Bird {
    public abstract void eat();
}
```

``` java
public interface Flyable {
    void fly();
}
```

``` java
public class Sparrow extends Bird implements Flyable {
    @Override
    public void eat() {
        System.out.println("Sparrow eats");
    }

    @Override
    public void fly() {
        System.out.println("Sparrow flies");
    }
}
```

``` java
public class Penguin extends Bird {
    @Override
    public void eat() {
        System.out.println("Penguin eats");
    }
}
```

Now code that requires flying accepts `Flyable`, not every `Bird`.

### Behavioral rules to remember

A well-behaved subtype should not: - Strengthen preconditions in a way
that rejects inputs accepted by the base contract. - Weaken
postconditions or fail to deliver promised results. - Break invariants
expected by callers. - Introduce unexpected failures for valid base-type
operations.

### Benefits

-   Reliable polymorphism.
-   Safer inheritance.
-   Fewer unexpected runtime errors.
-   Better-defined abstractions.

### Interview answer

LSP says a subtype must be substitutable for its base type without
changing the correctness expected by clients. If a subclass overrides a
method by throwing an unsupported-operation exception for behavior
promised by the parent, the hierarchy may violate LSP.

------------------------------------------------------------------------

## 18. I --- Interface Segregation Principle (ISP)

### Definition

Clients should not be forced to depend on methods they do not use.

Prefer several small, focused interfaces over one large interface
containing unrelated operations.

### Bad design

``` java
public interface Worker {
    void work();
    void eat();
    void attendMeeting();
}
```

A robot worker does not eat, but it is forced to implement the method:

``` java
public class RobotWorker implements Worker {

    @Override
    public void work() {
        System.out.println("Robot working");
    }

    @Override
    public void eat() {
        throw new UnsupportedOperationException();
    }

    @Override
    public void attendMeeting() {
        System.out.println("Robot attending meeting");
    }
}
```

This is a sign that the interface combines unrelated capabilities.

### Better design

``` java
public interface Workable {
    void work();
}
```

``` java
public interface Eatable {
    void eat();
}
```

``` java
public interface MeetingAttendee {
    void attendMeeting();
}
```

Human worker:

``` java
public class HumanWorker
        implements Workable, Eatable, MeetingAttendee {

    public void work() {
        System.out.println("Human working");
    }

    public void eat() {
        System.out.println("Human eating");
    }

    public void attendMeeting() {
        System.out.println("Human attending meeting");
    }
}
```

Robot worker:

``` java
public class RobotWorker implements Workable {
    public void work() {
        System.out.println("Robot working");
    }
}
```

### Benefits

-   Small and focused contracts.
-   Fewer unnecessary methods.
-   Less forced use of unsupported operations.
-   Easier implementation and testing.
-   Better separation of capabilities.

### Interview answer

ISP says a client should not be forced to depend on methods it does not
use. Split large interfaces into smaller capability-based interfaces so
each implementation only depends on the operations it actually needs.

------------------------------------------------------------------------

## 19. D --- Dependency Inversion Principle (DIP)

### Definition

1.  High-level modules should not depend directly on low-level modules.
    Both should depend on abstractions.
2.  Abstractions should not depend on details. Details should depend on
    abstractions.

### Bad design

``` java
public class MySqlBookingRepository {
    public void save() {
        System.out.println("Saving booking to MySQL");
    }
}
```

``` java
public class BookingService {
    private final MySqlBookingRepository repository =
            new MySqlBookingRepository();

    public void book() {
        repository.save();
    }
}
```

The high-level business service directly depends on a specific database
implementation.

### Better design

``` java
public interface BookingRepository {
    void save();
}
```

``` java
public class MySqlBookingRepository implements BookingRepository {
    @Override
    public void save() {
        System.out.println("Saving booking to MySQL");
    }
}
```

``` java
public class BookingService {
    private final BookingRepository repository;

    public BookingService(BookingRepository repository) {
        this.repository = repository;
    }

    public void book() {
        repository.save();
    }
}
```

Now `BookingService` depends on an abstraction. A PostgreSQL or
in-memory implementation can be supplied without changing the service.

### DIP and Dependency Injection are different

-   DIP is a design principle.
-   DI is a technique for supplying dependencies.
-   DI can help wire an application that follows DIP.
-   DI by itself does not guarantee DIP: injecting a concrete class
    still leaves the client coupled to that class.

### Benefits

-   Reduced coupling.
-   Easier testing.
-   Easier replacement of infrastructure.
-   Business logic can remain independent of database or messaging
    details.

### Interview answer

DIP says high-level and low-level modules should depend on abstractions
rather than concrete details. For example, a booking service should
depend on a repository interface rather than a concrete MySQL
repository. Spring constructor injection can supply the chosen
implementation.

------------------------------------------------------------------------

# Part 4 --- How OOP and SOLID Work Together

## 20. Mapping OOP concepts to SOLID

  -----------------------------------------------------------------------
  OOP concept                         How it supports SOLID
  ----------------------------------- -----------------------------------
  Encapsulation                       Protects invariants and helps keep
                                      responsibilities cohesive (SRP).

  Abstraction                         Supports OCP, ISP, and DIP through
                                      meaningful contracts.

  Inheritance                         Can support polymorphism, but must
                                      respect LSP.

  Polymorphism                        Helps extend behavior without
                                      changing stable client code (OCP).

  Composition                         Often provides flexible
                                      alternatives to deep inheritance.
  -----------------------------------------------------------------------

OOP gives us language features and modeling techniques. SOLID gives us
guidelines for using those features to build maintainable designs.

## 21. Composition over inheritance

Composition means building a class by giving it collaborators that
provide behavior, rather than inheriting all behavior from a parent
class.

Example:

``` java
public interface PaymentProcessor {
    void process(double amount);
}
```

``` java
public class CardPaymentProcessor implements PaymentProcessor {
    @Override
    public void process(double amount) {
        System.out.println("Processing card payment: " + amount);
    }
}
```

``` java
public class CheckoutService {
    private final PaymentProcessor paymentProcessor;

    public CheckoutService(PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }

    public void checkout(double amount) {
        paymentProcessor.process(amount);
    }
}
```

`CheckoutService` has a `PaymentProcessor`. It does not need to extend a
payment processor.

Advantages: - Behavior can be replaced through a collaborator. - Avoids
deep inheritance hierarchies. - Makes dependencies explicit. - Supports
testing with fakes and mocks.

Inheritance remains appropriate for genuine subtype relationships.
Composition is often preferable when the relationship is "uses" or "has
a capability."

------------------------------------------------------------------------

# Part 5 --- Common Interview Questions

## 22. What are the four pillars of OOP?

**Answer:** Encapsulation, Abstraction, Inheritance, and Polymorphism.

## 23. What is the difference between abstraction and encapsulation?

**Answer:** Abstraction focuses on exposing essential behavior and
hiding implementation complexity. Encapsulation focuses on bundling
state and behavior and controlling access to internal state.

## 24. What is the difference between overloading and overriding?

**Answer:** Overloading uses the same method name with different
parameter lists and is resolved at compile time. Overriding provides a
subtype implementation of an inherited instance method and participates
in runtime dynamic dispatch.

## 25. Does Java support multiple inheritance?

**Answer:** Java does not support a class extending multiple classes. A
class can implement multiple interfaces.

## 26. What is the difference between an interface and an abstract class?

**Answer:** An interface primarily defines a contract or capability and
allows a class to implement multiple interfaces. An abstract class can
provide shared state, constructors, and partial implementation, but a
class can extend only one class.

## 27. What is composition?

**Answer:** Composition is a strong whole-part relationship in which the
containing object controls the lifecycle of its parts in the model. In
implementation, the owner commonly creates and manages those parts.

## 28. What are the SOLID principles?

**Answer:** - SRP: One responsibility and one reason to change. - OCP:
Open for extension, closed for modification. - LSP: Subtypes must be
substitutable for their base types. - ISP: Clients should not depend on
methods they do not use. - DIP: High-level and low-level modules should
depend on abstractions.

## 29. Which SOLID principle is related to interfaces?

**Answer:** ISP encourages small, focused interfaces. DIP also uses
abstractions such as interfaces so high-level modules do not depend on
concrete implementation details.

## 30. Which SOLID principle is related to polymorphism?

**Answer:** Polymorphism is commonly used to implement OCP by allowing
new implementations to be added without changing stable client code. It
must also respect LSP so each subtype behaves according to its contract.

## 31. Is SOLID mandatory in every class?

**Answer:** No. SOLID is a set of design guidelines. Apply the
principles where they solve real coupling, maintainability,
substitutability, or testing problems. Overengineering every small class
with extra interfaces and layers can make a system harder to understand.

------------------------------------------------------------------------

# Part 6 --- Quick Revision Cheat Sheet

## 32. OOP quick revision

  -----------------------------------------------------------------------
  Concept                             Remember it as
  ----------------------------------- -----------------------------------
  Class                               Blueprint defining fields and
                                      behavior

  Object                              Runtime instance of a class

  Encapsulation                       Protect internal state and expose
                                      controlled operations

  Abstraction                         Expose what an object does; hide
                                      unnecessary implementation details

  Inheritance                         Reuse/specialize behavior through
                                      an is-a relationship

  Polymorphism                        One common type, multiple
                                      implementations

  Overloading                         Same method name, different
                                      parameter lists

  Overriding                          Subtype supplies an implementation
                                      of an inherited instance method

  Association                         Objects are connected

  Aggregation                         Weak whole-part relationship

  Composition                         Strong whole-part ownership
                                      relationship

  Constructor                         Initializes a new object

  `this`                              Current object reference

  `super`                             Access to superclass constructor or
                                      members

  `static`                            Belongs to the class

  `final`                             Prevents reassignment, overriding,
                                      or subclassing depending on usage
  -----------------------------------------------------------------------

## 33. SOLID quick revision

  Principle   Remember it as
  ----------- ----------------------------------------------------------
  SRP         One responsibility; one reason to change
  OCP         Extend behavior without repeatedly modifying stable code
  LSP         A subtype must honor the base contract
  ISP         Prefer small, focused interfaces
  DIP         Depend on abstractions, not concrete details

## 34. Final learning sequence

For a Java backend developer, study and practise in this order:

1.  Classes, objects, constructors, and access modifiers.
2.  Encapsulation and abstraction.
3.  Inheritance and polymorphism.
4.  Overloading and overriding.
5.  Association, aggregation, and composition.
6.  Interfaces and abstract classes.
7.  Composition over inheritance.
8.  SRP and OCP.
9.  LSP and ISP.
10. DIP, Dependency Injection, and Spring IoC.

**Final takeaway:** OOP helps you model software using objects and
relationships. SOLID helps you structure those objects so that the code
remains understandable, testable, extensible, and maintainable as the
application grows.

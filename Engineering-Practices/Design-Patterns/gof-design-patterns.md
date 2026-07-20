# GoF Design Patterns

The 23 classic object-oriented design patterns catalogued by the **"Gang of Four"** — Erich Gamma, Richard Helm,
Ralph Johnson, John Vlissides — in the 1994 book *"Design Patterns: Elements of Reusable Object-Oriented Software"*.
The book established a common vocabulary for reusable OO design and grouped the patterns into three families:
**Creational**, **Structural**, and **Behavioral**.

A staple of every OOP interview — interviewers expect you to name the categories, explain each pattern, recognise it
in code, and discuss trade-offs. Modern languages (Java 8+ lambdas, records, sealed interfaces) have shrunk the
boilerplate of many patterns, but the intent remains identical.

---

## Quick Reference

### Creational — how objects are constructed

| # | Pattern           | Intent in one line                                              |
|---|-------------------|------------------------------------------------------------------|
| 1 | **Singleton**     | Ensure a class has exactly one instance + global access point    |
| 2 | **Factory Method**| Define an interface for creating objects, defer creation to subs |
| 3 | **Abstract Factory** | Create families of related objects without specifying classes |
| 4 | **Builder**       | Separate construction of a complex object from its representation|
| 5 | **Prototype**     | Create new objects by cloning an existing instance               |

### Structural — how classes/objects are composed

| #  | Pattern        | Intent in one line                                                  |
|----|----------------|---------------------------------------------------------------------|
| 6  | **Adapter**    | Make incompatible interfaces work together                          |
| 7  | **Bridge**     | Decouple abstraction from implementation so they vary independently |
| 8  | **Composite**  | Treat individual objects and compositions uniformly                 |
| 9  | **Decorator**  | Add behaviour dynamically without subclassing                       |
| 10 | **Facade**     | Provide a simplified interface to a complex subsystem               |
| 11 | **Flyweight**  | Share fine-grained objects to reduce memory                         |
| 12 | **Proxy**      | Provide a placeholder controlling access to another object          |

### Behavioral — how objects communicate and distribute responsibility

| #  | Pattern                       | Intent in one line                                              |
|----|-------------------------------|------------------------------------------------------------------|
| 13 | **Chain of Responsibility**   | Pass a request along a chain until a handler handles it          |
| 14 | **Command**                   | Encapsulate a request as an object                               |
| 15 | **Interpreter**               | Define a grammar + interpreter for a language                    |
| 16 | **Iterator**                  | Access elements of a collection sequentially without exposing it |
| 17 | **Mediator**                  | Centralise complex communication between objects                 |
| 18 | **Memento**                   | Capture + restore an object's internal state                     |
| 19 | **Observer**                  | Notify dependents automatically when state changes               |
| 20 | **State**                     | Alter behaviour when the object's state changes                  |
| 21 | **Strategy**                  | Encapsulate interchangeable algorithms behind a common interface |
| 22 | **Template Method**           | Define the skeleton of an algorithm, defer steps to subclasses   |
| 23 | **Visitor**                   | Add operations to an object structure without modifying it       |

---

## Two Pattern Flavours

- **Class patterns** — use **inheritance** to compose structures statically (Factory Method, Adapter (class version),
  Interpreter, Template Method).
- **Object patterns** — use **composition and delegation** to compose structures dynamically. The vast majority of GoF
  patterns belong here. Modern OO design **strongly favours composition over inheritance**, so prefer the object form.

> Two recurring principles underlie almost every pattern: **Program to an interface, not an implementation** and
> **Favour object composition over class inheritance**.

---

# Creational Patterns

## 1. Singleton

> Ensure a class has **exactly one instance** and provide a global access point to it.

### Classic Java implementation

```java
public class Configuration {
    private static volatile Configuration INSTANCE;

    private Configuration() { /* read config files */ }

    public static Configuration getInstance() {
        if (INSTANCE == null) {
            synchronized (Configuration.class) {
                if (INSTANCE == null) INSTANCE = new Configuration();
            }
        }
        return INSTANCE;
    }
}
```

This is **double-checked locking** — the `volatile` keyword prevents a partially-constructed object from being
published. Without it, another thread can see `INSTANCE != null` but read uninitialised fields.

### Modern alternatives

```java
// Enum singleton — preferred (Effective Java item 3)
public enum Configuration {
    INSTANCE;
    public String get(String key) { /* ... */ }
}

// Or a dependency-injected bean (Spring default scope "singleton")
@Component public class Configuration { /* ... */ }
```

### Pitfalls

- A **global mutable** singleton is just a global variable; it couples everything that touches it.
- Hard to unit test — you cannot easily inject a fake.
- Spring `@Component` / `@Service` beans are singletons by design but injected via DI, which solves the coupling
  problem.

### Real-world examples

- `java.lang.Runtime#getRuntime()` — one runtime per JVM.
- Spring beans (singleton scope).
- `LoggerFactory` in SLF4J.

---

## 2. Factory Method

> Define an interface for creating an object, but let **subclasses decide** which class to instantiate.

```java
interface Transport { void deliver(String item); }

abstract class Logistics {
    public void planDelivery(String item) {
        Transport t = createTransport();   // Factory method
        t.deliver(item);
    }
    protected abstract Transport createTransport();   // subclass decides
}

class RoadLogistics  extends Logistics { protected Transport createTransport() { return new Truck(); } }
class SeaLogistics   extends Logistics { protected Transport createTransport() { return new Ship();  } }
```

The base class (`Logistics`) is **decoupled from the concrete `Transport`**: it works against the abstract type and
lets subclasses supply the implementation.

### Java 8+ variant

```java
interface TransportFactory { Transport create(); }
Logistics road = new Logistics(TransportFactory.of(Truck::new));
```

A `Supplier<T>` or `Function` often replaces the abstract-class factory in modern code.

### Real-world examples

- `java.util.List#iterator()` — every `List` implementation returns its own `Iterator`.
- `Calendar#getInstance()`, `NumberFormat#getInstance()`.
- Spring's `FactoryBean<T>`.

---

## 3. Abstract Factory

> Create **families of related objects** without specifying their concrete classes.

```java
interface UIFactory {
    Button createButton();
    TextField createTextField();
}

class MacUIFactory   implements UIFactory { /* returns MacButton, MacTextField   */ }
class WindowsUIFactory implements UIFactory { /* returns WinButton, WinTextField  */ }

class Application {
    private final UIFactory factory;
    Application(UIFactory factory) { this.factory = factory; }
    void render() {
        factory.createButton().render();
        factory.createTextField().render();
    }
}
```

**Factory Method** produces one product; **Abstract Factory** produces a *family* of products that must be consistent
(Mac button + Mac text field, never mixed).

### Real-world examples

- `javax.xml.parsers.DocumentBuilderFactory`.
- Cross-platform UI toolkits (Mac vs Windows controls).
- Database-specific connection factories (`MySQLFactory`, `PostgresFactory` producing `Connection`, `Statement`).

---

## 4. Builder

> Separate construction of a complex object from its representation so the same build process can create different
> representations.

Solves the **telescoping constructor** anti-pattern:

```java
// Anti-pattern
new Pizza("M", true, true, false, false, true, false, true);
```

### Fluent builder

```java
public final class Pizza {
    private final String size;
    private final boolean cheese, pepperoni, mushroom, bacon;

    private Pizza(Builder b) {
        this.size = b.size; this.cheese = b.cheese; this.pepperoni = b.pepperoni;
        this.mushroom = b.mushroom; this.bacon = b.bacon;
    }
    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String size;
        private boolean cheese, pepperoni, mushroom, bacon;

        public Builder size(String s)              { this.size = s; return this; }
        public Builder cheese(boolean v)           { this.cheese = v; return this; }
        public Builder pepperoni(boolean v)        { this.pepperoni = v; return this; }
        public Builder addMushroom()               { this.mushroom = true; return this; }
        public Pizza build() {
            if (size == null) throw new IllegalStateException("size required");
            return new Pizza(this);
        }
    }
}

Pizza p = Pizza.builder().size("L").cheese(true).addMushroom().build();
```

### Modern alternative: records + compact validation

```java
public record Pizza(String size, boolean cheese, boolean pepperoni) {
    public Pizza {
        if (size == null) throw new IllegalArgumentException("size required");
    }
}
// For many optional fields, a builder still wins; for few, records are cleaner.
```

### Real-world examples

- `StringBuilder` (builds a `String` incrementally).
- `Stream.Builder`, `HttpRequest.Builder` (Java 11+ HTTP client).
- Lombok `@Builder`.

---

## 5. Prototype

> Create new objects by **cloning** a pre-built instance.

Useful when construction is expensive (DB lookup, deep computation) but copying is cheap.

```java
public abstract class Shape implements Cloneable {
    protected int x, y;
    public abstract Shape clone();      // shallow copy usually; deep copy if needed
}

public class Circle extends Shape {
    private int radius;
    @Override public Circle clone() {
        Circle c = new Circle();
        c.x = x; c.y = y; c.radius = radius;
        return c;
    }
}

Shape prototype = new Circle(/* configured */);
Shape copy = prototype.clone();
```

### Pitfalls

- Java's `clone()` is widely considered **broken** (no constructor call, shallow by default, clumsy `Cloneable`
  marker). Effective Java recommends copy constructors or `Copy` factories instead.
- Deep vs shallow copies are an easy interview trap.

### Real-world examples

- `ArrayList#clone()` (shallow).
- Prototyping game objects, configuration templates.

---

# Structural Patterns

## 6. Adapter

> Convert the interface of a class into another interface clients expect.

```
   Client ──expects──▶ Target interface
                              │
                       Adapter ◀── delegates to ── Adaptee (incompatible interface)
```

```java
// Target — what the client wants
interface Logger { void log(String message); }

// Adaptee — third-party class with a different API
class ThirdPartyLogger { void writeLine(String category, String text) { /* ... */ } }

// Adapter
class ThirdPartyLoggerAdapter implements Logger {
    private final ThirdPartyLogger adaptee;
    ThirdPartyLoggerAdapter(ThirdPartyLogger l) { this.adaptee = l; }
    public void log(String msg) { adaptee.writeLine("APP", msg); }
}
```

Two forms:
- **Class adapter** — via multiple inheritance (Java disallows; achievable with interfaces + extending adaptee).
- **Object adapter** — via composition (preferred).

### Real-world examples

- `java.util.Arrays#asList()` — adapts an array to the `List` interface.
- `InputStreamReader` adapts a byte `InputStream` to a character `Reader`.
- SLF4J bridging modules for JUL/log4j.

---

## 7. Bridge

> **Decouple abstraction from implementation** so they can vary independently.

Solves the **exploding class hierarchy** problem:

```
   Without Bridge:                 With Bridge:
   Shape                            Shape ──▶ Renderer (interface)
   ├── Circle                          ├── Circle
   ├── Square                          └── Square
   ├── FilledCircle                  Renderer
   ├── FilledSquare                  ├── VectorRenderer
   ├── OutlinedCircle                └── RasterRenderer
   └── OutlinedSquare                (2 shapes × 2 renderers = 4 combinations
   (n shapes × m styles =            instead of n*m classes)
    n*m classes)
```

```java
interface Renderer { void renderCircle(double radius); }

class VectorRenderer  implements Renderer { public void renderCircle(double r) { /* vector */ } }
class RasterRenderer  implements Renderer { public void renderCircle(double r) { /* raster */ } }

abstract class Shape {
    protected final Renderer renderer;
    Shape(Renderer r) { this.renderer = r; }
    public abstract void draw();
}

class Circle extends Shape {
    private final double radius;
    Circle(Renderer r, double radius) { super(r); this.radius = radius; }
    public void draw() { renderer.renderCircle(radius); }
}
```

### Real-world examples

- JDBC: the abstraction (`java.sql.Connection`) is decoupled from each driver's implementation.
- PIMPL ("Pointer to IMPL") idiom in C++.

---

## 8. Composite

> Compose objects into **tree structures** to represent part-whole hierarchies; let clients treat individual objects
> and compositions uniformly.

```java
interface FileSystemNode { long size(); }

class File implements FileSystemNode {
    private final long size;
    File(long size) { this.size = size; }
    public long size() { return size; }
}

class Directory implements FileSystemNode {
    private final List<FileSystemNode> children = new ArrayList<>();
    public void add(FileSystemNode n) { children.add(n); }
    public long size() {
        return children.stream().mapToLong(FileSystemNode::size).sum();
    }
}
```

A `Directory` and a `File` share the same interface; clients treat them identically. This is the basis of every
recursive tree operation: UI containers, org charts, file systems, ASTs.

### Real-world examples

- `java.awt.Container` / `Component`.
- DOM / React element trees.
- `Map<String, Object>` JSON structures.

---

## 9. Decorator

> Attach additional responsibility to an object **dynamically** without subclassing.

```java
interface Coffee { double cost(); }

class SimpleCoffee implements Coffee { public double cost() { return 2.0; } }

abstract class CoffeeDecorator implements Coffee {
    protected final Coffee inner;
    CoffeeDecorator(Coffee c) { this.inner = c; }
}

class Milk  extends CoffeeDecorator { Milk(Coffee c)  { super(c); } public double cost() { return inner.cost() + 0.5; } }
class Sugar extends CoffeeDecorator { Sugar(Coffee c) { super(c); } public double cost() { return inner.cost() + 0.2; } }

Coffee c = new Sugar(new Milk(new SimpleCoffee()));   // 2.7
```

Decorators wrap the inner object, delegate calls, and add behaviour. The client sees the same interface.

### Java 8+ alternative: function composition

```java
Coffee c = Stream.<Function<Coffee,Coffee>>of(Milk::new, Sugar::new)
                .reduce(Function.identity(), Function::andThen)
                .apply(new SimpleCoffee());
```

### Real-world examples

- `java.io` — `new BufferedInputStream(new FileInputStream("f"))`. The whole `java.io` package is decorators all the
  way down.
- `Collections.unmodifiableList`, `synchronizedList`.
- Spring `HandlerInterceptor`, AOP proxies.

---

## 10. Facade

> Provide a **unified interface** to a set of interfaces in a subsystem; makes the subsystem easier to use.

```java
class CPU        { void freeze() {} void jump(long addr) {} void execute() {} }
class Memory     { void load(long addr, byte[] data) {} }
class HardDrive  { byte[] read(long lba, int size) { return new byte[size]; } }

// Facade — clients only see this
class Computer {
    private static final long BOOT_ADDRESS = 0x0000_0000L;
    private static final long BOOT_SECTOR  = 0L;
    private static final int  SECTOR_SIZE  = 512;

    private final CPU cpu; private final Memory memory; private final HardDrive hd;

    Computer() { this.cpu = new CPU(); this.memory = new Memory(); this.hd = new HardDrive(); }

    public void start() {
        cpu.freeze();
        memory.load(BOOT_ADDRESS, hd.read(BOOT_SECTOR, SECTOR_SIZE));
        cpu.jump(BOOT_ADDRESS);
        cpu.execute();
    }
}
```

A Facade does **not encapsulate** the subsystem — clients can still reach inner classes if needed. It only provides a
shortcut.

### Real-world examples

- Spring's `JdbcTemplate` (hides `Connection`/`Statement`/`ResultSet` dance).
- SLF4J as a logging facade.
- A microservice "API gateway" is a facade at the architectural level.

---

## 11. Flyweight

> **Share** fine-grained objects to support large numbers efficiently.

Separates **intrinsic** state (shared, immutable) from **extrinsic** state (passed in at call time).

```java
public record TreeType(String name, Color color, String texture) {}   // intrinsic — cached

public class TreeFactory {
    private static final Map<String, TreeType> CACHE = new HashMap<>();
    public static TreeType get(String name, Color c, String t) {
        return CACHE.computeIfAbsent(name, k -> new TreeType(name, c, t));
    }
}

class Tree {                                   // each Tree instance is small
    private final int x, y;                    // extrinsic
    private final TreeType type;               // shared
    void draw(Canvas canvas) { /* use type + x + y */ }
}
```

A forest of 1,000,000 trees uses only ~5 shared `TreeType` objects plus lightweight `Tree` instances holding positions.

### Real-world examples

- `Integer.valueOf(int)` — caches values in `IntegerCache` (−128..127).
- `String` intern pool.
- `Boolean.TRUE` / `Boolean.FALSE`.
- Game engines sharing textures.

---

## 12. Proxy

> Provide a **placeholder** that controls access to another object.

Categories of proxy:
- **Remote** — represents an object in another address space (RMI, gRPC stubs).
- **Virtual** — creates expensive objects lazily.
- **Protection** — checks access permissions before forwarding.
- **Smart** — adds caching, logging, metrics, reference counting.

```java
interface Image { void draw(); }

class RealImage implements Image {
    private final String file;
    RealImage(String f) { this.file = f; loadFromDisk(); }   // expensive
    private void loadFromDisk() { /* read 50 MB */ }
    public void draw() { /* render */ }
}

class LazyImageProxy implements Image {
    private RealImage real;
    private final String file;
    LazyImageProxy(String f) { this.file = f; }
    public void draw() {
        if (real == null) real = new RealImage(file);   // lazy
        real.draw();
    }
}
```

### Real-world examples

- Spring AOP beans — every `@Transactional`/`@Async`/`@Cacheable` method call goes through a dynamic proxy.
- Hibernate lazy-loading proxies.
- JDK `Proxy.newProxyInstance` — generates proxy classes at runtime.
- `java.lang.reflect.Proxy`, gRPC client stubs.

---

# Behavioral Patterns

## 13. Chain of Responsibility

> Pass a request along a chain of handlers; each decides to handle it or pass it on.

```java
abstract class Handler {
    protected Handler next;
    Handler setNext(Handler n) { this.next = n; return n; }
    public abstract void handle(Request req);
}

class AuthHandler  extends Handler { public void handle(Request r) { if (authFails(r)) reject(); else if (next != null) next.handle(r); } }
class LogHandler   extends Handler { public void handle(Request r) { log(r); if (next != null) next.handle(r); } }
class RateLimiter  extends Handler { public void handle(Request r) { if (overLimit()) reject(); else if (next != null) next.handle(r); } }

Handler chain = new AuthHandler();
chain.setNext(new RateLimiter()).setNext(new LogHandler());
chain.handle(request);
```

### Modern Java variant — functional

```java
List<Predicate<Request>> chain = List.of(this::auth, this::rateLimit, this::log);
boolean ok = chain.stream().allMatch(p -> p.test(request));
```

### Real-world examples

- Servlet `Filter` chain.
- Spring `HandlerInterceptor` chain.
- `java.util.logging` handlers.
- Express middleware, ASP.NET pipeline.

---

## 14. Command

> Encapsulate a request as an object — letting you queue, log, undo, parameterise clients.

```java
interface Command { void execute(); void undo(); }

class Light {
    public void on()  { /* ... */ }
    public void off() { /* ... */ }
}

class LightOnCommand implements Command {
    private final Light light;
    LightOnCommand(Light l) { this.light = l; }
    public void execute() { light.on(); }
    public void undo()    { light.off(); }
}

class RemoteControl {
    private final Deque<Command> history = new ArrayDeque<>();
    public void submit(Command c) { c.execute(); history.push(c); }
    public void undo() { if (!history.isEmpty()) history.pop().undo(); }
}
```

A Command object turns a method call into a **value** — so it can be stored, queued, serialised, replayed.

### Java 8+ variant — `Runnable` / `Consumer`

```java
Map<String, Runnable> commands = Map.of(
    "LIGHT_ON",  () -> light.on(),
    "LIGHT_OFF", () -> light.off()
);
commands.get("LIGHT_ON").run();
```

When you don't need undo/history/serialisation, a lambda is enough.

### Real-world examples

- `Runnable`, `Callable`, `Consumer<T>` in the JDK.
- Spring Batch `Job`/`Step` model.
- Event-sourcing systems (each event is a command applied to a state).

---

## 15. Interpreter

> Given a language, define a representation for its grammar + an interpreter that uses it.

```bnf
Expression := Number | Expression "+" Expression | Expression "-" Expression
```

```java
interface Expr { int interpret(); }
record NumberExpr(int value)              implements Expr { public int interpret() { return value; } }
record PlusExpr(Expr a, Expr b)           implements Expr { public int interpret() { return a.interpret() + b.interpret(); } }
record MinusExpr(Expr a, Expr b)          implements Expr { public int interpret() { return a.interpret() - b.interpret(); } }

Expr e = new PlusExpr(new NumberExpr(5), new MinusExpr(new NumberExpr(10), new NumberExpr(3)));
e.interpret();   // 5 + (10 − 3) = 12
```

### Caveats

- The GoF version is **rarely practical** for non-trivial grammars — performance and complexity are bad.
- Modern alternatives: parser combinators, ANTLR, hand-written recursive-descent parsers.

### Real-world examples

- `java.util.regex.Pattern` (regex is a small language).
- SpEL (Spring Expression Language), JEXL.
- SQL, JSONPath, GraphQL parsers.

---

## 16. Iterator

> Provide a way to **access elements of a collection sequentially** without exposing its underlying representation.

```java
interface Iterator<T> { boolean hasNext(); T next(); }

class ArrayIterator<T> implements Iterator<T> {
    private final T[] items; private int pos = 0;
    ArrayIterator(T[] items) { this.items = items; }
    public boolean hasNext() { return pos < items.length; }
    public T next()          { return items[pos++]; }
}

for (String s : iterable) { /* ... */ }   // implicit via java.util.Iterator
```

### Real-world examples

- `java.util.Iterator` + the enhanced `for` loop (since Java 5).
- `Stream<T>`.
- `ResultSet` in JDBC, `Scanner`.

---

## 17. Mediator

> Define an object that **encapsulates how a set of objects interact**; promotes loose coupling by keeping objects
> from referring to each other explicitly.

Without a mediator, components form an N×N graph. With one, every component talks only to the mediator.

```java
class ChatRoom {                                   // Mediator
    public void sendMessage(User from, String msg) { /* broadcast */ }
}

class User {
    private final ChatRoom room;
    void send(String msg) { room.sendMessage(this, msg); }
}
```

### Real-world examples

- `java.util.Timer` (mediates timer tasks).
- Spring `ApplicationContext` (mediates beans).
- Message broker (Kafka/RabbitMQ) is a distributed Mediator.
- UI dialog controllers — buttons, text fields, and lists talk through the form controller instead of each other.

---

## 18. Memento

> Capture and externalise an object's internal state so it can be restored later — without violating encapsulation.

Three roles: **originator** (has the state), **memento** (immutable snapshot), **caretaker** (stores snapshots).

```java
class EditorMemento { private final String text; EditorMemento(String t) { this.text = t; } String text() { return text; } }

class Editor {                                       // Originator
    private String text = "";
    public void type(String w)   { text += w; }
    public EditorMemento save()  { return new EditorMemento(text); }
    public void restore(EditorMemento m) { text = m.text(); }
}

Editor e = new Editor();
e.type("Hello "); EditorMemento m1 = e.save();
e.type("World");
e.restore(m1);   // back to "Hello "
```

### Real-world examples

- Undo/redo stacks in editors, IDEs.
- Serializable snapshots / checkpoints in workflows.
- Java serialization (a heavy-weight version of the same idea).

---

## 19. Observer

> Define a **one-to-many dependency** so that when one object changes state, all dependents are notified.

```java
interface Observer<T> { void update(T event); }

class Subject<T> {
    private final List<Observer<T>> observers = new CopyOnWriteArrayList<>();
    public void subscribe(Observer<T> o)   { observers.add(o); }
    public void unsubscribe(Observer<T> o) { observers.remove(o); }
    protected void publish(T event) { observers.forEach(o -> o.update(event)); }
}
```

### Java 9+ variant — `Flow` / reactive streams

```java
import java.util.concurrent.Flow.*;

class Sensor implements Publisher<Integer> {
    private final SubmissionPublisher<Integer> pub = new SubmissionPublisher<>();
    public void subscribe(Subscriber<? super Integer> s) { pub.subscribe(s); }
    public void emit(int v) { pub.submit(v); }
}
```

### Real-world examples

- `java.beans.PropertyChangeListener`.
- `Flow.Publisher` / `Flow.Subscriber` (Java 9 reactive streams).
- RxJava, Project Reactor, Kafka consumers, event buses.
- Spring `ApplicationEventPublisher`.
- Browser DOM events, Node.js `EventEmitter`.

---

## 20. State

> Allow an object to **alter its behaviour when its internal state changes** — the object will appear to change class.

Replaces giant `switch (state)` blocks with polymorphism:

```java
interface OrderState {
    OrderState next();
    OrderState cancel();
}

class Created  implements OrderState { public OrderState next()   { return new Paid();    } public OrderState cancel() { return new Cancelled(); } }
class Paid     implements OrderState { public OrderState next()   { return new Shipped(); } public OrderState cancel() { throw new IllegalStateException(); } }
class Shipped  implements OrderState { public OrderState next()   { return new Delivered(); } public OrderState cancel() { throw new IllegalStateException(); } }
class Delivered implements OrderState { public OrderState next()  { return this; } public OrderState cancel() { throw new IllegalStateException(); } }
class Cancelled implements OrderState { public OrderState next()  { return this; } public OrderState cancel() { return this; } }

class Order {
    private OrderState state = new Created();
    public void advance()  { state = state.next(); }
    public void cancel()   { state = state.cancel(); }
}
```

### Real-world examples

- `ThreadPoolExecutor` internally transitions between states.
- Spring `StateMachine` framework.
- TCP connection states (`CLOSED`, `SYN_SENT`, `ESTABLISHED`, …).
- React component lifecycle (mount → update → unmount).

### Strategy vs State — interview trap

| Pattern   | Why it switches behaviour              | Who chooses               |
|-----------|----------------------------------------|---------------------------|
| Strategy  | Client picks one algorithm externally  | Client                    |
| State     | Object's own state drives transitions  | The object itself         |

---

## 21. Strategy

> Define a family of algorithms, encapsulate each one, and make them interchangeable.

```java
interface PricingStrategy { double price(ShoppingCart cart); }

class RegularPricing  implements PricingStrategy { public double price(ShoppingCart c) { return c.subtotal(); } }
class DiscountPricing implements PricingStrategy { public double price(ShoppingCart c) { return c.subtotal() * 0.9; } }

class Checkout {
    private PricingStrategy strategy;
    Checkout(PricingStrategy s) { this.strategy = s; }
    public double total(ShoppingCart c) { return strategy.price(c); }
}
```

### Java 8+ variant — lambdas replace strategy classes

```java
Checkout c = new Checkout(cart -> cart.subtotal() * 0.85);
// or
Map<String, Function<ShoppingCart, Double>> strategies = Map.of(
    "REGULAR",  ShoppingCart::subtotal,
    "DISCOUNT", c -> c.subtotal() * 0.9
);
```

If your strategy is a single-method interface ("functional interface"), a lambda is the modern idiom. Keep the class
form when the strategy holds state or has multiple methods.

### Real-world examples

- `Comparator<T>` — the canonical strategy in the JDK.
- `java.util.concurrent.RejectedExecutionHandler` policies.
- `Resource`-pattern: replace `if (locale == FR) ... else if (locale == DE) ...` with a strategy map.

---

## 22. Template Method

> Define the **skeleton of an algorithm** in a base class, deferring individual steps to subclasses.

```java
abstract class HttpRequestProcessor {
    public final void process(Request req, Response res) {     // template — fixed skeleton
        authenticate(req);
        authorize(req);
        res.setBody(handle(req));
        audit(req);
    }
    protected abstract void  authenticate(Request req);          // subclass steps
    protected void authorize(Request req) { /* default: allow */ }
    protected abstract Object handle(Request req);
    protected void audit(Request req)    { /* default: no-op */ }
}

class OrderProcessor extends HttpRequestProcessor {
    @Override protected void authenticate(Request r) { /* check JWT */ }
    @Override protected Object handle(Request r)     { return placeOrder(r); }
}
```

The base class controls the **flow**; subclasses fill in the **blanks**. Use the `final` keyword on the template to
prevent subclasses from breaking the ordering.

### Modern alternatives

- Strategy + composition (subclassing is rare in modern Java).
- Java 8 default methods on interfaces provide a lightweight template.

### Real-world examples

- `java.util.AbstractList` — `get()`, `size()` are abstract; `indexOf`, `clear` have default templates built on them.
- `HttpServlet` — `doGet`, `doPost`, etc.; `service()` is the template.
- Spring `JdbcTemplate`, `RestTemplate` — execute-the-boilerplate-and-call-back styles.

---

## 23. Visitor

> Add operations to an object structure **without modifying the classes** of the elements.

Useful for **stable** structures (e.g., ASTs, document models) where you frequently add new operations (type check,
pretty print, codegen) but rarely new element types.

```java
interface ShapeVisitor {
    void visit(Circle c);
    void visit(Square s);
}

interface Shape { void accept(ShapeVisitor v); }     // "double dispatch"

class Circle implements Shape {
    public double radius() { return 5; }
    public void accept(ShapeVisitor v) { v.visit(this); }
}

class Square implements Shape {
    public double side() { return 4; }
    public void accept(ShapeVisitor v) { v.visit(this); }
}

class AreaCalculator implements ShapeVisitor {
    private double total = 0;
    public void visit(Circle c) { total += Math.PI * c.radius() * c.radius(); }
    public void visit(Square s) { total += s.side() * s.side(); }
    public double total() { return total; }
}

List<Shape> shapes = List.of(new Circle(), new Square());
AreaCalculator calc = new AreaCalculator();
shapes.forEach(s -> s.accept(calc));
```

This is **double dispatch**: Java selects the overload based on the static type, so we make each element `accept` the
visitor and call `visitor.visit(this)` — `this` is statically typed, so the right overload is picked.

### Visitor + sealed interfaces (Java 17+)

```java
sealed interface Shape permits Circle, Square {}
record Circle(double radius) implements Shape {}
record Square(double side)   implements Shape {}

double area(Shape s) {
    return switch (s) {                              // pattern switch = visitor in 3 lines
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square q -> q.side() * q.side();
    };
}
```

Sealed types + pattern matching often replace Visitor with far less ceremony. Visitor remains useful for **open**
structures (elements can be added) or when you need stateful, named visitor classes.

### Real-world examples

- ASM bytecode library (`ClassVisitor`, `MethodVisitor`).
- Java compiler AST visitors (`JCTree.Visitor`).
- XPath/XSLT processors.

---

# Modern Considerations (Java 8–21)

| Pattern              | Modern shortcut                                                  |
|----------------------|------------------------------------------------------------------|
| Strategy             | Single-method interface → lambda / method reference              |
| Command              | `Runnable`, `Consumer<T>`, `Function<I,O>`                       |
| Iterator             | `Stream<T>` + `forEach`                                          |
| Observer             | `Flow.Publisher`/`Flow.Subscriber`, Project Reactor              |
| Template Method      | Default methods on interfaces, or Strategy + composition         |
| Factory Method       | `Supplier<T>` passed to constructor                              |
| Builder              | Records + compact constructors when fields are few               |
| Singleton            | Enum singleton, or Spring `@Component`                           |
| Visitor              | Sealed interface + switch pattern matching                       |
| Decorator            | `Function<T,T>` composition (`andThen`)                          |
| Chain of Resp.       | `Stream<Predicate<T>>` / function pipeline                       |

> The patterns don't disappear — they get **shorter**. Understanding the original form is still essential: you'll
> meet it in legacy code, libraries, and interview questions.

---

# Pattern Selection Cheat Sheet

| When you want to…                                                | Consider                          |
|------------------------------------------------------------------|-----------------------------------|
| Wrap a third-party API behind your own interface                 | Adapter                           |
| Build a complex object step by step                              | Builder                           |
| Add behaviour without subclassing                                | Decorator                         |
| Simplify access to a complex subsystem                           | Facade                            |
| Swap algorithms at runtime                                       | Strategy                          |
| React to state changes automatically                             | Observer                          |
| Behaviour depends on the object's state                          | State                             |
| Decouple abstraction from implementation, both varying           | Bridge                            |
| Treat individual and composite objects uniformly                 | Composite                         |
| Create families of related objects                               | Abstract Factory                  |
| Defer creation of an expensive object                            | Proxy (virtual) / Lazy init       |
| Process a request through several handlers                       | Chain of Responsibility           |
| Encapsulate a method call as a value (queue/undo)                | Command                           |
| Add operations to a stable object structure                      | Visitor                           |
| Define an algorithm skeleton, customise steps                    | Template Method                   |
| Share large numbers of fine-grained objects                      | Flyweight                         |
| Restore object state                                             | Memento                           |
| Centralise N×N communication between components                  | Mediator                          |
| Single global instance                                           | Singleton                         |
| Clone an existing object                                         | Prototype                         |

---

# Common Interview Questions

1. **What are the three GoF categories?**
   **Creational** (object construction), **Structural** (object composition), **Behavioral** (object interaction).

2. **Singleton — why "double-checked locking"? Why `volatile`?**
   Two threads can both pass the first `null` check. The lock prevents both creating the instance. `volatile` ensures
   no other thread sees a partially-constructed object due to instruction reordering.

3. **Difference between Factory Method and Abstract Factory?**
   Factory Method produces **one** product via subclassing; Abstract Factory produces a **family** of related products
   via composition.

4. **Difference between Builder and Prototype?**
   Builder constructs a complex object **step by step**; Prototype **clones** an existing instance.

5. **Adapter vs Bridge vs Decorator — what's the difference?**
   - **Adapter** makes an existing incompatible interface usable *after* it's built.
   - **Bridge** is designed **up-front** to decouple abstraction from implementation so they evolve independently.
   - **Decorator** *adds* behaviour without changing the interface; Adapter *changes* the interface.

6. **Composite vs Decorator?**
   Both use recursive composition. **Composite** treats a group the same as one element (part-whole hierarchy).
   **Decorator** adds responsibilities to a single object without changing its interface.

7. **Proxy vs Decorator?**
   Both wrap an object and forward calls. **Decorator** adds behaviour; **Proxy** controls access (lazy, remote,
   protection). They share structure but differ in intent.

8. **Strategy vs State?**
   Same structure (object delegates to a polymorphic collaborator). **Strategy** is chosen **externally** by the
   client. **State** transitions are driven **internally** by the object's own state.

9. **Observer vs Pub/Sub?**
   Observer is typically **synchronous** and **in-process** — the subject holds references to its observers. Pub/Sub
   decouples publisher and subscriber through a **broker** (Kafka, RabbitMQ) and is **asynchronous**.

10. **When would you NOT use a Singleton?**
    When the singleton is **mutable** (it becomes a global variable), when it makes code **hard to test**, or when you
    need multiple instances in different configurations. Prefer DI and a normal class.

11. **What does "double dispatch" mean in the Visitor pattern?**
    Java picks overloads by the **static** type. Visitor works around this with two virtual calls: `element.accept(v)`
    dispatches to the right element class, which then calls `visitor.visit(this)` — and `this` is now statically typed
    as the concrete element, so the right `visit` overload is selected.

12. **How have lambdas / records / sealed types changed GoF patterns?**
    Many single-method-interface patterns collapse to **lambdas** (Strategy, Command, Factory Method, Chain of
    Responsibility). **Records** shrink Builder for simple cases. **Sealed interfaces + pattern matching** replace
    Visitor. The intent of each pattern remains — the ceremony is smaller.

13. **Is MVC a GoF pattern?**
    No. MVC is an **architectural** pattern that *uses* several GoF patterns (Observer, Strategy, Composite). The GoF
    book mentions MVC as a motivating example.

14. **Favour composition over inheritance — which patterns illustrate this?**
    Strategy, State, Decorator, Adapter (object form), Bridge, Proxy, Composite, Facade — all use composition by
    default. Template Method, (class) Adapter, and Interpreter are the main inheritance-based exceptions.

---

# Mental Cheat-Sheet

> **5 / 7 / 11** — five Creational, seven Structural, eleven Behavioral.
> Three principles recur everywhere:
> 1. **Program to an interface, not an implementation.**
> 2. **Favour object composition over class inheritance.**
> 3. **Encapsulate what varies.**

```
   Creational          Structural          Behavioral
   ──────────          ───────────         ──────────
   Singleton           Adapter             Chain of Resp.
   Factory Method      Bridge              Command
   Abstract Factory    Composite           Interpreter
   Builder             Decorator           Iterator
   Prototype           Facade              Mediator
                        Flyweight           Memento
                        Proxy               Observer
                                            State
                                            Strategy
                                            Template Method
                                            Visitor
```

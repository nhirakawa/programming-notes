# Java Concurrency in Practice

## Chapter 2. Thread Safety

## Chapter 3. Sharing Objects

- This chapter is about objects that can be safely accessed by multiple threads
- `synchronized` is not just about atomicity or demarcating "critical sections"
  - Also memory visibility
- Allow other threads to see changes

### 3.1 Visibility

- For single-threaded environments, if you write a value and read then you get the same value
  - Not necessarily true for multiple threads
  - Generally no guarantee that a read will be eventually consistent

##### Example
```java
public class NoVisibility {

    private static boolean ready;
    private static int number;

    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready) {
                Thread.yield();
            }
            System.out.println(number);
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```

- It's possible for `NoVisibility` to loop forever
  - Could also print `0`
- Main thread writes to `number` then `ready`, but reader thread might see that in opposite order
  - Or not at all
- Allows JVM to take advantage of modern multiprocessor hardware

#### 3.1.1 Stale data

- `NoVisibility` demonstrates stale data
- Staleness is not all or nothing
  - Variable by variable basis

##### Listing 3.2 Non-thread-safe Mutable Integer Holder

  ```java
  @NonThreadSafe
  public class MutableInteger {
    private int value;

    public int get() { return value; }
    public void set(int value) { this.value = value; }
  }
  ```

  ##### Listing 3.3 Thread-safe Mutable Integer Holder

  ```java
@ThreadSafe
public class SynchronizedInteger {
  @GuardedBy("this") private int value;

  public synchronized int get() { return value; }
  public synchronized void set(int value) { this.value = value; }
}
  ```

#### 3.1.2 Nonatomic 64-bit operations

- Sales values were at some point valid
  - Not some random value
- Called _out-of-thin-air safety_
- Does not apply to non-`volatile` 64-bit numeric values (`long` and `double`)
- Java Memory Model requires fetch and store operations to be atomic, but JVM can treat a 64-bit read or write into 2 32-bit operations
  - Possible to read high bits from one value and low bits from another
  - When the JVM Specification was written, 64-bit arithmetic wasn't always efficient

#### 3.1.4 Locking and Visibility

- Intrinsic locking can be used to guarantee one thread sees the effects of another
- If thread B enters a `synchronized` block while thread A is executing, thread B will see all changes visibile to thread A just before thread A releases the lock
- Locking is not just about mutual exclusion
  - Also memory visibility

#### 3.1.4 Volatile Variables

- `volatile` variables are weaker form of synchronization
- Ensures updates to a variable are propagated predictably to other threads
- Compiler and runtime know that the variable is shared and memory operations should not be reordered.
- Not cached in registers or caches that are hidden from other processors
- Behaves similar `SynchronizedInteger` (Listing 3.3), but no locking and executing thread does not block
- Writing to a `volatile` variable is like a `synchronized` block
  - Affects other variables as well

##### Listing 3.4 Counting Sheep

```java
volatile boolean asleep;

while(!asleep) {
  countSomeSheep();
}
```

- Commonly used for completion/interruption/status flags
- `volatile` is not strong enough to make incrememnt operation atomic unless only written from a single thread
- Locking guarantees visibility and atomicity
  - `volatile` only guarantees visibility
- Use only when
  - Writes do not depend on current value, or single thread ever modifies value
  - Variable does not participate in invariants with other variables
  - Locking is not required for another reason while the variable is accessed

### 3.2 Publication and Escape

- _Publishing_ an object means making it available outside of its current scope
  - Storing a reference
  - Returning from a nonprivate method
  - Passing to a method in another class
- Publishing an object that should not have been is said to have _escaped_
- Most blatant form of publication is storing a reference in a `public static` field

##### Listing 3.5 Publishing an Object

```java
public static Set<Secret> knownSecrets;

public void initialize() {
  knownSecrets = new HashSet<>();
}
```

- Any `Secret` added to the set is also published

##### Listing 3.6 Allowing Internal Mutable State to Escape _Don't do this_

```java
class UnsafeStates {
  private String[] states = new String[] {
    "AK", "AL" ...
  };
  public String[] getStates() { return states; }
}
```

- Publishing `states` is problematic because any caller can modify it
- Any object that is _reachable_ from a published object through some chain of nonprivate field references or method calls is also published
- From the perspective of a class `C`, an _alien_ method is one whose behavior is not fully specified by `C`.
  - Includes methods in other classes and overrideable methods in `C`
- Passing an object to an alient method is publishing
  - Whether or not the alien method actually does publishing doesn't matter
- Inner class instances can also publish

##### Listing 3.7 Implicitly Allowing the `this` Reference to Escape. _Don't do this_

```java
public class ThisEscape {
  public ThisEscape(EventSource source) {
    source.registerListener(
      new EventListener() {
        public void onEvent(Event e) {
          doSomething(e);
        }
     });
  }
}
```

- `ThisListener` publishes `EventListener`, and implicily publishes the enclosing instance

#### 3.2.1 Safe Construction Practices

- `ThisEscape` illustrates `this` reference escaping during construction
- Publishing an object from within its constructor can publish an incompletely constructed object
  - Even if publication is last statement in constructor
- Oject is considered not property constructed
- Common mistake is starting a thread in a constructor
- Almost always shares `this` reference
  - By passing it into constructor or `Thread`/`Runnable` is an inner class
- Starting the thread from constructor, not simply creating, is the issue
  - Expose a `start` or `initialize` method that starts the owned thread
If tempted, avoid improper construction by using a private constructor and public factory method

##### Listing 3.8 Using a Factory Method to Prevent the `this` reference from Escaping During Construction

```java
public class SafeListener {
  private final EventListener listener;

  private SafeListener() {
    listener = new EventListener() {
      public void onEvent(Event e) {
        doSomething(e);
      }
    };
  }

  public static SafeListener newInstance(EventSource source) {
    SafeListener safe = new SafeListener();
    source.registerListener(safe.listener);
    return safe;
  }
}
```

### 3.3 Thread Confinement

- Accessing shared, mutable data requires synchronization
  - Or don't share
- _Thread confinement_ is simplest way to achieve thread safety
- JDBC uses thread confinement for `Connection` objects
  - Calling thread acquires a connection from the pool, uses it, and returns it
  - Pool will not dispense same connection to another thread until it has been returned
- Not enforceable by Java language

#### 3.3.1 Ad-hoc Thread Confinement

- Responsibility for maintaining thread confinement depends on implementation
- No language features assist
- Often a consequence of implementing a particular subsystem as a single thread
- Special case applies to `volatile` variables
  - Modification on single thread, ensures other threads always read the most up-to-date value
- Should be used sparingly

#### 3.3.2 Stack Confinement

- Special case of thread confinement
- Only reached through local variables
- Local variables intrinsically confined to executing thread
  - Exist on executing thread's stack, which is not accessible to other threads
- No way to obtain reference to primitive variables, so language semantics ensure that primitive local variables are always stack confined
 
 ##### Listing 3.9 Thread Confinement of Local Primitive and Reference Variables

 ```java
public int loadTheArk(Collection<Animal> candidates) {
  SortedSet<Animal> animals;
  int numPairs = 0;
  Animal canidate = null;

  // animals confined to method, don't let them escape
  animals = new TreeSet<Animal>(new SpeciesGenderComparator());
  animals.addAll(candidates);
  for(Animal a : animals) {
    if(candidate == null || !candidate.isPotentialMate(a)) {
      candidate = a;
    } else {
      ark.load(new AnimalPair(candidate, a));
      ++numPairs;
      candidate = null;
    }
  }

  return numPairs;
}
 ```

- Maintaining stack confinement for object references requires programmer assistance
- Reference to `TreeSet<Animal>` is only reference, confined to executing thread
- Must not publish `animals` or they will escape
- Using a thread-unsafe object in a within-thread context is thread-safe
  - This knowledge only exists in the head of the developer
  - If not clearly documented, object might escape in future development

#### 3.3.3 ThreadLocal

- `ThreadLocal` allows associating a pre-thread value with a value-holding object
- Provides `get` and `set` methods that maintain a separate copy of the value for each thread that uses it
  - `get` returns most recent value passed to `set` from currently executing thread
- Often used to prevent sharing in designs based on mutatble singletons or global variables
- Single-threaded application might maintain a global database connection that is initialized on startup
  - JDBC connections may not be thread-safe, so a multithreaded application that uses a global connection without coordination is not thread-safe
  - Using `ThreadLocal` means each thread has its own connection

##### Listing 3.10 Using `ThreadLocal` to Ensure thread Confinement

```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<>() {
  public Connection initialValue() {
    return DriverManager.getConnection(DB_URL);
  }
};

public static Connection getConnection() {
  return connectionHolder.get();
}
```

- Can also be used when a frequently used operation requires a temporary object and want to avoid reallocating the temporary object
  - Before Java 5.0, `Integer.toString` used a `ThreadLocal` to store a 12-byte buffer used for formatting its result, rather than a shared buffer or allocating a new buffer each time
  - Unlikely to be more performant unless operation is performed very frequently or allocation is unusually expensive
- When thread calls `ThreadLocal.get` for first time `initialValue` is consulted to provide initial value for that thread
  -`ThreadLocal<T>` is conceptually similar to `Map<Thread, T>`
- Thread-specific values are stored in the `Thread` object itself
  - When thread terminates, the thread-specific values can be garbage collected
- When converting single-threaded applications to multithreaded, you can preserve safety by converting shared global variables into `ThreadLocal`s
  - As long as they aren't intended to be global caches
- Easy to abuse `ThreadLocal` as license to create global variables

### 3.4 Immutability

- Other way to avoid synchronization is to use immutable objects
- Most atomicity and visibility issues so far are around accessing mutable state at the same time
  - If object cannot be modified, these issues aren't possible
- Immutable means cannot be changed after construction
- Always thread-safe
- Simple
  - Ony one state, which is controlled by constructor
- Safer
  - Passing mutable object to untrusted code is danger
  - No control over how object is used (or abused)
  - Not possible with immutable objects
- Immutability not formally defined
  - Not as simple as declaring references `final`
  - `final` field may be reference to mutable object
- Object is immutable if
  - State cannot be modified after construction
  - All fields are final
  - It is properly constructed (the `this` reference does not escape during construction)
- Can still use mutable objects internally to manage state

##### Listing 3.11 Immutable Class Built Out of Mutable Underlying Objects

```java
@Immutable
public final class ThreeStooges {
  private final Set<String> stooges = new HashSet<>();

  public ThreeStooges() {
    stooges.add("Moe");
    stooges.add("Larry");
    stooges.add("Curly");
  }

  public boolean isStooge(String name) {
    return stooges.contains(name);
  }
}
```

- Difference between object being immutable and reference being immutable
- Program state can be updated by replacing immutable objects with new instance holding updated state

#### 3.4.1 Final Fields

- The `final` keyword supports construction of immutable objects
- Reference cannot be modified
  - Objects can be modified, if mutable
- Guarantees initialization safety
- Mutable object with final fields makes reasoning easier
  - Object with 4 mutable fields is more complex than object with 2 mutable fields
- Good practice to make all fields final unless they need to be mutable

#### 3.4.2 Example: Using Volatile to Publish Immutable Objects

- Immutable objects can sometimes provide weak atomicity
- When a group of related data items must be acted on atomically, create an immutable holder class
  - A reference to the object means the object won't change
  - If the object needs to be changed, a new reference is created and other threads still see a consistent state

##### Listing 3.12 Immutable Holder for Caching a Number and its Factor

```java
@Immutable
class OneValueCache {
  private final BigInteger lastNumber;
  private final BigInteger[] lastFactors;

  public OneValueCache(BigInteger i,
                       BigInteger[] factors) {
    lastNumber = i;
    lastFactors = Arrays.copyOf(factors, factors.length);
  }

  public BigInteger[] getFactors(BigInteger i) {
    if (lastNumber == null || !lastNumber.equals(i)) {
      return null;
    } else {
      return Arrays.copyOf(lastFactors, lastFactors.length);
    }
  }
}
```

##### Listing 3.13 Caching the Last Result Using a Volatile Reference to an Immutable Holder Object

```java
@ThreadSafe
public class VolatileCachedFactorizer implements Servlet {
  private volatile OneValueCache cache = new OneValueCache(null, null);

  public void service(ServletRequest req, ServletResponse resp) {
    BigInteger i = extractFromRequest(req);
    BigInteger[] factors = cache.getFactors(i);
    if (factors == null) {
      factors = factor(i);
      cache = new OneValueCache(i, factors);
    }
    encodeIntoResponse(resp, factors);
  }
}
```

- `VolatileCachedFactorizer` uses a `OneValueCache` to store the cached number and factors
  - When a thread sets the volatile `cache` field to reference a new `OneValueCache`, the new cached data is immediately visible to other threads
- Cache operations cannot interfere because `OneValueCache` is immutable and `cache` field is only accessed once in each of the relevant code paths

### 3.5 Safe Publication

- Sometimes need to share objects across threads
- Cannot simply store reference a public field

##### Listing 3.14 Publishing an Object without Adequate Synchronization. _Don't do this_

```java
// Unsafe publication
public Holder holder;

public void initialize() {
  holder = new Holder(42);
}
```

- `Holder` could appear to be in an inconsistent state, even though invariants were established by constructor
- Could allow observing a partially constructed object

#### 3.5.1 Improper Publication: When Good Objects Go Bad

- Cannot rely on integrity of partially constructed objects
- Thread could see object in an inconsistent state, then state changes without modification since publication

##### 3.15 Class at Risk of Failure if Not Property Published

```java
public class Holder {
  private int n;

  public Holder(int n) { this.n = n; }

  public void assertSanity() {
    if(n != n) {
      throw new AssertionError("This statement is false");
    }
  }
}
```

- If `Holder` is published from Listing 3.14, a different thread could encounter an `AssertionError`
  - `Holder` ws not property published
- Two issues with improperly published objects
  - Other threads see `null` reference or older value, even though it has been set
  - Other threads see up-to-date value for `holder` reference but stale values for _state_ of the `Holder`
    - `Object` construtor first writes default values to all fields before subclass constructors run. It is possible to see the default value for a field as a stale value.

#### 3.5.2 Immutable Objects And Initialization Safety

- Java Memory Model offers special guarantee of initialization safety
- An object reference becoming visible to another thread doesn not mean that the state of that object is visible
  - Need synchronization to guarantee that
- Immutable objects can be accessed even when synchronization is not used to publish the object reference
- All requirements must be met
  - Unmodifiable state
  - All fields are `final`
  - Proper construction
- Guarantee extends to values of all `final` fields of properly constructed objects
  - `final` fields can be safely accessed without additional synchronization
  - If `final` fields refer to mutable objects, synchronization still required to access state of objects

#### 3.5.3 Safe Publication Idioms

- Non-immutable objects must be )safely published_
  - Synchronization for publishing and consuming thread
- Reference to the object and object's state must be visible at same time
- Properly constructed object can be safely published by (either)
  - Initializing object reference from static initializer
  - Storing reference in `volatile` field or `AtomicReference`
  - Storing reference in `final` field of a properly constructed object
  - Storing reference in field guarded by lock
- Synchronization in thread-safe collections fulfills last condition
- Thread-safe library collections offer publication guarantees (Javadoc not clear)
  - Placing a key or value in `Hashtable`, `synchronizedMap` or `ConcurrentMap` safely publishes to any thread that retrieves from map (directly or iterator)
  - Placing element in `Vector`, `CopyOnWriteArrayList`, `CopyOnWriteArraySet`, `synchronizedList` or `synchronizedSet` safely publishes to any thread that retrieves from collection
  - Placing an element on a `BlockingQueue` or `ConcurrentLinkedQueue` safely publishes to any thread that retrieves from queue
- `Future` and `Exchanger` also constitue safe publication
- Static initializer is easiest and safest way to publish objects
  - Executed by JVM at class initialization time
  - Guaranteed to safely publish objects initialized this way

#### 3.5.4 Effectively Immutable Objects

- Safe publication is sufficient for other threads to access (not modify) objects without synchronization
- As-published state is visible to all threads when reference is visible
  - If state isn't changed, any access is safe
- Mutable objects that aren't modified after publication are _effectively immutable_
  - Can simplify development and reduce need for synchronization
- Safely published effectively immutable objects can be used safely without synchronization

#### 3.5.5 Mutable Objects

- If object can be modified after construction, safe publication only ensures visibility of as-published state
  - Synchronization needs to be used when publishing and when accessing
  - Must be safely published _and_ either thread-safe or guarded by lock
- Publication requirements for object depends on mutablility
  - Immutable objects can be published however
  - Effectively immutable objects must be safely published
  - Mutable objects must be safely publisehd, and thread-safe or guarded by lock

#### 3.5.6 Sharing Objects Safely

- When acquiring a reference, know what you are allowed to do
- Useful policies for using/sharing objects
  - Thread-confined - Owned exclusively by and confined to one thread, can be modified by owning thread
  - Shared read-only - Access concurrently by multiple threads without synchronization, but cannot be modified by any thread. Include immutable and effectively immutable objects
  - Shared thread-safe - Performs synchronization internally, other threads need no futher synchronization
  - Guarded - Access only with specific lock. Include objects encapsulated in other thread-safe objects and published objects that are guarded by specific lock

## Chapter 4. Composing Objects

### 4.1 Designing a Thread-Safe Class

- Encapsulation makes it possible to determine that a class is thread-safe without reading entire program
- Designing a thread-safe class includes three elements
  - Identify variables that form object state
  - Identify invariants that constraint state variables
  - Establish policy for managing concurrent access to object state
- Object state starts with fields
  - If all primitive, fields are entire state
  - If object has references to objects, state encompasses fields from referenced objects

##### Listing 4.1 Simple Thread-safe Counter Using the Java Monitor Pattern

```java
@ThreadSafe
public final class Counter {
  @GuardedBy("this") private long value = 0;

  public synchronized long getValue() {
    return value;
  }

  public synchronized long increment() [
    if (value == Long.MAX_VALUE) {
      throw new IllegalStateException("counter overflow");
    }

    return ++value;
  ]
}
```

- Synchronization policy defines how object coordinates access to state
  - Specifies what combination of immutability, thread confinement, and locking is used to maintain thread safety and which variables are guarded by locks.

#### 4.1.1 Gathering Synchronization Requirements

- Making class thread-safe means ensuring invariants hold under concurrent access
- Objects and variables have state space
  - Range of possible values
- Smaller state space is easier to reason about
- By using final fields wherever practical, simpler to analyze possible states
  - Immutable objects can only be in single state
- Classes have invariants that identify states as valid or invalid
- The `value` in `Counter` is `long`
  - `long` can only have values from `Long.MIN_VALUE` to `Long.MAX_VALUE`
  - `Counter` places additional positive constraint
- Operations may have postconditions that identify invalid state transitions
  - If current state of `Counter` is `17`, only `18` is valid next state
- When next state is derived from current state, operation is compound action
- Constraints placed on state or state transition create synchronization or encapsulation requirements
- Class can have invariants that constraint multiple state variables
- `NumberRange` in Listing 4.10 maintains state variables for lower and upper bounds
  - Must obey constraint that lower bound is less than or equal to upper bound
- Multivariable invariants create atomicity requirements
  - Object may be in invalid state between acquiring locks

#### 4.1.2 State-dependent Operations

- Some objects have methods with state-based preconditions
  - Cannot remove item from empty queue
- Called _state-dependent_ operations
- In a single-threaded program, no other choice but to fail
- In concurrent program, precondition may become true later
  - Add possiblity of waiting until precondition is true
- Built-in mechanisms for waiting are tightly bound to intrinsic locking, difficult to use correctly
  - `wait` and `notify`
- Easier to use existing library classes
  - Blocking queues or semaphores

#### 4.1.3 State Ownership

- When defining object state, only consider the data the object owns
- Not embedded in language, element of class design
- When allocating/populating `HashMap`
  - Allocate a `HashMap` object and several `Map.Entry` objects
  - State of `HashMap` includes `HashMap` and all `Map.Entry` objects
- Ownership and encapsulation go together
  - Object enapsulates state and owns state it encapsulates
- Ownership means deciding locking protocol
  - Publishing a reference loses that control
- Collection classes are a form of "split ownership"
  - Collection owns state of infrastructure, client owns state of actual objects

#### 4.2 Instance Confinement

- If object is not thread-safe, techniques to let it be used safely
  - Ensure it is accessed from single thread
  - Ensure all access is guarded by lock
- Encapsulation promotes instance confinement
- All code paths have have access are known and can be analyzed more easily
- Combined with appropriate locking, can be used in thread-safe manner
- Encapsulation means data is always accessed with appropriate lock
- Confined objects must not escape intended scope
  - Class instance
  - Lexical scope (local variable)
  - Thread

##### Listing 4.2 Using Confinement to Ensure Thread Safety

```java
@ThreadSafe
public class PersonSet {
  @GuardedBy("this")
  private final Set<Person> mySet = new HashSet<>();

  public synchronized void addPerson(Person p) {
    mySet.add(p);
  }

  public synchronized boolean containsPerson(Person p) {
    return mySet.contains(p);
  }
}
```

- `PersonSet` is managed by `HashSet` (thread-unsafe)
  - `HashSet` is private and not allowed to escape
  - Access to `mySet` is guarded by locks
  - `PersonSet` is thread-safe
- Example makes no assumptions about thread-safety of `Person`
  - Most reliable is making `Person` thread-safe
  - Less reliable is to guard `Person` with locks
- Instance confinement allows choice in locking strategy
- Different state variables can be guarded by different locks
- Java synchronized `Collections` methods provide a synchronized wrapper around thread-unsafe classes
  - As long as only reference is wrapped, access is thread-safe

#### 4.2.1 The Java Monitor Pattern

- Object encapsulates all its mutable state and guards it with own intrinsic lock
- Any lock object can be used as long as use is consistent

##### Listing 4.3 Guarding State with a Private Lock

```java
public class PrivateLock {
  private final Object myLock = new Object();
  @GuardedBy("myLock") Widget widget;

  void someMethod() {
    synchronized(myLock) {
      // Access or modify state of widget
    }
  }
}
```

- Using a private lock means client code cannot acquire it
  - Clients that improperly acquire locks can cause liveness problems

#### 4.2.2 Example: Tracking Fleet Vehicles

- Build a vehicle tracker for dispatching fleet vehicles
  - Taxis, police cars, delivery trucks

```java
Map<String, Point> locations = vehicles.getLocations();
for (String key : locations.keyset()) {
  renderVehicle(key, locations.get(key));
}
```

```java
void vehicleMoved(VehicleMovedEvent event) {
  Point location = event.getNewLocation();
  vehicles.setLocation(event.getVehicleId(), location.x, location.y);
}
```

- Data model must be thread-safe because GUI thread and updater thread will access concurrently

##### 4.4 Monitor-based Vehicle Tracker Implementation

```java
@ThreadSafe
public class MonitorVehicleTracker {
  @GuardedBy("this")
  private final Map<String, MutablePoint> locations;

  public MonitorVehicleTracker(Map<String, MutablePoint> locations) {
    this.locations = deepCopy(locations);
  }

  public synchronized Map<String, MutablePoint> getLocations() {
    return deepCopy(locations);
  }

  public synchronized MutablePoint getLocation(String id) {
    MutablePoint location = locations.get(id);
    return loc == null ? null : new MutablePoint(location);
  }

  public synchronized void setLocation(String id) {
    MutablePoint location = locations.get(id);
    if(location == null) {
      throw new IllegalArgumentException("No such ID: " + id);
    }

    location.x = x;
    location.y = y;
  }

  private static Map<String, MutablePoint> deepCopy(Map<String, MutablePoint> map) {
    Map<String, MutablePoint> result = new HashMap<>();
    for (String id : map.keySet()) {
      result.put(id, new MutablePoint(m.get(id)));
    }

    return Collections.unmodifiableMap(result);
  }
}
```

##### Listing 4.5. Mutable Point Class Similar to `Java.awt.Point`

```java
@NotThreadSafe
public class MutablePoint {
  public int x, y;

  public MutablePoint() {
    x = 0;
    y = 0;
  }

  public MutablePoint(MutablePoint point) {
    this.x = point.x;
    this.y = point.y;
  }
}
```

- `MutablePoint` is not thread-safe, but tracker class is
- `MutablePoint`s are never published
- Copying mutable data may be performance issue at-scale

### 4.3 Delegating Thread Safety

- Sometimes need to add additional layer of safety, even if components are already thread-safe
- `CountingFactorizer` was thread-safe after adding an `AtomicLong`
  - Entire state of `CountingFactorizer` in `AtomicLong` and no additional validity requirements
- `CountingFactorizer` _delegates_ thread safety to `AtomicLong`

#### 4.3.1. Example: Vehicle Tracker Using Delegation

- Construct the vehicle tracker with thread-safe `Map`
  - `ConcurrentHashMap`
- Also store location in immutable `Point` class

##### 4.6 Immutable `Point` class used by `DelegatingVehicleTracker`

```java
@Immutable
public class Point {
  public final int x, y;

  public Point(int x, int y) {
    this.x = x;
    this.y = y;
  }
}
```

##### 4.7 Delegating Thread Safety to a `ConcurrentHashMap`

```java
@ThreadSafe
public class DelegatingVehicleTracker {
  private final ConcurrentMap<String, Point> locations;
  private final Map<String, Point>  unmodifiableMap;

  public DelegatingVehicleTracker(Map<String, Point> points) {
    locations = new ConcurrentHashMap<>(points);
    unmodifiableMap = Collections.unmodifiableMap(locations);
  }

  public Map<String, Point> getLocations() {
    return unmodifiableMap;
  }

  public Point getLocation(String id) {
    return locations.get(id);
  }

  public void setLocation(String id, int x, int y) {
    if(locations.replace(id, new Point(x, y)) == null) {
      throw new IllegalArgumentException("invalid vehicle name: " + id);
    }
  }
}
```

- Using `MutablePoint` would break encapsulation
  - `getLocations` would publish reference to (thread-unsafe) mutable state
- `getLocations` returns a live view of the map
  - Could be a benefit or a liability

##### Listing 4.8. Returning a Static Copy of the Location Set Instead of a "Live" One

```java
public Map<String, Point> getLocations() {
  return Collections.unmodifiableMap(new HashMap<>(locations));
}
```

#### 4.3.2. Independent State Variables

- Can also delegate thread safety to multiple state variables
  - As long as they are independent
  - Enclosing class does not impose invariants involving multiple state variables

##### Listing 4.9. Delegating Thread Safety to Multiple Underlying State Variables

```java
public class VisualComponent {
  private final List<KeyListener> keyListeners = new CopyOnWriteArrayList<>();
  private final List<MouseListener> mouseListeners = new CopyOnWriteArrayList<>();

  public void addKeyListener(KeyListener listener) {
    keyListeners.add(listener);
  }

  public void addMouseListener(MouseListener listener) {
    mouseListeners.add(listener);
  }

  public void removeKeyListener(KeyListener listener) {
    keyListeners.remove(listener);
  }

  public void removeMouseListener(MouseListener listener) {
    mouseListeners.remove(listener);
  }
}
```

- `CopyOnWriteArrayList` is a thread-safe `List` implementation
- `VisualComponent` delegates thread safety to underlying `mouseListeners` and `keyListeners`

#### 4.3.3. When Delegation Fails

- Most classes are not as simple

##### Listing 4.10. Number Range Class that does Not Sufficiently Protect Its Invariants. _Don't do this._

```java
public class NumberRange {
  // INVARIANT: lower <= upper
  private final AtomicInteger lower = new AtomicInteger(0);
  private final AtomicInteger upper = new AtomicInteger(0);

  public void setLower(int i) {
    // Warning -- unsafe check-then-act
    if (i > upper.get()) {
      throw new IllegalArgumentException("can't set lower to " + i + " > upper");
   }
   lower.set(i);
 }

 public void setUpper(int i) {
   // Warning -- unsafe check-then-act
   if (i < lower.get()) {
     throw new IllegalArgumentException("can't set upper to " + i + " < lower");
   }
   upper.set(i);
 }

 public boolean isInRange(int i) {
   return (i >= lower.get() && i <= upper.get());
 }
}
```

- `NumberRange` is not thread-safe
  - Does not preserve invariant that constrains `lower` and `upper`
- Could be made thread-safe by using locking
  - Guard `lower` and `upper` with common lock
- Must also avoid publishing `lower` and `upper`

#### 4.3.4. Publishing underlying state variables

- When can you publish state variables when delegating thread-safety to them?
  - It depends
- Publishing state variables means other classes can change them
  - May violate invariants
- If state variable is thread-safe, does not participate in invariants, and has no prohibited state transitions, it can be published

#### 4.3.5. Example: Vehicle Tracker that Publishes Its State

- Another version of vehicle tracker that publishes mutable state

##### Listing 4.11 Thread-safe Mutable Point Class

```java
@ThreadSafe
public class SafePoint {
  @GuardedBy("this") private int x, y;

  private SafePoint(int[] a) {
    this(a[0], a[1]);
  }

  public SafePoint(SafePoint safePoint) {
    this(safePoint.get());
  }

  public SafePoint(int x, int y) {
    this.x = x;
    this.y = y;
  }

  public synchronized int[] get() {
    return new int[] {x, y};
  }

  public synchronized void set(int x, int y) {
    this.x = x;
    this.y = y;
  }
}
```

- Provides getter that retrieves both `x` and `y` values at same time
  - If separate getters, values could change between calls

##### Listing 4.12 Vehicle Tracker That Safely Publishes Underlying State

```java
@ThreadSafe
public class PublishingVehicleTracker {
  private final Map<String, SafePoint> locations;
  private final Map<String, SafePoint> unmodifiableMap;

  public PublishingVehicleTracker(Map<String, SafePoint> locations) {
    this.locations = new ConcurrentHashMap<>(locations);
    this.unmodifiableMap = Collections.unmodifiableMap(this.locations);
  }

  public Map<String, SafePoint> getLocations() {
    return unmodifiableMap;
  }

  public SafePoint getLocation(String id) {
    return locations.get(id);
  }

  public void setLocation(String id, int x, int y) {
    if (!locations.containsKey(id)) {
      throw new IllegalArgumentException("invalid vehicle name: " + id);
    }
    locations.get(id).set(x, y);
  }
}
```

- Derives thread safety from delegation to `ConcurrentHashMap`
  - Contents are thread-safe mutable points
- Callers cannot add/remove vehicles, but can modify vehicle locations
  - Whether or not this is desired depends on the requirements

### 4.4. Adding Functionality To Existing Thread-Safe Classes

- Sometimes need to add new operations without undermining thread safety
- E.g. Adding an atomic put-if-absent method for a `List`
- May not always have access to underlying source
  - Also need to understand original implementation's synchronization policy
- Another approach is to extend the class
  - Fragile because synchronization policy now lives in multiple classes

##### Listing 4.13. Extending `Vector to have a `put-if-absent` method

 ```java
@ThreadSafe
public class BetterVector<E> extends Vector<E> {
  public synchronized boolean putIfAbsent(E x) {
    boolean absent = !contains(x);
    if(absent) {
      add(x);
    }
    return absent;
  }
}
 ```

#### 4.4.1. Client-side Locking

- For `ArrayList` wrapped with `Collections.synchronizedList`, neither approach works
- Third strategy is to extend the functionality without extending the class
  - Helper class

##### Listing 4.14. Non-thread-safe Attempt to Implement a put-if-absent. _Don't do this_

```java
@NotThreadSafe
public class ListHelper<E> {
  public List<E> list = Collections.synchronizedList(new ArrayList<>());

  public synchronized boolean putIfAbsent(E x) {
    boolean absent = !list.contains(x);
    if (absent) {
      list.add(x);
    }
    return absent;
  }
}
```

- Won't work because synchronizes on the wrong lock
- No guarantee another thread won't modify the list while `putIfAbsent` is executing
- `Vector` documentation uses intrinsic lock

##### Listing 4.15. Implementing put-if-absent with Client-Side Locking

```java
@ThreadSafe
public class ListHelper<E> {
  public List<E> list = Collections.synchronizedList(new ArrayList<>());

  public boolean putIfAbsent(E x) {
    synchronized(list) {
      boolean absent = !list.contains(x);
      if (absent) {
        list.add(x);
      }
      return absent;
    }
  }
}
```

- Even more fragile than extending a class
  - Puts locking code for class `C` into classes that are not `C`

#### 4.4.2. Composition

- Less fragile alternative

##### Listing 4.16. Implementing put-if-absent Using Composition

```java
@ThreadSafe
public class ImprovedList<T> implements List<T> {
  private final List<T> list;

  public ImprovedList(List<T> list) {
    this.list = list;
  }

  public synchronized boolean putIfAbsent(T x) {
    boolean contains = list.contains(x);
    if (contains) {
      list.add(x);
    }
    return !contains;
  }

  public synchronized void clear() {
    list.clear();
  }
  //...delegate to other List methods
}
```

- `ImprovedList` provides its own consistent locking
- Less fragile than attempting to mimic other object's locking strategy
- Guaranteed thread safety as long as our class holds only reference to underlying `List`

### 4.5. Documenting Synchronization Policies

- Users look to documentation to see if class is thread-safe
  - Document thread-safety
- Maintainers look to understand implementation strategy
  - Document synchronization policy
- Use of `synchronized`, `volatile`, or thread-safe class reflects synchronization policy
- Current documentation is lacking
  - `java.text.SimpleDateFormat` is not thread-safe, but not documented as such until JDK 1.4
-

#### 4.5.1. Interpreting Vague Documentation

- Need to guess based on how it will be implemented
- Servlets and database connections are frequently run in threads
  - Implementations should expect to need to deal with concurrent access

## Chapter 5. Building Blocks

### 5.1. Synchronized Collections

- Include `Vector` and `Hashtable`
  - Along with `Collections.synchronizedXxx` factory methods

#### 5.1.1. Problems with Synchronized Collections

- Thread-safe but may need additional locking for compound actions
  - Iteration, navigation, conditional operation (put-if-absent)
- Technically thread-safe, but may not behave as expected under concurrent access

##### Listing 5.1. Compound Actions on a `Vector` that may produce confusing results

```java
public static Object getLast(Vector list) {
  int lastIndex = list.size() - 1;
  return list.get(lastIndex);
}

public static void deleteLast(Vector list) {
  int lastIndex = list.size() - 1;
  list.remove(lastIndex);
}
```

- Operations can be interleaved
- Synchronized collections use implementation that allows client-side locking
  - Possible to create new operations that are atomic
  - Can acquire lock on synchronized collection object
- Size of list and corresponding get also affects iteration

##### Listing 5.2. Compound actions on `Vector` using client-side locking

```java
public static Object getLast(Vector list) {
  synchronized (list) {
    int lastIndex = list.size() - 1;
    return list.get(lastIndex);
  }
}

public static void deleteLast(Vector list) {
  synchronized(list) {
    int lastIndex = list.size() - 1;
    list.remove(lastIndex);
  }
}
```

##### Listing 5.3. Iteration that may throw `ArrayIndexOutOfBoundsException`

```java
for (int i = 0; i < vector.size(); i++) {
  doSomething(vector.get(i));
}
```

#### 5.1.2. Iterators and `ConcurrentModificationException`

- Standard way of iterating through a `Collection` is `Iterator`
  - Explicitly or with `for-each` loop
- Iterators returned by synchronized collections not designed to deal with concurrent modification
  - Fail-fast if collection has changed since iteration began
  - Throw unchecked `ConcurrentModificationException`
- Designed to catch errors on "good-faith-effort"
- Implemented by associating modification count with collection
  - If modification count changes during iteration, `hasNext` or `next` throws `ConcurrentModificationException`
  - Done without synchronization

##### Listing 5.5. Iterating a `List` with an `Iterator`

```java
List<Widget> widgetList = collections.synchronizedList(new ArrayList<>());
// May throw ConcurrentModificationException
for (Widget w : widgetList) {
  doSomething(w);
}
```

- Locking collection during iteration may be undesirable
- Other threads will block on access
  - Might wait a long time
- Risk factor for dead lock
- Even if it works, hurts application scalability
  - Lock contention leads to lower throughput and CPU utilization
- Alternative is to clone collection and iterate the copy
  - Thread-confined
  - Eliminates possibility of `ConcurrentModificationException`
  - Original collection must still be locked for clone
- Cloning still has performance cost

#### 5.1.3. Hidden Iterators

- Have to remember to use locking everywhere a shared collection might be iterated

#####  Listing 5.6. Iteration hidden within string concatenation. _Don't do this_

```java
public class HiddenIterator {
  @GuardedBy("this")
  private final Set<Integef> set = new HashSet<>();

  public synchronized void add(Integer i) {
    set.add(i);
  }

  public synchronized void remove(Integer i) {
    set.remove(i);
  }

  public void addTenThings() {
    Random r = new Random();
    for (int i = 0; i < 10; i++) {
      add(r.nextInt());
    }
    System.out.println("DEBUG: added ten elements to " + set);
  }
}
```

- String concatenation turns into `StringBuilder.append(Object)`, which invokes collection's `toString`, which iterates the collection and calls `toString` on each element
- The greater the distance between state and synchronization, the more likely someone will forget to use proper synchronization
- Iteration also invoked by collection's `hashCode` and `equals` methods
  - `containsAll`, `removeAll`, and `retainAll` methods will iterate over collection parameter

### 5.2 Concurrent Collections

- Java 5 provides several concurrent collection classes
- Synchronized collections serialize all access
  - Poor concurrency
- Also addes `Queue` and `BlockingQueue`
  - `ConcurrentLinkedQueue` and `PriorityQueue`
- Java 6 adds `ConcurrentSkipListMap` and `ConcurrentSkiplistSet`
  - Replace synchronized `SortedMap` and `SortedSet`

#### 5.2.1. ConcurrentHashMap

- Synchronized collections hold lock for duration of operation
  - Iterating collection might be necessary for some calls
- `ConcurrentHashMap` uses a more performant locking strategy
  - Lock striping
  - Arbitrary number of reading threads
  - Reads don't compete with writes
  - Limited number of writers
  - Higher throughput for concurrent access, little performance penalty for single-threaded access
- Concurrent collections provide iterators that do not throw `ConcurrentModificationException`
- Iterators are weakly-consistent
  - Tolerant of concurrent modification
  - Traverses elements as existed when iterator was constructed
  - May (but not guaranteed) relfect modifications after construction of iterator
- Semantics of methods that operate on entire map (`size`, `isEmpty`) are slightly weakened
  - Could be out of date
  - Generally not useful in concurrent environments, so weakness is tolerable
- Synchronized collections allow exclusive access
  - Only real tradeoff

#### 5.2.2. Additional Atomic Map Operations

- `ConcurrentMap` offers put-if-absent, remove-if-equal, and replace-if-equal atomic eoperations

##### Listing 5.7. `ConcurrentMap` interface

```java
public interface ConcurrentMap<K, V> extends Map<K, V> {
  // Insert into map only if no value is mapped from K
  V putIfAbsent(K key, V value);

  // Remove only if K is mapped to V
  boolean remove(K key, V value);

  // Replace value only if K is mapped to some value
  V replace(K key, V newValue);
}
```

#### 5.2.3. CopyOnWriteArrayList

- Concurrent replacement for synchronized `List`
- Rely on properly publishing immutable objects
- Create and republish new copy of collection every time it is modified
- Iterators retain reference to backing array from start of iteration
  - Only need to synchronize briefly to ensure visibility of array contents
- Some cose to copying the backing array
  - Reasonable only when iteration is far more common than modification
  - E.g. event-notification systems

### 5.3. Blocking Queues and the Producer-Consumer Pattern

- Blocking queues provide blocking `put` and `take` methods
  - Also timed `offer` and `put`
- If queue is full/empty, thread will block until condition is met
- Can be bounded or unbounded
- Support producer-consumer pattern
  - Separates the identification of work with execution of work
  - Producers place data in queue as it's available
  - Consumers take data when they are ready
- Producers/consumers don't need to know about the other
- Blocking queue simplifies consumers
  - `take` blocks until data is available
  - If producers aren't fast enough, consumers will wait
- If produers generate work faster than consumption, will run out of memory
  - Bounded queue will cause producer to block
- `offer` will return a failure status if item cannot be enqueued
  - Can use alternate strategy (shedding load, write to disk, scale down, throttling)

#### 5.3.1. Example: Desktop Search

- Producer task searches for files meeting index criteria, consumer task takes file names and indexes them

##### Listing 5.7. Producer and Consumer Tasks in a Desktop Search Application

```java
public class FileCrawler implements Runnable {
  private final BlockingQueue<File> fileQueue;
  private final FileFilter fileFilter;
  private final File root;

  public void run() {
    try {
      crawl(root);
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
    }
  }

  private void crawl(File root) throws InterruptedException {
    File[] entries = root.listFiles(fileFilter);
    if (entries != null) {
      for (File entry : entries) {
        if (entry.isDirectory()) {
          crawl(entry);
        } else if(!alreadyIndexed(entry)) {
          fileQueue.put(entry);
        }
      }
    }
  }
}

public class Indexer implements Runnable {
  private final BlockingQueue<File> queue;

  public Indexer(BlockingQueue<File> queue) {
    this.queue = queue;
  }

  public void run() {
    try {
      while (true) {
        indexFile(queue.take());
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }
}
```

##### Listing 5.9. Starting the Desktop Search

```java
public static void startIndexing(File[] roots) {
  BlockingQueue<File> queue = new LinkedBlockingQueue<File>(BOUND);
  FileFilter filter = new FileFilter() {
    public boolean accept(File file) { return true; }
  }

  for (File root : roots) {
    new Thread(new FileCrawler(queue, filter, root)).start();
  }

  for(int i = 0; i < N_CONSUMERS; i++) {
    new Thread(new Indexer(queue)).start();
  }
}
```

#### 5.3.2. Serial Thread Confinement

- Blocking queue implementations contain sufficient internal synchronization to safely publish objects from producer thread to consumer thread
- For mutable objects, facilitate serial thread confinement
  - Owned exclusively by thread, but ownership can be transferred by publishing it
- Object pools lend object to requesting thread
  - As long as pooled object is safely published and clients do not published pooled object or use it after returning, ownership is safely transferred
- Could use other publication methods but necessary to ensure only one thread receives it

#### 5.3.3. Deques and Work Stealing

- Java 6 also adds `Deque` ("deck") and `BlockingDeque`
  - Double-ended queue
- Useful for work stealing
- If consumer exhausts own deque, it steals work from tail of another deque
- Workers don't contend for a shared queue
  - When accessing other deques, read from tail instead of head further reduces contention

### 5.4. Blocking and Interruptible Methods

- Threads may block
  - I/O, lock, wake up from `Thread.sleep`, computation on another thread
- Usually suspended and placed in a blocked thread state
  - `BLOCKED`, `WAITING`, `TIMED_WAITING`
- Blocked thread must wait for an evant beyond its control
  - Then placed in `RUNNABLE` state and is eligible for scheduling
- `put` and `take` throw `InterruptedException`
  - Signals that it blocks, and that if interrupted it will make an effort to stop blocking early
- `Thread` provides `interrupt` for interrupting a thread and querying whether a thread has been interrupted
  - Boolean property that represents interrupted status
- Interruption is cooperative
  - When thread A interrupts thread B, it only requests that B stops when it is convenient
- When handling `InterruptedException`, there are basically 2 choices
  - Propagate the exception
  - Restore the interrupt
    - Must catch exception and call `interrupt` on current thread
    - Code higher in the call stack will see that an interrupt was issued

##### Listing 5.10. Restoring the Interrupted Status so as Not to Swallow the Interrupt

```java
public class TaskRunnable implements Runnable {
  BlockingQueue<Task> queue;

  public void run() {
    try {
      processTask(queue.take());
    } catch (InterruptedException e) {
      // restore interrupted status
      Thread.currentThread().interrupt();
    }
  }
}
```

- Should not swallow `InterruptedException`
  - Code higher in the call stack cannot respond to interruption
  - Only acceptable when extending Thread and control all code higher in the call stack

### 5.5. Synchronizers

- Blocking queues act as containers, and coordinate flow of producer and consumer threads
- Synchronizer is object that coordinates flow of threads based on its state
  - e.g. blocking queues, semaphores, barriers, latches
- Encapsulate state that determines whether threads should pass or wait
- Provide methods to manipulate state
- Provide methods to wait efficiently for synchronizer to enter desired state

#### 5.5.1. Latches

- Delays progress of threads until it reaches terminal state, like a gate
- Until latch reaches terminal state, gate is closed and no thread passes
- In terminal state, latch opens and all threads pass
- Once latch reaches terminal state, it cannot change state
  - Remains open forever
- Useful for one-time activities
  - Resource initialization
  - Dependent services
  - Waiting until all parties in activity are ready
- `CountDownLatch` allows one or more threads to wait for a set of events to occur
  - Latch state is a positive counter
  - `countDown` decrements counter, indicates method occurred
  - `await` waits for counter to reach 0

##### Listing 5.11. Using `CountDownLatch` for Starting and Stopping Threads in Timing Tests

```java
public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
  final CountDownLatch startGate = new CountDownLatch(1);
  final CountDownLatch endGate = new CountDownLatch(nThreads);

  for (int i  = 0; i < nThreads; i++) {
    Thread t = new Thread() {
      public void run() {
        try {
          startGate.await();
          try {
            task.run();
          } finally {
            endGate.countDown();
          }
        } catch (InterruptedException ignored) {}
      }
    };
    t.start();
  }

  long start = System.nanoTime();
  startGate.countDown();
  endGate.await();
  long end = System.nanoTime();
  return end-start;
}
```

- `TestHarness` creates threads to run concurrently
- Uses a starting gate and an ending gate
- Each thread waits on starting gate, then counts down end gate
- Master thread wiats until last worker has finished, so it can calculate elapsed time

#### 5.5.2. FutureTask

- Acts like a latch
- Implemented with a `Callable`
  - 3 states: waiting to run, running, completed
- When `FutureTask` enters completed state, stays that way forever
- `Future.get` dpends on state of task
  - If completed, returns result immediately
  - Otherwise blocks until task transitions to completed and returns result or throws exception
- Conveys result from thread executing to thread retrieving result
  - Guarantees that transfer is a safe publication
- Used by `Executor` to represent asynchronus tasks
- Can be used to represent lengthy computation that is started before results are needed

##### Listing 5.12. Using `FutureTask` to Preload Data that is Needed Later

```java
public class Preloader {
  private final FutureTask<ProductInfo> future = new FutureTask<ProductInfo>(new Callables<ProductInfo>() {
    public ProductInfo call() throws DataLoadException {
      return loadProductInfo();
    }
  });
}
private final Thread thread = new Thread(future);

public void start() { thread.start(); }

public ProductInfo get() throws DataLoadException, InterruptedException {
  try {
    return future.get();
  } catch (ExecutionException e) {
    Throwable cause = e.getCause();
    if (cause instanceof DataLoadException) {
      throw (DataLoadException) cause;
    } else {
      throw launderThrowable(cause);
    }
  }
}
```

- `Preloader` creates a `FutureTask` that loads product information from database
- Provides `start` method to start thread
  - Starting thread from constructor is indavisable
- `Callable` can throw checked and unchecked exceptions
  - Wrapped in `ExecutionException` and rethrown from `Future.get`
- Call site must deal with `ExecutionException`, and type is `Throwable`
- One of three categories
  - Checked exception thrown from `Callable`
  - `RuntimeException`
  - `Error`
- `launderThrowable` hanldes messy exception handling
- Test for known checked exceptions before dealing with others
  - Rethrow `Error` as-is
  - Return `RuntimeException`, which is generally rethrown
  - Throw `IllegalStateException` for non-`RuntimeException`s

##### Listing 5.13. Coercing an Unchecked `Throwable` to a `RuntimeException`

```java
public static RuntimeException launderThrowable(Throwable t) {
  if (t instanceof RuntimeException) {
    return (RuntimeException) t;
  } else if (t instanceof Error) {
    throw (Error) t;
  } else {
    throw new IllegalStateException("Not unchecked", t);
  }
}
```

#### 5.5.2. Semaphores

- Counting semaphores control number of activities that access resource or perform action at the same time
- Can be used to implement resource pools or impose bound
- `Semaphore` manages set of permits
  - Initial number is passed to constructor
- Activities acquire permits (if there are any available) and release them
  -  `acquire` blocks until a permit is available, or until interrupted, or until operation times out
  - `release` returns permit
- Degenerate case of binary semaphore
  - Can be used as a mutex
- Useful for resource pools
  - e.g. database connection pools
- Can also use to turn any collection into a bounded blocking collection

##### Listing 5.14. Using `Semaphore` to Bound a Collection

```java
public class BoundedHashSet<T> {
  private final Set<T> set;
  private final Semaphore sem;

  public BoundedHashSet(int bound) {
    this.set = Collections.synchronizedSet(new HashSet<>());
    sem = new Semaphore(bound);
  }

  public boolean add(T o) throws InterruptedException {
    sem.acquire();
    boolean wasAdded = false;
    try {
      wasAdded = set.add(o);
      return wasAdded;
    } finally {
      if(!wasAdded) {
        sem.release();
      }
    }
  }

  public boolean remove(Object o) {
    boolean wasRemoved = set.remove(o);
    if (wasRemoved) {
      sem.release();
    }
    return wasRemoved;
  }
}
```

#### 5.5.4. Barriers

- Latches are single-use
- Barriers are similar to latches
  - Block a group of threads until some event has occurred
- All threads must come together at the same time
- Latches are for waiting for an event, barriers are for waiting for threads
- `CyclicBarrier` allows fixed number of parties to rendevous repeatedly
  - Useful in parallel iterative algorithms that break down subproblems
- Threads call `await` and block until all threads have reached the barrier
- If `await` times out or a thread is interrupted, barrier is broken
  - All outstanding calls terminate with `BrokenBarrierException`
- If `await` is successful, returns a unique arrival index for each thread
  - Can be used to elect leader
- Can pass `Runnable` to barrier constructor
  - Runs when barrier is successfully passed, but before blocked threads are released
- Useful for simulations
  - Each step must complete before next step begins

##### Listing 5.15. Coordinating Computation in a Cellular Automaton with `CyclicBarrier`

```java
public class CellularAutomata {
  private final Board mainBoard;
  private final CyclicBarrier barrier;
  private final Worker[] workers;

  public CellularAutomata(Board board) {
    this.mainBoard = board;
    int count = Runtime.getRuntime().availableProcessors();
    this.barrier = new CyclicBarrier(
      count,
      new Runnable() {
        public void run() {
          mainBoard.commitNewValues();
        }
      }
    );
    this.workers = new Worker[count];
    for (int i = 0; i < count; i++) {
      workers[i] = new Worker(mainBoard.getSubBoard(count, i));
    }
  }

  private class Worker implements Runnable {
    private final Board board;

    public Worker(Board board) { this.board = board; }
    public void run() {
      while(!board.hasConverged()) {
        for (int x = 0; x < board.getMaxX(); x++) {
          for (int y = 0; y < board.getMaxY(); y++) {
            board.setNewValue(x, y, computeValue(x, y));
          }
        }
        try {
          barrier.await();
        } catch (InterruptedException ex) {
          return;
        } catch (BrokenBarrierException ex) {
          return;
        }
      }
    }
  }

  public void start() {
    for (int i = 0; i < workers.length; i++) {
      new Thread(workers[i]).start();
    }
    mainBoard.waitForConvergence();
  }
}
```

- `Exchanger` is a two-party barrier
  - Useful when performing asymetric activities
  - e.g. one thread fills buffer, other thread consumes data from buffer
- Exchange constitues safe publication of both objects to other party
- Timing of exchange depends on responsiveness requirements
  - Exchange when buffer is full and when buffer is empty
  - Exchange when buffer is full, or certain amount of time has elapsed

### 5.6. Building an Efficient, Scalable Result Cache

- Naive cache implementation turns performance bottleneck into scalability bottleneck

##### Listing 5.16. Initial Cache Attempt using `HashMap` and Synchronization

```java
public interface Computable<A, V> {
  V compute(A arg) throws InterruptedException;
}

public class ExpensiveFunction implements Computable<String, BigInteger> {
  public BigInteger compute(String arg) {
    // after deep though
    return new BigInteger(arg);
  }
}

public class Memoizer1<A, V> implements Computable<A, V> {
  @GuardedBy("this")
  private final Map<A, V> cache = new HashMap<>();
  private final Computable<A, V> c;

  public Memoizer1(Computable<A, V> c) {
    this.c = c;
  }

  public synchronized V compute(A arg) throws InterruptedException {
    V result = cache.get(arg);
    if (result == null) {
      result = c.compute(arg);
      cache.put(arg, result);
    }
    return result;
  }
}
```

- `Memoizer1` uses `HashMap` to store previous results
  - Returns result if cached, otherwise computes, stores, and returns
  - `HashMap` is not thread safe, so takes conservative approach of using `synchronized`

##### Listing 5.17. Replacing `HashMap` with `ConcurrentHashMap`

```java
public class Memoizer2<A, V> implements Computable<A, V> {
  private final Map<A, V> cache = new ConcurrentHashMap<>();
  private final Computable<A, V> c;

  public Memoizer2(Computable<A, V> c){ this.c = c; }

  public V compute(A arg) throws InterruptedException {
    V result = cache.get(arg);
    if (result == null) {
      result - c.compute(arg);
      cache.put(arg, result);
    }
    return result;
  }
}
```

- Multiple threads can use `Memoizer2` concurrently
  - Still possible for 2 threads to compute same value at same time
  - For memoization, just slow
  - More generally, safety risk
- Other threads not aware that another thread has already started computation
  - `FutureTask` does exactly this

##### Listing 5.18. Memoizing Wrapper using `FutureTask`

```java
public class Memoizer3<A, V> implements Computable<A, V> {
  private final Map<A, Future<V>> cache = new ConcurrentHashMap<A, Future<V>>();
  private final Computable<A, V> c;

  public Memoizer3(Computable<A, V> c) { this.c = c; }

  public V compute(final A arg) throws InterruptedException {
    Future<V> f = cache.get(arg);
    if (f == null) {
      Callable<V> eval = new Callable<>() {
        public V call() throws InterruptedException {
          return c.compute(arg);
        }
      };
      FutureTask<V> ft = new FutureTask<>(eval);
      f = ft;
      cache.put(arg, ft);
      ft.run();
    }
    try {
      return f.get();
    } catch (ExecutionException e) {
      throw launderThrowable(e.getCause());
    }
  }
}
```

- Good concurrency, result returned efficiently if already computed, and threads wait if computation is already running
- Still a small window when threads might compute same value
  - Need an atomic put-if-absent
- Caching `Future` means cache might be polluted
  - e.g. `Future` is cancelled or fails
  - Should remove cancelled futures (possible failed futures if future attempts would succeed)

##### Listing 5.19. Final Implementation of `Memoizer`

```java
public class Memoizer<A, V> implements Computable<A, V> {
  private final ConcurrentMap<A, Future<V>> cache = new ConcurrentHashMap<>();
  private final Computable<A, V> c;

  public Memoier (Computable<A, V> c) { this.c = c; }

  public V compute(final A arg) throws InterruptedException {
    while (true) {
      Future<V> f = cache.get(arg);
      if (f == null) {
        Callable<V> eval = new Callable<V>() {
          public V call() throws InterruptedException {
            return c.compute(arg);
          }
        };
        FutureTask<V> ft = new FutureTask<>(eval);
        f = cache.putIfAbsent(arg, ft);
        if (f == null) {
          f = ft;
          ft.run();
        }
        try {
          return f.get();
        } catch (CancellationException e) {
          cache.remove(arg, f);
        } catch (ExecutionException e) {
          throw launderThrowable(e.getCause());
        }
      }
    }
  }
}
```

##### Listing 5.20. Factorizing Servlet that Caches Results Using `Memoizer`

```java
@ThreadSafe
public class Factorizer implements Servlet {
  private final Computable<BigInteger, BigInteger[]> c = new Computable<>() {
    public BigInteger[] compute(BigInteger arg) {
      return factor(arg);
    }
  };
  private final Computable<BigInteger, BigInteger[]> cache = new Memoizer<>(c);

  public void service(ServletRequest req,
                      ServletResponse resp) {
    try {
      BigInteger i = extractFromRequest(req);
      encodeIntoResponse(resp, cache.compute(i));
    } catch (InterruptedException e) {
      encodeError(resp, "factorization interrupted");
    }
  }
}
```

## Chapter 6. Task Execution

### 6.1. Executing Tasks in Threads

- First step in organizing around task execution is task boundaries
- Ideally independent tasks
  - Work that doesn't depend on state, result, or side effects of other tasks
- For better scheduling and load balancing, tasks should represent small fraction of application's processing capacity
- Server aplications should exhibit good throughput and good responsiveness
  - Should also exhibit graceful degradation when overloaded

#### 6.1.1. Executing Tasks Sequentially

- Simplest execution policy is sequential

##### Listing 6.1. Sequential Web Server

```java
class SingleThreadWebServer {
  public static void main(String... args) throws IOException {
    ServerSocket socket = new ServerSocket(80);
    while (true) {
      Socket connection = socket.accept();
      handleRequest(connection);
    }
  }
}
```

- Theoretically correct, but not performant
- New requests must wait while server handles current request
- Processing web request involves computation and I/O
  - I/O usually means blocking
  - CPU is idle while waiting for I/O

#### 6.1.2. Explicitly Creating Threads for Tasks

- More responsive approach is new thread for each request

##### Listing 6.1.2. Explicitly Creating Threads for Tasks

```java
class ThreadPerTaskWebServer {
  public static void main(String... args) throws IOException {
    ServerSocket socket = new ServerSocket(80);
    while (true) {
      final Socket connection = socket.accept();
      Runnable task = new Runnable() {
        public void run() {
          handleRequest(connection);
        }
      };
      new Thread(task).start();
    }
  }
}
```

- Main thread still alternates between accepting incoming connection and dispatching request
  - Request handled in thread
- 3 consequences
  - Task processing occurs off main thread
    - New connections can be accepted before previous threads complete
  - Tasks can be processed in parallel
    - Improve throughput if multiple processors or blocking on I/O, locking, or resource availability
  - Task-handling code must be thread safe
- As long as request rate does not exceed server's capacity, this approach offers better responsiveness and throughput

#### 6.1.3. Disadvantages of Unbounded Thread Creation

- A few issues with thread-per-task approach
  - Thread lifecycle overhead
    - Creation/teardown not free
  - Resource consumption
    - Threads may sit idle, but still consume resources (e.g. memory)
    - Might lead to GC pressure
    - Threads may also compete for CPU
  - Stability
    - Limit to number of threads that can be created
    - Most likely result is `OutOfMemoryError`
- Best way to stay out of danger is place upper bound on how many threads are created
  - Also test application to ensure it doesn't run out of resources when thread limit reached
- Thread-per-task means no limit to number of threads except request rate
  - May appear fine until deployed and under heavy load

### 6.2. The Executor Framework

- Thread pools offer same benefit as bounded queues for thread management

##### Listing 6.3. `Executor` Interface

```java
public interface Executor {
  void execute(Runnable command);
}
```

- `Executor` provides standard means of decoupling task submission from task execution
  - Also provides lifecycle support and hooks for statistics gathering, application management, and monitoring
- Based on producer-consumer pattern

#### 6.2.1. Example: Web Server Using Executor

- Submission of request-handling task is decoupled from execution
- Can easily change `Executor` implementations
  - Configuration is one-time event and can be exposed for deploy-time configuration

##### Listing 6.4. Web Server Using a Thread Pool

```java
class TaskExecutionWebServer {
  private static final int NTHREADS = 100;
  private static final Executor exec = Executors.newFixedthreadPool(NTHREADS);

  public static void main(String... args) throws IOException {
    ServerSocket socket = new ServerSocket(80);
    while (true) {
      final Socket connection = socket.accept();
      Runnable task = new Runnable() {
        public void run() {
          handleRequest(connection);
        }
      };
      exec.execute(task);
    }
  }
}
```

##### Listing 6.5. `Executor` that Starts a New Thread for Each Task

```java
public class ThreadPerTaskExecutor implements Executor {
  public void execute(Runnable r) {
    new Thread(r).start();
  }
}
```

- Can also create an `Executor` that processes tasks sequentially

##### Listing 6.6 `Executor` that Executes tasks Synchronously in the Calling Thread

```java
public class WithinThreadExecutor implements Executor {
  public void execute(Runnable r) {
    r.run();
  }
}
```

#### 6.2.2. Execution Policies

- Decoupling submission from execution means changing execution policy is easy
  - "what, where, when, and how" of task execution
  - In what thread will tasks be executed
  - In what order should tasks be executed
  - How many tasks may execute concurrently
  - How many tasks may be queued pending executiong
  - If task is rejected because of system overload, which task is victim, and how is application notified
  - What actions are taken before/after executing a task
- Optimal policy depends on available resources and quality-of-service requirements

#### 6.2.3. Thread Pools

- Thread pools manage homogenous pool of worker threads
  - Tightly bound to work queue holding tasks to be executed
- Worker threads request next task from queue, execute it, then go back to waiting for task
- Reusing existing thread amortizes thread creation and teardown costs
- Since thread already exists, latency is also lower
- Proper tuning means threads keep processors busy without running out of memory or thrashing for resources
- Some useful pre-defined configurations in `Executors`
  - `newFixedThreadPool`
    - Creates threads as tasks are submitted, up to maximum pool size, then attempts to keep pool size constant
  - `newCachedThreadPool`
    - More flexibility to reap idle threads when current size of pool exceeds demand for processing, and add  new threads when demand increases
    - No bounds on size of pool
  - `newSingleThreadExecutor`
    - Single worker thread to process tasks, replaced if it dies unexpectedly
    - Guaranteed to be processed sequentially according to order imposed by task queue
  - `newScheduledThreadPool`
    - Fixed-size thread pool that supports delayed and periodic task execution
- `newFixedThreadPool` and `newCachedThreadPool` factories return instances of `ThreadPoolExecutor`
- Switching from thread-per-task to pool-based has big effect
  - Won't fail under heavy load
- Degrades more gracefully
  - No thousands of threads competing for CPU and memory
- Also opens door to tuning, management, monitoring, logging, error reporting, and other possibilities

#### 6.2.4. Executor Lifecycle

- JVM can't exit until all (nondaemon) threads have terminated
  - Failing to shutdown `Executor` may prevent JVM from exiting
- At any given time, state of previous submitted tasks is not obvious
- There is spectrum of ways to shutdown
  - Graceful shutdown (finish started, but don't accept new work)
  - Abrupt shutdown (turn off power to machine room)
- `ExecutorService` extends `Executor`
  - Adds methods for lifecycle management
- `Executors` should be able to be shutdown (both gracefully and abruptly) and expose information about status of tasks that were affected by shutdown

##### Listing 6.7. Lifecycle Methods in `ExecutorService`

```java
public interface ExecutorService extends Executor {
  void shutdown();
  List<Runnable> shutdownNow();
  boolean isShutdown();
  boolean isTerminated();
  boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
  ...
}
```

- `ExecutorService` has 3 states
  - Running
  - Shutting down
  - Terminated
- Initially created in running state
- `shutdown` initiates graceful shutdown
  - No new tasks are accepted but already submitted tasks are completed
- `shutdownNow` initiates abrupt shutdown
  - Cancels outstanding tasks and does not start any enqueued tasks
- Tasks submitted after shutdown are handled by rejected execution handler
  - Might silently discard task or throw unchecked `RejectedExecutionException`
- When tasks have completed, moves to `terminated` state
- Can wait for terminated state with `awaitTermination` or polling with `isTerminated`
  - Common to follow `shutdown` with `awaitTermination`

- `LifecycleWebServer` extends web server with lifecycle support
  - Can stop with `stop` method or through client HTTP request

##### Listing 6.8. Web Server with Shutdown Support

```java
class LifecycleWebServer {
  private final ExecutorService exec = ...;

  public void start() throws IOException {
    ServerSocket socket = new ServerSocket(80);
    while (!exec.isShutdown()) {
      try {
        final Socket conn = socket.accept();
        exec.execute(new Runnable() {
          public void run() { handleRequest(conn); }
        });
      } catch (RejectedExecutionException e) {
        if (!exec.isShutdown()) {
          log("task submission rejected", e);
        }
      }
    }
  }

  public void stop() { exec.shutdown(); }

  void handleRequest(Socket connection) {
    Request req = readRequest(connection);
    if (isShutdownRequest(req)) {
      stop();
    } else {
      dispatchRequest(req);
    }
  }
}
```

#### 6.2.5. Delayed and Periodic Tasks

- `Timer` manages execution of deferred and periodic tasks
- Has some drawbacks that `ScheduledThreadPoolExecutor` patches up
- `Timer` creates only single thread for executing timer tasks
  - If one task takes too long, may affect accuracy of other `TimerTask`s
- Scheduled thread pools allow multiple threads for execution
- `Timer` behaves poorly if `TimerTask` throws unchecked exception
  - Does not catch exception
  - Terminates timer thread, and it is not resurrected

##### Listing 6.9. Class Illustrating Confusing `Timer` Behavior

```java
public class OutOfTime {
  public static void main(String... args) throws Exception {
    Timer timer = new Timer();
    timer.schedule(new ThrowTask(), 1);
    SECONDS.sleep(1);
    timer.schedule(new ThrowTask(), 1);
    SECONDS.sleep(5);
  }

  static class ThrowTask extends TimerTask {
    public void run() { throw new RuntimeException(); }
  }
}
```

- Might expect `OutOfTime` to run for 6 seconds and exit
  - Actually terminates after 1 second with `IllegalArgumentException` with message `"Timer already cancelled"`
- If building scheduling service, may be able to use `DelayQueue`
  - `BlockingQueue` that provides scheduling functionality of `ScheduledThreadPoolExecutor`
- Manages collection of `Delayed` objects
- Only lets you take `Delayed object` if its delay has expired
- Ordered by time associated with delay

### 6.3. Finding Exploitable Parallelism

- Must describe task as `Runnable` to use `Executor`s
- Example takes page of HTML and renders it to image buffer
  - Only text interspersed with image elemnts with pre-specified dimensions and URLs

#### 6.3.1. Example: Sequential Page Renderer

- Simplest approach is sequential
- Can also render text elements first, leaving placeholders for images
  - Then go back and fetch/render images

##### Listing 6.10. Rendering Page Elements Sequentially

```java
public class SingleThreadRenderer {
  void renderPage(CharSequence source) {
    renderText(source);
    List<ImageData> imageData = new ArrayList<>();
    for (ImageInfo imageInfo : scanForImageInfo(source)) {
      imageData.add(imageInfo.downloadImage());
    }
    for (ImageData data : imageData) {
        renderImage(data);
      }
  }
}
```

#### 6.3.2. Result-bearing Tasks: `Callable` and `Future`

- `Executor` uses `Runnable` but fairly limiting abstracting
  - Cannot return value or throw checked exceptions
  - Can have side effects
- Many tasks are deferred computations
- `Callable` is a better abstraction for this
- Tasks are finite
  - Starting point and eventually terminate
- 4 phases
  - Created
  - Submitted
  - Started
  - Completed
- `Future` represents lifecycle of task
  - Provides methods to test whether task has completed or been cancelled, retrieve its result, and cancel task
- Behavior of `get` varies depending on task state (not yet started, running, completed)
  - Returns immediately or throws `Exception` if task has already completed, and blocks until completed
  - If task completes by throwing exception, `get` rethrows in wrapped `ExecutionException`
  - If cancelled, throws `CancellationException`
  - If `ExecutionException`, `getCause` reveals underlying cause

##### Listing 6.11. `Callable` and `Future` Interfaces

```java
public interface Callable<V> {
  V call() throws Exception;
}

public interface Future<V> {
  boolean cancel(boolean mayInterruptIfRunning);
  boolean isCancelled();
  boolean isDone();
  V get() throws InterruptedException, ExecutionException, CancellationException;
  V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, CancellationException, TimeoutException;
}
```

- Several ways to create `Future`
  - `ExecutorService#submit` returns `Future`
  - Instantiate `FutureTask`
- In Java 6, `ExecutorService` can override `newTaskFor` to control instantiation of `Future`
  - Default implementation creates new `FutureTask`

##### Listing 6.12. Default Implementation of `newTaskFor` in `ThreadPoolExecutor`

```java
protected <T> RunnableFuture<T> newTaskFor(Callable<T> task) {
  return new FutureTask<>(task);
}
```

- Submitting to `Executor` constitutes safe publication
- Setting result value for `Future` constitutes safe publication to any thread that calls `get`

#### 6.3.3. Example: Page Renderer with Future

- Divide into 2 tasks
  - Render text
  - Downloads images
- Create `Callable` to download images and submit to `Executor`
- By the time we need images, they're hopefully available
- Better, but no need to wait for _all_ images before drawing first
  - Probably better to draw individual images as available

##### Listing 6.13. Waiting for Image Download with `Future`

```java
public class FutureRenderer {
  private final ExecutorService executor = ...;

  void renderPage(CharSequence source) {
    final List<ImageInfo> imageInfos = scanForImageInfo(source);
    Callable<List<ImageData>> task = new Callable<List<ImageData>>() {
      public List<ImageData> call() {
        List<ImageData> result = new ArrayList<>();
        for (ImageInfo imageInfo : imageInfos) {
          result.add(imageInfo.downloadImage());
        }
        return result;
      }
    };
    Future<List<ImageData>> future = executor.submit(task);
    renderText(source);

    try {
      List<ImageData> imageData = future.get();
      for (ImageData data : imageData) {
        renderImage(data);
      }
    } catch (InterruptedException e) {
        // Re-assert the thread's interrupted status
        Thread.currentThread().interrupt();
        // We don't need the result, so cancel the task too
        future.cancel(true);
    } catch (ExecutionException e) {
      throw launderThrowable(e.getCause());
    }
  }
}
```

#### 6.3.4. Limitations of Parallelizing Heterogeneous Tasks

- Heterogenous task execution doesn't scale well
  - Need finer-grained parallelism
- If different tasks take longer, may not see much speedup
- Division requires coordination overhead
- Real speedup when large number of independent, homogenous tasks

#### 6.3.5. `CompletionService`: Executor Meets `BlockingQueue`

- If want to retrieve results as available from batch of computations, could poll `Future`s with timeout of 0
  - Tedious
- `CompletionService` combines functionality of `Executor` and `BlockingQueue`
  - Can submit `Callable`s and call `take` and `poll` to retrieve results (packaged as `Future`s)
- `ExecutorCompletionService` implements `CompletionService`, delegates to `Executor`

##### Listing 6.14. `QueueingFuture` Class used By `ExecutorCompletionService`

```java
private class QueueingFuture<V> extends FutureTask<V> {
  QueueingFuture(Callable<V> c){ super(c); }
  QueueingFuture(Runnable t, V r){ super(t, r); }

  protected void done() {
    completionQueue.add(this);
  }
}
```

#### 6.3.6. Example: Page Renderer with `CompletionService`

- Create separate task for downloading _each_ image

##### Listing 6.15. Using `CompletionService` to render page elements as they become available

```java
public class Renderer {
  private final ExecutorService executor;

  Renderer(ExecutorService executor) { this.executor = executor; }

  void renderPage(CharSequence source) {
    List<ImageInfo> info = scanForImageInfo(source);
    Completionservice<ImageData> completionService = new ExecutorCompletionService<ImageData>(executor);
    for (final ImageInfo inageInfo : info) {
      completionService.submit(new Callable<ImageData>() {
        public ImageData call() {
          return imageInfo.downloadImage();
        }
      });
    }
    renderText(source);
    try {
      for (int t = 0, n = info.size(); t < n; t++) {
        Future<ImageData> f = completionService.take();
        ImageData imageData = f.get();
        renderImage(imageData);
      }
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
    } catch (ExecutionException e) {
      throw launderThrowable(e.getCause());
    }
  }
}
```

- Multiple `ExecutorCompletionSerivce`s can share single `Executor`
- Private `CompletionService`s sharing common `Executor` acts as handle for batch of computations

#### 6.3.7. Placing Time Limits on Tasks

- Sometimes result isn't needed after certain amount of time
- `Future` supports timeouts by throwing `TimeoutException`
- Also need to cancel any work being done to release resources
  - If `Future` times out, can cancel the task

##### Listing 6.16. Fetching an Advertisement with a Time Budget

```java
Page renderPageWithAd() throws InterruptedException {
  long endNanos = System.nanoTime() + TIME_BUDGET;
  Future<Ad> f = exec.submit(new FetchAdTask());
  Page page = renderPageBody();
  Ad ad;
  try {
    long timeLeft = endNanos - System.nanoTime();
    ad = f.get(timeLeft, NANOSECONDS);
  } catch (ExecutionException e) {
    ad = DEFAULT_AD;
  } catch (TimeoutException e) {
    ad = DEFAULT_AD;
    f.cancel(true);
  }
  page.setAd(ad);
  return page;
}
```

#### 6.3.8. Example: A Travel Reservations Portal

- Can generalize time budget to arbitrary number of tasks
- Rather than wait for slowest request, might be better to render only what's ready after certain time period

##### Listing 6.17. Requesting Travel Quotes Under a Time Budget

```java
private class QuoteTask implements Callable<TravelQuote> {
  private final TravelCompany company;
  private final TravelInfo travelInfo;
  ...
  public TravelQuote call() throws Exception {
    return company.solicitQuote(travelInfo);
  }
}

public List<TravelQuote> getRankedTravelQuotes(TravelInfo travelInfo,
                                               Set<TravelCompany> companies,
                                               Comparator<TravelQuote> ranking,
                                               long time,
                                               TimeUnit unit) throws InterruptedException {
    List<QuoteTask> tasks = new ArrayList<>();
    for(TravelCompany company : companies) {
      tasks.add(new QuoteTask(company, travelInfo));
    }

    List<Future<TravelQuote>> futures = exec.invokeAll(tasks, time, unit);

    List<TravelQuote> quotes = new ArrayList<>(tasks.size());
    Iterator<QuoteTask> taskIter = tasks.iterator();
    for(Future<TravelQuote> f : futures) {
      QuoteTask task = taskIter.next();
      try {
        quotes.add(f.get());
      } catch (ExecutionException e) {
        quotes.add(task.getFailureQuote(e.getCause()));
      } catch (CancellationException e) {
        quotes.add(task.getTimeoutQuote(e));
      }
    }

    Collections.sort(quotes, ranking);
    return quotes;
  }
)
```

- `invokeAll` takes collection of tasks and returns collection of `Future`s
  - Timed version will return when all tasks have completed, the calling thread is interrupted, or timeout expires
  - Any tasks not complete at timeout are cancelled
  - On return, each task will be completed or cancelled

## Chapter 7. Cancellation and Shutdown

- Sometimes want to stop tasks or threads early
  - Cancel operation or application needs to shutdown
- Java has no way to force-stop a thread
  - Interruption is cooperative
- Rarely want immediate stop
  - Could leave shared data structures in inconsistent state
- Instead clean up any in-progress work then terminate

### 7.1. Task Cancellation

- Activity is cancellable if external code can move it to completion earlier than normal
- Several reasons why
  - User-requested cancellation
  - Time-limited activities
  - Application events
  - Errors
  - Shutdown
- One cooperative mechanism is setting flag
  - If flag is set, task terminates early

##### Listing 7.1. Using a `volatile` Field to hold Cancellation State

```java
@ThreadSafe
public class PrimeGenerator implements Runnable {
  @GuardedBy("this")
  private final List<BigInteger> primes = new ArrayList<>();
  private volatile boolean cancelled;

  public void run() {
    BigInteger p = BigInteger.ONE;
    while (!cancelled) {
      p = p.nextProbablyPrime();
      synchronized (this) {
        primes.add(p);
      }
    }
  }

  public void cancel() { cancelled = true; }

  public synchronized List<BigInteger> get() {
    return new ArrayList<>(primes);
  }
}
```

- Polls `volatile` flag and returns early if the flag is set

##### Listing 7.2. Generating a Second's Worth of Prime Numbers

```java
List<BigInteger> aSecondOfPrimes() throws InterruptedException {
  PrimeGenerator generator = new PrimeGenerator();
  new Thread(generator).start();
  try {
    SECONDS.sleep(1);
  } finally {
    generator.cancel();
  }
  return generator.get();
}
```

- May not stop after eactly 1 second
  - Some delay between cancellation and next loop check

- Code must have cancellation policy
  - "how", "when", "what" other code can request cancellation

#### 7.1.1. Interruption

- Cancellation in `PrimeGenerator` may take a while
- If task calls blocking method, task might never check cancellation flag and might never terminate

##### Listing 7.3. Unreliable Cancellation That Can Leave Producers in a Blocking Operation. _DO NOT DO THIS_

```java
class BrokenPrimeProducer extends Thread {
  private final BlockingQueue<BigInteger> queue;
  private volatile boolean cancelled = false;

  BrokenPrimeProducer(BlockingQueue<BigInteger> queue) {
    this.queue = queue;
  }

  public void run() {
    try {
      BigInteger p = BigInteger.ONE;
      whiel (!cancelled) {
        queue.put(p = p.nextProbablePrime());
      } catch (InterruptedException consumed){ }
    }
  }

  public void cancel() { cancelled = true; }
}

void consumePrimes() throws InterruptedException {
  BlockingQueue<BigInteger> primes = ...;
  BrokenPrimeProducer producer = new BrokenPrimeProducer(primes);
  producer.start();
  try {
    while (needMorePrimes()) {
      consume(primes.take());
    }
  } finally {
    producer.cancel();
  }
}
```

- If producer gets ahead of consumer and queue is full, `put` will block
- If consumer tries to cancel, it will set cancelled flag but producer will never check
  - Still blocked
- Nothing official, but using interruption for anything except cancellation is suspect
- Each thread has boolean interrupted status
  - Interrupting thread sets the status to true
- `interrupt` method interrupts target thread
- `isInterrupted` returns interrupted status
- `interrupted` clears interrupted status of current thread and returns previous value
- `Thread.sleep` and `Object.wait` try to detect interruption and return early
  - Respond by clearing interrupted status and throwing `InterruptedException`
  - No guarantee how quickly this happens, but usually quick

##### Listing 7.4. Interruption Methods in `Thread`

```java
public class Thread {
  public void interrupt() { ... }
  public boolean isInterrupted(){ ... }
  public static boolean interrupted() { ... }
}
```

- If thread interrupted when not blocked, interrupted status is set
  - Up to activity being performed to detect interruption
- Interruption is sticky
  - If no `InterruptedException`, status will stay until it is explicitly cleared
- Interruption is only a request
- Well behaved methods may ignore interruption as long as they allow calling code to do something with it
  - Poorly behaved methods swallow the interruption
- `interrupted` should be used with caution
  - Clears current status
  - If returns `true`, either throw `InterruptedException` or restore interrupted status (`interrupt`)
- `BrokenPrimeProducer` can be fixed with interruption
  - 2 places where interruption can be detected
    - Blocking `put` call
    - Polling interrupted status in loop

##### Listing 7.5. Using Interruption for Cancellation

```java
class PrimeProducer extends Thread {
  private final BlockingQueue<BigInteger> queue;

  PrimeProducer(BlockingQueue<BigInteger> queue) {
    this.queue = queue;
  }

  public void run() {
    try {
      BigInteger p = BigInteger.ONE;
      while (!Thread.currentThread().isInterrupted()) {
        queue.put(p = p.nextProbablePrime());
      } catch (InterruptedException consumed) {
        // Allow thread to exit
      }
    }

    public void cancel() { interrupt(); }
  }
}
```

#### 7.1.2. Interruption Policies

- Interruption policy determines how thread interprets interruption
  - What it does
  - What units of work are atomic with interruption
  - How quickly it reacts to interruption
- Most sensible is thread-level or service-level cancellation
  - Exit as quickly as possible (with cleanup)
  - Possibly notify something that thread is exiting
- Possible to pause/resume a service
  - Thread pools with nonstandard interruption policies restricted to tasks that are aware of policy
- Interruption may mean both cancel current task and shutdown worker thread
- Tasks don't execut in threads they own
  - Execute in borrowed threads (e.g. thread pool)
- Guest code should preserve interrupted status
- Most blocking library methods will simply throw `InterrutedException`
  - Get out of the way and let caller take action
- Task doesn't need to respond to interruption immediately
  - May clean up data structures
- Task shouldn't assume interruption policy of executing thread
  - Unless service has specific interruption policy
- Cancellation code should not assumpe interruption policy of arbitrary threads
- Thread should only be interrupted by its owner
- Should not interrupt thread unless you know what interruption means for that thread

#### 7.1.3. Responding to Interruption

- 2 strategies for handling `InterruptedException`
  - Propagate exception
  - Restore interruption status so calling code can deal with it

##### Listing 7.6. Propagating `InterruptedException` to Callers

```java
BlockingQueue<Task> queue;
...
public Task getNExtTask() throws InterruptedException {
  return queue.take();
}
```

- Can be as adding `InterruptedException` to `throws` clause
- If can't propagate `InterruptedException`, need to preserve interruption request
  - Standard way is to call `interrupt` again
- Do not swallow `InterruptedException`
- Only code that implements thread's interruption policy may swallow interruption request
- Activities that do not support cancellation but call interruptible blocking methods need to call in loop
  - Retry when interrupted
  - Preserve interrupt status and restore just before returning
  - Setting too early could result in infinite loop

##### Listing 7.7. Noncancelable Task That Restores Interruption Before Exit

```java
public Task getNextTask(BlockingQueue<Task> queue) {
  boolean interrupted = false;
  try {
    while (true) {
      try {
        return queue.take();
      } catch (InterruptedException e) {
        interrupted = true;
        // fall through and retry
      }
    }
  } finally {
    if (interrupted) {
      Thread.currentThread().interrupt();
    }
  }
}
```

- If code does not call interruptible blocking methods, could poll interrupted status
- Interruption could be used to get thread's attention
  - Out-of-band information for more instruction about what to do

#### 7.1.4. Example: Timed Run

- Putting upper bound on execution time can be useful
- `aSecondOfPrimes` starts a `PrimeGenerator` and interrupts after a second
  - Will eventually stop
  - Want to know if task throws an exception

##### Listing 7.8. Scheduling an Interrupt on a Borrowed Thread. _Don't do this._

```java
private static final ScheduledExecutorService cancelExec = ...;

public static void timedRun(Runnable r, long timeout, TimeUnit unit) {
  final Thread taskThread = Thread.currentThread();
  cancelExec.schedule(
    () -> taskThread.interrupt(),
    timeout,
    unit
  );
  r.run();
}
```

- Violates rule of not interrupting threads when not knowing interruption policy

##### Listing 7.9. Interrupting a Task in a Dedicated Thread

```java
public static void timedRun(final Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
  class RethrowableTask implements Runnable {
    private volatile Throwable t;
    public void run() {
      try {
        r.run();
      } catch (Throwable t) {
        this.t = t;
      }
    }

    void rethrow() {
      if (t != null) {
        throw launderThrowable(t);
      }
    }
  }

  RethrowableTask task = new RethrowableTask();
  final Thread taskThread = new Thread(task);
  cancelExec.schedule(
    () -> taskThread.interrupt();,
    timeout,
    unit
  );

  taskThread.join(unit.toMillis(timeout));
  task.rethrow();
}
```

- Created thread can have own execution policy
- Even if task doesn't respond to interrupt, run method can return to caller
- `join` will rethrow an exception (in same thread as `timedRun` is called) if one was thrown
  - No information about whether or not `join` returned normally or because of timeout

#### 7.1.5. Cancellation Via `Future`

- `ExecutorService#submit` returns `Future`
  - Has `cancel` method with `mayInterruptIfRunning` argument
- When `mayInterruptIfRunning` is `true`, thread is interrupted
- Setting to `false` means "don't run task if it hasn't started yet"
  - Used for tasks that can't handle interruption
- When is `mayInterruptIfRunning` ok?
- Standard `Executor` implementations allow tasks to be cancelled with interruption
- Should not interrupt pool thread directly
  - No idea what task is being executed
  - Only do this through `Future`

##### Listing 7.10. Cancelling a Task Using `Future`

```java
public static void timedrun(Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
  Future<?> task = taskExec.submit(r);
  try {
    task.get(timeout, unit);
  } catch (TimeoutException e) {
    // task will be cancelled below
  } catch (ExecutionException e) {
    // exception thrown in task; rethrow
    throw launderThrowable(e.getCause());
  } finally {
    // Harmless if task already completed
    task.cancel(true); // interrupt if running
  }
}
```

#### 7.1.6. Dealing with Non-interruptible Blocking

- Many library methods respond to interruption by returning early and throwing `InterruptedException`
- If thread is blocked performing synchronous socket I/O or waiting on intrinsic lock, interruption has no effect (other than interruption status)
- Can sometimes unblock thread, but requires knowing why thread is blocked
- Synchronous socket I/O in java.io
  - `read` and `write` don't respond to interruption
  - Closing underlying socket makes `read`/`write` throw `SocketException`
- Synchronous socket I/O in java.nio
  - Interrupting thread waiting on `InterruptibleChannel` throws `ClosedByInterruptException`
  - Closing `InterruptibleChannel` causes blocked threads to throws `AsynchronousCloseException`
- Asynchronous I/O with selector
  - `close` or `wakeup` causes premature return
- Lock acquisition
  - Nothing to do for intrinsic lock except ensuring lock is eventually held
  - Explicit `Lock` class offer `lockInterruptibly`

##### Listing 7.11. Encapsulating Nonstandard Cancellation in a `Thread` by Overriding `Interrupt`

 ```java
public class ReaderThread extends Thread {
  private final Socket socket;
  private final InputStream in;

  public ReaderThread(Socket socket) throws IOException {
    this.socket = socket;
    this.in = socket.getInputStream();
  }

  public void interrupt() {
    try {
      socket.close();
    } catch (IOException ignored) { }
    finally {
      super.interrupt();
    }
  }

  public void run() {
    try {
      byte[] buf = new byte[BUFSZ];
      while(true) {
        int count = in.read(buf);
        if (count < 0) {
          break;
        } else if (count > 0) {
          processBuffer(buf, count);
        }
      }
    } catch (IOException e) {
      // Allow thread to exit
    }
  }
}
 ```

- Manages single socket connection
- `ReaderThread` overrides `interrupt` to deliver interrupt and close underlying socket

#### 7.1.7. Encapsulating Nonstandard Cancellation with `newTaskFor`

- Hook added to `ThreadPoolExecutor`
- When `Callable` is submitted, `newTaskFor` factory method creates the `Future` representing the task
- Can override `Future#cancel`
  - Perform logging or gather statistics on cancellation
  - Cancel activities that are not responsive to interruption

##### Listing 7.12. Encapsulating Nonstandard Cancellation in a Task with `newTaskFor`

```java
public interface CancellableTask<T> extends Callable<T> {
  void cancel();
  RunnableFuture<T> newTask();
}

@ThreadSafe
public class CancellingExecutor extends ThreadPoolExecutor {
  ...
  protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    if(callable instanceof CancellableTask) {
      return (CancellableTask<T> callable).newTask();
    } else {
      return super.newTaskFor(callable);
    }
  }
}

pubilc abstract class SocketUsingTask<T> implements CancellableTask<T> {
  @GuardedBy("this") private Socket socket;

  protected synchronized void setsocket(Socket s) {
    socket = s;
  }

  public synchronized void cancel() {
    try {
      if (socket != null) {
        socket.close();
      }
    } catch (IOException ignored) { }
  }
}

public RunnableFuture<T> newTask() {
  return new FutureTask<T>(this) {
    public boolean cancel(boolean mayInterruptIfRunning) {
      try {
        SocketUsingTask.this.cancel();
      } finally {
        return super.cancel(mayInterruptIfRunning);
      }
    }
  }
}
```

- If `SocketUsingTask` is cancelled through `Future`, socket is closed and executing thread is interrupted
  - Can safely call interruptible blocking methods, but remain responsive to cancellation
  - Can also call blocking socket I/O methods

### 7.2. Stopping a Thread-Based Service

- Lifetime of thread pools is usually longer than calling method
- Threads need to be terminated gracefully
- No formalized owner of thread
  - Makes sense to ascribe it to class that created it
- Thread ownership is not transitive
- Service should provide lifecycle methods for shutting down
  - Callers can shutdown service, service can shutdown threads
- `ExecutorService` provides `shutdown` and `shutdownNow` methods

#### 7.2.1. Example: A Logging Service

- Inline logging can have performance costs in high volume applications
- Can queue log message for separate thread

##### Listing 7.13. Producer-Consumer Logging Service with No Shutdown Support

```java
public class LogWriter {
  private final BlockingQueue<String> queue;
  private final LoggerThread logger;

  public LogWriter(Writer writer) {
    this.queue = new LinkedBlockingQueue<String>(CAPACITY);
    this.logger = new LoggerThread(writer);
  }

  public void start() {
    logger.start();
  }

  public void log(String msg) throws InterruptedException {
    queue.put(msg);
  }

  private class LoggerThread extends Thread {
    private final PrintWriter writer;
    ...
    public void run() {
      try {
        while (true) {
          writer.println(queue.take());
        } catch (InterruptedException ignored) { }
        finally {
          writer.close();
        }
      }
    }
  }
}
```

- Need to be able to terminate logging thread so it doesn't prevent JVM exit
- If logger thread exits on `InterruptedException`, then interrupting logger thread stops service
  - Would lose log messages
  - Threads blocked in `log` because queue is full will never become unblocked
- Need to cancel both producers and consumers
- Could use flag to mean shutdown
  - Consumer would drain queue and unblock any blocked producers
  - Vulnerable to race conditions

##### Listing 7.14. Unreliable Way to Add Shutdown Support to Logging Service

```java
public void log(String msg) throws InterruptedException {
  if (!shutdownRequested) {
    queue.put(msg);
  } else {
    throw new IllegalStateException("logger is shut down");
  }
}
```

- Instead, need to make adding a log message atomic
  - Without locking
  - Atomically check for shutdown and conditionally increment counter to reserve right to submit message

##### Listing 7.15. Adding Reliable Cancellation to `LogWriter`

```java
public class LogService {
  private final BlockingQueue<String> queue;
  private final LoggerThread loggerThread;
  private final PrintWriter writer;
  @GuardedBy("this") private boolean isShutdown;
  @GuardedBy("this") private int reservations;

  public void start() {
    loggerThread.start();
  }

  public void stop() {
    synchronized (this) {
      isShutdown = true;
    }
    loggerThread.interrupt();
  }

  public void log(String msg) throws InterruptedException {
    synchronized (this) {
      if(isShutdown) {
        throw new IllegalStateException(...);
      }
    }

    queue.put(msg);
  }

  private class LoggerThread extends Thread {
    public void run() {
      try {
        while (true) {
          try {
            synchronized (LogService.this) {
              if (isShutdown && reservations == 0) {
                break;
              }
            }
            String msg = queue.take;
            synchronized (LogService.this) {
              --reservations;
            }
            writer.println(msg);
          } catch (InterruptedException e) {
            // retry
          }
        }
      } finally {
        writer.close();
      }
    }
  }
}
```

#### 7.2.2. `ExecutorService` Shutdown

- `ExecutorService#shutdownNow` returns list of tasks that had not yet started
- Abrupt termination is faster but riskier because tasks may be interrupted in middle of execution
- Normal termination is slower but safer because all queued tasks are processed
- Simple programs can use global `ExecutorService` on `main` thread
- May need to encapsulate in higher-level service with lifecycle methods

### Listing 7.16. Logging Service that Uses an `ExecutorService`

```java
public class LogService {
  private final ExecutorService exec = new SingleThreadExecutor();
  ...
  public void start() { }

  public void stop() throws InterruptedException {
    try {
      exec.shutdown();
      exec.awaitTermination(TIMEOUT, UNIT);
    }
  } finally {
    writer.close();
  }
}

public void log(String msg) {
  try {
    exec.execute(new WriteTask(msg));
  } catch (REjectedExecutionException ignored) { }
}
```

#### 7.2.3. Poison Pills

- Poison pill is recognizable object that means "stop
- With FIFO queue, ensures consumers finish before shutting down
  - Producer should not submit more work after enqueueing poison pill
- Only work when number of producers and consumers is known
  - And only work reliably with unbounded queues

##### Listing 7.17. Shutdown with Poison Pill

```java
public class IndexingService {
  private static final File POISON = new File("");
  private final IndexerThread consumer = new IndexerThread();
  private final CrawlerThread producer = new CrawlerThread();
  private final BlockingQueue<File> queue;
  private final FileFilter fileFilter;
  private final File root;

  public void start() {
    producer.start();
    consumer.start();
  }

  public void stop() {
    producer.interrupt();
  }

  public void awaitTermination() throws InterruptedException {
    consumer.join();
  }
}
```

##### Listing 7.18

```java
public class CrawlerThread extends Thread {
  public void run() {
    try {
      crawl(root);
    } catch (InterruptedException e) {
      // fall through
    } finally {
      while (true) {
        try {
          queue.put(POISON);
          break;
        } catch (InterruptedException e1) {
          // retry
        }
      }
    }
  }

  private void crawl(File root) throws InterruptedException {
    ...
  }
}
```

##### Listing 7.19

```java
public class IndexerThread extends Thread {
  public void run() {
    try {
      while(true) {
        File file = queue.take();
        if (file == POISON) {
          break;
        } else {
          indexFile(file);
        }
      }
    } catch (InterruptedException consumed) { }
  }
}
```

#### 7.2.4. Example: A One-shot Execution Service

#### 7.2.5. Limitations of `shutdownNow`

### 7.3. Handling Abnormal Thread Termination

#### 7.3.1. Uncaught Exception Handlers

### 7.4. JVM Shutdown

#### 7.4.1. Shutdown Hooks

#### 7.4.2. Daemon Threads

#### 7.4.3. Finalizers

## Chapter 8. Applying Thread Pools

## Chapter 9. GUI Applications

## Chapter 10. Avoiding Liveness Hazards

## Chapter 11. Performance and Scalability

## Chapter 12. Testing Concurrent Programs

## Chapter 13. Explicit Locks

## Chapter 14. Building Custom Synchronizers

## Chapter 15. Atomic Variables and Nonblocking Synchronization

## Chapter 16. The Java Memory Model
  - Guarded - Access only with specific lock. Include objects encapsulated in other thread-safe objects and published objects that are guarded by specific lock
  ## Chapter 4. Composing Objects

  ### 4.1 Designing a Thread-Safe Class
  - Encapsulation makes it possible to determine that a class is thread-safe without reading entire program
  - Designing a thread-safe class includes three elements
    - Identify variables that form object state
    - Identify invariants that constraint state variables
    - Establish policy for managing concurrent access to object state
  - Object state starts with fields
    - If all primitive, fields are entire state
    - If object has references to objects, state encompasses fields from referenced objects
  
##### Listing 4.1 Simple Thread-safe Counter Using the Java Monitor Pattern
```java
@ThreadSafe
public final class Counter {
  @GuardedBy("this") private long value = 0;

  public synchronized long getValue() {
    return value;
  }

  public synchronized long increment() [
    if (value == Long.MAX_VALUE) {
      throw new IllegalStateException("counter overflow");
    }

    return ++value;
  ]
}
```

- Synchronization policy defines how object coordinates access to state
  - Specifies what combination of immutability, thread confinement, and locking is used to maintain thread safety and which variables are guarded by locks.

#### 4.1.1 Gathering Synchronization Requirements
- Making class thread-safe means ensuring invariants hold under concurrent access
- Objects and variables have state space
    - Range of possible values
- Smaller state space is easier to reason about
- By using final fields wherever practical, simpler to analyze possible states
  - Immutable objects can only be in single state
- Classes have invariants that identify states as valid or invalid
- The `value` in `Counter` is `long`
  - `long` can only have values from `Long.MIN_VALUE` to `Long.MAX_VALUE`
  - `Counter` places additional positive constraint
- Operations may have postconditions that identify invalid state transitions
  - If current state of `Counter` is `17`, only `18` is valid next state
- When next state is derived from current state, operation is compound action
- Constraints placed on state or state transition create synchronization or encapsulation requirements
- Class can have invariants that constraint multiple state variables
- `NumberRange` in Listing 4.10 maintains state variables for lower and upper bounds
  - Must obey constraint that lower bound is less than or equal to upper bound
- Multivariable invariants create atomicity requirements
  - Object may be in invalid state between acquiring locks

#### 4.1.2 State-dependent Operations
- Some objects have methods with state-based preconditions
  - Cannot remove item from empty queue
- Called _state-dependent_ operations
- In a single-threaded program, no other choice but to fail
- In concurrent program, precondition may become true later
  - Add possiblity of waiting until precondition is true
- Built-in mechanisms for waiting are tightly bound to intrinsic locking, difficult to use correctly
  - `wait` and `notify`
- Easier to use existing library classes
  - Blocking queues or semaphores

#### 4.1.3 State Ownership
- When defining object state, only consider the data the object owns
- Not embedded in language, element of class design
- When allocating/populating `HashMap`
  - Allocate a `HashMap` object and several `Map.Entry` objects
  - State of `HashMap` includes `HashMap` and all `Map.Entry` objects
- Ownership and encapsulation go together
  - Object enapsulates state and owns state it encapsulates
- Ownership means deciding locking protocol
  - Publishing a reference loses that control
- Collection classes are a form of "split ownership"
  - Collection owns state of infrastructure, client owns state of actual objects

#### 4.2 Instance Confinement
- If object is not thread-safe, techniques to let it be used safely
  - Ensure it is accessed from single thread
  - Ensure all access is guarded by lock
- Encapsulation promotes instance confinement
- All code paths have have access are known and can be analyzed more easily
- Combined with appropriate locking, can be used in thread-safe manner
- Encapsulation means data is always accessed with appropriate lock
- Confined objects must not escape intended scope
  - Class instance
  - Lexical scope (local variable)
  - Thread

##### Listing 4.2 Using Confinement to Ensure Thread Safety
```java
@ThreadSafe
public class PersonSet {
  @GuardedBy("this")
  private final Set<Person> mySet = new HashSet<>();

  public synchronized void addPerson(Person p) {
    mySet.add(p);
  }

  public synchronized boolean containsPerson(Person p) {
    return mySet.contains(p);
  }
}
```

- `PersonSet` is managed by `HashSet` (thread-unsafe)
  - `HashSet` is private and not allowed to escape
  - Access to `mySet` is guarded by locks
  - `PersonSet` is thread-safe
- Example makes no assumptions about thread-safety of `Person`
  - Most reliable is making `Person` thread-safe
  - Less reliable is to guard `Person` with locks
- Instance confinement allows choice in locking strategy
- Different state variables can be guarded by different locks
- Java synchronized `Collections` methods provide a synchronized wrapper around thread-unsafe classes
  - As long as only reference is wrapped, access is thread-safe

#### 4.2.1 The Java Monitor Pattern
- Object encapsulates all its mutable state and guards it with own intrinsic lock
- Any lock object can be used as long as use is consistent

##### Listing 4.3 Guarding State with a Private Lock
```java
public class PrivateLock {
  private final Object myLock = new Object();
  @GuardedBy("myLock") Widget widget;

  void someMethod() {
    synchronized(myLock) {
      // Access or modify state of widget
    }
  }
}
```

- Using a private lock means client code cannot acquire it
  - Clients that improperly acquire locks can cause liveness problems

#### 4.2.2 Example: Tracking Fleet Vehicles
- Build a vehicle tracker for dispatching fleet vehicles
  - Taxis, police cars, delivery trucks

```java
Map<String, Point> locations = vehicles.getLocations();
for (String key : locations.keyset()) {
  renderVehicle(key, locations.get(key));
}
```

```java
void vehicleMoved(VehicleMovedEvent event) {
  Point location = event.getNewLocation();
  vehicles.setLocation(event.getVehicleId(), location.x, location.y);
}
```

- Data model must be thread-safe because GUI thread and updater thread will access concurrently

##### 4.4 Monitor-based Vehicle Tracker Implementation
```java
@ThreadSafe
public class MonitorVehicleTracker {
  @GuardedBy("this")
  private final Map<String, MutablePoint> locations;

  public MonitorVehicleTracker(Map<String, MutablePoint> locations) {
    this.locations = deepCopy(locations);
  }

  public synchronized Map<String, MutablePoint> getLocations() {
    return deepCopy(locations);
  }

  public synchronized MutablePoint getLocation(String id) {
    MutablePoint location = locations.get(id);
    return loc == null ? null : new MutablePoint(location);
  }

  public synchronized void setLocation(String id) {
    MutablePoint location = locations.get(id);
    if(location == null) {
      throw new IllegalArgumentException("No such ID: " + id);
    }

    location.x = x;
    location.y = y;
  }

  private static Map<String, MutablePoint> deepCopy(Map<String, MutablePoint> map) {
    Map<String, MutablePoint> result = new HashMap<>();
    for (String id : map.keySet()) {
      result.put(id, new MutablePoint(m.get(id)));
    }

    return Collections.unmodifiableMap(result);
  }
}
```

##### Listing 4.5. Mutable Point Class Similar to `Java.awt.Point`
```java
@NotThreadSafe
public class MutablePoint {
  public int x, y;

  public MutablePoint() {
    x = 0;
    y = 0;
  }

  public MutablePoint(MutablePoint point) {
    this.x = point.x;
    this.y = point.y;
  }
}
```

- `MutablePoint` is not thread-safe, but tracker class is
- `MutablePoint`s are never published
- Copying mutable data may be performance issue at-scale

### 4.3 Delegating Thread Safety
- Sometimes need to add additional layer of safety, even if components are already thread-safe
- `CountingFactorizer` was thread-safe after adding an `AtomicLong`
  - Entire state of `CountingFactorizer` in `AtomicLong` and no additional validity requirements
- `CountingFactorizer` _delegates_ thread safety to `AtomicLong`

#### 4.3.1. Example: Vehicle Tracker Using Delegation
- Construct the vehicle tracker with thread-safe `Map`
  - `ConcurrentHashMap`
- Also store location in immutable `Point` class

##### 4.6 Immutable `Point` class used by `DelegatingVehicleTracker`
```java
@Immutable
public class Point {
  public final int x, y;

  public Point(int x, int y) {
    this.x = x;
    this.y = y;
  }
}
```

##### 4.7 Delegating Thread Safety to a `ConcurrentHashMap`
```java
@ThreadSafe
public class DelegatingVehicleTracker {
  private final ConcurrentMap<String, Point> locations;
  private final Map<String, Point>  unmodifiableMap;

  public DelegatingVehicleTracker(Map<String, Point> points) {
    locations = new ConcurrentHashMap<>(points);
    unmodifiableMap = Collections.unmodifiableMap(locations);
  }

  public Map<String, Point> getLocations() {
    return unmodifiableMap;
  }

  public Point getLocation(String id) {
    return locations.get(id);
  }

  public void setLocation(String id, int x, int y) {
    if(locations.replace(id, new Point(x, y)) == null) {
      throw new IllegalArgumentException("invalid vehicle name: " + id);
    }
  }
}
```

- Using `MutablePoint` would break encapsulation
  - `getLocations` would publish reference to (thread-unsafe) mutable state
- `getLocations` returns a live view of the map
  - Could be a benefit or a liability

##### Listing 4.8. Returning a Static Copy of the Location Set Instead of a "Live" One
```java
public Map<String, Point> getLocations() {
  return Collections.unmodifiableMap(new HashMap<>(locations));
}
```

#### 4.3.2. Independent State Variables
- Can also delegate thread safety to multiple state variables
  - As long as they are independent
  - Enclosing class does not impose invariants involving multiple state variables

##### Listing 4.9. Delegating Thread Safety to Multiple Underlying State Variables
```java
public class VisualComponent {
  private final List<KeyListener> keyListeners = new CopyOnWriteArrayList<>();
  private final List<MouseListener> mouseListeners = new CopyOnWriteArrayList<>();

  public void addKeyListener(KeyListener listener) {
    keyListeners.add(listener);
  }

  public void addMouseListener(MouseListener listener) {
    mouseListeners.add(listener);
  }

  public void removeKeyListener(KeyListener listener) {
    keyListeners.remove(listener);
  }

  public void removeMouseListener(MouseListener listener) {
    mouseListeners.remove(listener);
  }
}
```

- `CopyOnWriteArrayList` is a thread-safe `List` implementation
- `VisualComponent` delegates thread safety to underlying `mouseListeners` and `keyListeners`

#### 4.3.3. When Delegation Fails
- Most classes are not as simple

##### Listing 4.10. Number Range Class that does Not Sufficiently Protect Its Invariants. _Don't do this._
```java
public class NumberRange {
  // INVARIANT: lower <= upper
  private final AtomicInteger lower = new AtomicInteger(0);
  private final AtomicInteger upper = new AtomicInteger(0);

  public void setLower(int i) {
    // Warning -- unsafe check-then-act
    if (i > upper.get()) {
      throw new IllegalArgumentException("can't set lower to " + i + " > upper");
   }
   lower.set(i);
 }

 public void setUpper(int i) {
   // Warning -- unsafe check-then-act
   if (i < lower.get()) {
     throw new IllegalArgumentException("can't set upper to " + i + " < lower");
   }
   upper.set(i);
 }

 public boolean isInRange(int i) {
   return (i >= lower.get() && i <= upper.get());
 }
}
```

- `NumberRange` is not thread-safe
  - Does not preserve invariant that constrains `lower` and `upper`
- Could be made thread-safe by using locking
  - Guard `lower` and `upper` with common lock
- Must also avoid publishing `lower` and `upper`

#### 4.3.4. Publishing underlying state variables
- When can you publish state variables when delegating thread-safety to them?
  - It depends
- Publishing state variables means other classes can change them
  - May violate invariants
- If state variable is thread-safe, does not participate in invariants, and has no prohibited state transitions, it can be published

#### 4.3.5. Example: Vehicle Tracker that Publishes Its State
- Another version of vehicle tracker that publishes mutable state

##### Listing 4.11 Thread-safe Mutable Point Class
```java
@ThreadSafe
public class SafePoint {
  @GuardedBy("this") private int x, y;

  private SafePoint(int[] a) {
    this(a[0], a[1]);
  }

  public SafePoint(SafePoint safePoint) {
    this(safePoint.get());
  }

  public SafePoint(int x, int y) {
    this.x = x;
    this.y = y;
  }

  public synchronized int[] get() {
    return new int[] {x, y};
  }

  public synchronized void set(int x, int y) {
    this.x = x;
    this.y = y;
  }
}
```

- Provides getter that retrieves both `x` and `y` values at same time
  - If separate getters, values could change between calls

##### Listing 4.12 Vehicle Tracker That Safely Publishes Underlying State
```java
@ThreadSafe
public class PublishingVehicleTracker {
  private final Map<String, SafePoint> locations;
  private final Map<String, SafePoint> unmodifiableMap;

  public PublishingVehicleTracker(Map<String, SafePoint> locations) {
    this.locations = new ConcurrentHashMap<>(locations);
    this.unmodifiableMap = Collections.unmodifiableMap(this.locations);
  }

  public Map<String, SafePoint> getLocations() {
    return unmodifiableMap;
  }

  public SafePoint getLocation(String id) {
    return locations.get(id);
  }

  public void setLocation(String id, int x, int y) {
    if (!locations.containsKey(id)) {
      throw new IllegalArgumentException("invalid vehicle name: " + id);
    }
    locations.get(id).set(x, y);
  }
}
```

- Derives thread safety from delegation to `ConcurrentHashMap`
  - Contents are thread-safe mutable points
- Callers cannot add/remove vehicles, but can modify vehicle locations
  - Whether or not this is desired depends on the requirements

### 4.4. Adding Functionality To Existing Thread-Safe Classes
- Sometimes need to add new operations without undermining thread safety
- E.g. Adding an atomic put-if-absent method for a `List`
- May not always have access to underlying source
  - Also need to understand original implementation's synchronization policy
- Another approach is to extend the class
  - Fragile because synchronization policy now lives in multiple classes

##### Listing 4.13. Extending `Vector to have a `put-if-absent` method
 ```java
@ThreadSafe
public class BetterVector<E> extends Vector<E> {
  public synchronized boolean putIfAbsent(E x) {
    boolean absent = !contains(x);
    if(absent) {
      add(x);
    }
    return absent;
  }
}
 ```

#### 4.4.1. Client-side Locking
- For `ArrayList` wrapped with `Collections.synchronizedList`, neither approach works
- Third strategy is to extend the functionality without extending the class
  - Helper class

##### Listing 4.14. Non-thread-safe Attempt to Implement a put-if-absent. _Don't do this_
```java
@NotThreadSafe
public class ListHelper<E> {
  public List<E> list = Collections.synchronizedList(new ArrayList<>());

  public synchronized boolean putIfAbsent(E x) {
    boolean absent = !list.contains(x);
    if (absent) {
      list.add(x);
    }
    return absent;
  }
}
```
- Won't work because synchronizes on the wrong lock
- No guarantee another thread won't modify the list while `putIfAbsent` is executing
- `Vector` documentation uses intrinsic lock

##### Listing 4.15. Implementing put-if-absent with Client-Side Locking
```java
@ThreadSafe
public class ListHelper<E> {
  public List<E> list = Collections.synchronizedList(new ArrayList<>());

  public boolean putIfAbsent(E x) {
    synchronized(list) {
      boolean absent = !list.contains(x);
      if (absent) {
        list.add(x);
      }
      return absent;
    }
  }
}
```

- Even more fragile than extending a class
  - Puts locking code for class `C` into classes that are not `C`

#### 4.4.2. Composition
- Less fragile alternative

##### Listing 4.16. Implementing put-if-absent Using Composition
```java
@ThreadSafe
public class ImprovedList<T> implements List<T> {
  private final List<T> list;

  public ImprovedList(List<T> list) {
    this.list = list;
  }

  public synchronized boolean putIfAbsent(T x) {
    boolean contains = list.contains(x);
    if (contains) {
      list.add(x);
    }
    return !contains;
  }

  public synchronized void clear() {
    list.clear();
  }
  //...delegate to other List methods
}
```

- `ImprovedList` provides its own consistent locking
- Less fragile than attempting to mimic other object's locking strategy
- Guaranteed thread safety as long as our class holds only reference to underlying `List`

### 4.5. Documenting Synchronization Policies
- Users look to documentation to see if class is thread-safe
  - Document thread-safety
- Maintainers look to understand implementation strategy
  - Document synchronization policy
- Use of `synchronized`, `volatile`, or thread-safe class reflects synchronization policy
- Current documentation is lacking
  - `java.text.SimpleDateFormat` is not thread-safe, but not documented as such until JDK 1.4
-

#### 4.5.1. Interpreting Vague Documentation
- Need to guess based on how it will be implemented
- Servlets and database connections are frequently run in threads
  - Implementations should expect to need to deal with concurrent access

## Chapter 5. Building Blocks

### 5.1. Synchronized Collections
- Include `Vector` and `Hashtable`
  - Along with `Collections.synchronizedXxx` factory methods

#### 5.1.1. Problems with Synchronized Collections
- Thread-safe but may need additional locking for compound actions
  - Iteration, navigation, conditional operation (put-if-absent)
- Technically thread-safe, but may not behave as expected under concurrent access

##### Listing 5.1. Compound Actions on a `Vector` that may produce confusing results
```java
public static Object getLast(Vector list) {
  int lastIndex = list.size() - 1;
  return list.get(lastIndex);
}

public static void deleteLast(Vector list) {
  int lastIndex = list.size() - 1;
  list.remove(lastIndex);
}
```
- Operations can be interleaved
- Synchronized collections use implementation that allows client-side locking
  - Possible to create new operations that are atomic
  - Can acquire lock on synchronized collection object
- Size of list and corresponding get also affects iteration

##### Listing 5.2. Compound actions on `Vector` using client-side locking
```java
public static Object getLast(Vector list) {
  synchronized (list) {
    int lastIndex = list.size() - 1;
    return list.get(lastIndex);
  }
}

public static void deleteLast(Vector list) {
  synchronized(list) {
    int lastIndex = list.size() - 1;
    list.remove(lastIndex);
  }
}
```

##### Listing 5.3. Iteration that may throw `ArrayIndexOutOfBoundsException`
```java
for (int i = 0; i < vector.size(); i++) {
  doSomething(vector.get(i));
}
```

#### 5.1.2. Iterators and `ConcurrentModificationException`
- Standard way of iterating through a `Collection` is `Iterator`
  - Explicitly or with `for-each` loop
- Iterators returned by synchronized collections not designed to deal with concurrent modification
  - Fail-fast if collection has changed since iteration began
  - Throw unchecked `ConcurrentModificationException`
- Designed to catch errors on "good-faith-effort"
- Implemented by associating modification count with collection
  - If modification count changes during iteration, `hasNext` or `next` throws `ConcurrentModificationException`
  - Done without synchronization

##### Listing 5.5. Iterating a `List` with an `Iterator`
```java
List<Widget> widgetList = collections.synchronizedList(new ArrayList<>());
// May throw ConcurrentModificationException
for (Widget w : widgetList) {
  doSomething(w);
}
```

- Locking collection during iteration may be undesirable
- Other threads will block on access
  - Might wait a long time
- Risk factor for dead lock
- Even if it works, hurts application scalability
  - Lock contention leads to lower throughput and CPU utilization
- Alternative is to clone collection and iterate the copy
  - Thread-confined
  - Eliminates possibility of `ConcurrentModificationException`
  - Original collection must still be locked for clone
- Cloning still has performance cost

#### 5.1.3. Hidden Iterators
- Have to remember to use locking everywhere a shared collection might be iterated

#####  Listing 5.6. Iteration hidden within string concatenation. _Don't do this_
```java
public class HiddenIterator {
  @GuardedBy("this")
  private final Set<Integef> set = new HashSet<>();

  public synchronized void add(Integer i) {
    set.add(i);
  }

  public synchronized void remove(Integer i) {
    set.remove(i);
  }

  public void addTenThings() {
    Random r = new Random();
    for (int i = 0; i < 10; i++) {
      add(r.nextInt());
    }
    System.out.println("DEBUG: added ten elements to " + set);
  }
}
```

- String concatenation turns into `StringBuilder.append(Object)`, which invokes collection's `toString`, which iterates the collection and calls `toString` on each element
- The greater the distance between state and synchronization, the more likely someone will forget to use proper synchronization
- Iteration also invoked by collection's `hashCode` and `equals` methods
  - `containsAll`, `removeAll`, and `retainAll` methods will iterate over collection parameter

### 5.2 Concurrent Collections
- Java 5 provides several concurrent collection classes
- Synchronized collections serialize all access
  - Poor concurrency
- Also addes `Queue` and `BlockingQueue`
  - `ConcurrentLinkedQueue` and `PriorityQueue`
- Java 6 adds `ConcurrentSkipListMap` and `ConcurrentSkiplistSet`
  - Replace synchronized `SortedMap` and `SortedSet`

#### 5.2.1. ConcurrentHashMap
- Synchronized collections hold lock for duration of operation
  - Iterating collection might be necessary for some calls
- `ConcurrentHashMap` uses a more performant locking strategy
  - Lock striping
  - Arbitrary number of reading threads
  - Reads don't compete with writes
  - Limited number of writers
  - Higher throughput for concurrent access, little performance penalty for single-threaded access
- Concurrent collections provide iterators that do not throw `ConcurrentModificationException`
- Iterators are weakly-consistent
  - Tolerant of concurrent modification
  - Traverses elements as existed when iterator was constructed
  - May (but not guaranteed) relfect modifications after construction of iterator
- Semantics of methods that operate on entire map (`size`, `isEmpty`) are slightly weakened
  - Could be out of date
  - Generally not useful in concurrent environments, so weakness is tolerable
- Synchronized collections allow exclusive access
  - Only real tradeoff

#### 5.2.2. Additional Atomic Map Operations
- `ConcurrentMap` offers put-if-absent, remove-if-equal, and replace-if-equal atomic eoperations

##### Listing 5.7. `ConcurrentMap` interface
```java
public interface ConcurrentMap<K, V> extends Map<K, V> {
  // Insert into map only if no value is mapped from K
  V putIfAbsent(K key, V value);

  // Remove only if K is mapped to V
  boolean remove(K key, V value);

  // Replace value only if K is mapped to some value
  V replace(K key, V newValue);
}
```

#### 5.2.3. CopyOnWriteArrayList
- Concurrent replacement for synchronized `List`
- Rely on properly publishing immutable objects
- Create and republish new copy of collection every time it is modified
- Iterators retain reference to backing array from start of iteration
  - Only need to synchronize briefly to ensure visibility of array contents
- Some cose to copying the backing array
  - Reasonable only when iteration is far more common than modification
  - E.g. event-notification systems

### 5.3. Blocking Queues and the Producer-Consumer Pattern
- Blocking queues provide blocking `put` and `take` methods
  - Also timed `offer` and `put`
- If queue is full/empty, thread will block until condition is met
- Can be bounded or unbounded
- Support producer-consumer pattern
  - Separates the identification of work with execution of work
  - Producers place data in queue as it's available
  - Consumers take data when they are ready
- Producers/consumers don't need to know about the other
- Blocking queue simplifies consumers
  - `take` blocks until data is available
  - If producers aren't fast enough, consumers will wait
- If produers generate work faster than consumption, will run out of memory
  - Bounded queue will cause producer to block
- `offer` will return a failure status if item cannot be enqueued
  - Can use alternate strategy (shedding load, write to disk, scale down, throttling)

#### 5.3.1. Example: Desktop Search
- Producer task searches for files meeting index criteria, consumer task takes file names and indexes them

##### Listing 5.7. Producer and Consumer Tasks in a Desktop Search Application
```java
public class FileCrawler implements Runnable {
  private final BlockingQueue<File> fileQueue;
  private final FileFilter fileFilter;
  private final File root;

  public void run() {
    try {
      crawl(root);
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
    }
  }

  private void crawl(File root) throws InterruptedException {
    File[] entries = root.listFiles(fileFilter);
    if (entries != null) {
      for (File entry : entries) {
        if (entry.isDirectory()) {
          crawl(entry);
        } else if(!alreadyIndexed(entry)) {
          fileQueue.put(entry);
        }
      }
    }
  }
}

public class Indexer implements Runnable {
  private final BlockingQueue<File> queue;

  public Indexer(BlockingQueue<File> queue) {
    this.queue = queue;
  }

  public void run() {
    try {
      while (true) {
        indexFile(queue.take());
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }
}
```

##### Listing 5.9. Starting the Desktop Search
```java
public static void startIndexing(File[] roots) {
  BlockingQueue<File> queue = new LinkedBlockingQueue<File>(BOUND);
  FileFilter filter = new FileFilter() {
    public boolean accept(File file) { return true; }
  }

  for (File root : roots) {
    new Thread(new FileCrawler(queue, filter, root)).start();
  }

  for(int i = 0; i < N_CONSUMERS; i++) {
    new Thread(new Indexer(queue)).start();
  }
}
```

#### 5.3.2. Serial Thread Confinement
- Blocking queue implementations contain sufficient internal synchronization to safely publish objects from producer thread to consumer thread
- For mutable objects, facilitate serial thread confinement
  - Owned exclusively by thread, but ownership can be transferred by publishing it
- Object pools lend object to requesting thread
  - As long as pooled object is safely published and clients do not published pooled object or use it after returning, ownership is safely transferred
- Could use other publication methods but necessary to ensure only one thread receives it

#### 5.3.3. Deques and Work Stealing
- Java 6 also adds `Deque` ("deck") and `BlockingDeque`
  - Double-ended queue
- Useful for work stealing
- If consumer exhausts own deque, it steals work from tail of another deque
- Workers don't contend for a shared queue
  - When accessing other deques, read from tail instead of head further reduces contention

### 5.4. Blocking and Interruptible Methods
- Threads may block
  - I/O, lock, wake up from `Thread.sleep`, computation on another thread
- Usually suspended and placed in a blocked thread state
  - `BLOCKED`, `WAITING`, `TIMED_WAITING`
- Blocked thread must wait for an evant beyond its control
  - Then placed in `RUNNABLE` state and is eligible for scheduling
- `put` and `take` throw `InterruptedException`
  - Signals that it blocks, and that if interrupted it will make an effort to stop blocking early
- `Thread` provides `interrupt` for interrupting a thread and querying whether a thread has been interrupted
  - Boolean property that represents interrupted status
- Interruption is cooperative
  - When thread A interrupts thread B, it only requests that B stops when it is convenient
- When handling `InterruptedException`, there are basically 2 choices
  - Propagate the exception
  - Restore the interrupt
    - Must catch exception and call `interrupt` on current thread
    - Code higher in the call stack will see that an interrupt was issued

##### Listing 5.10. Restoring the Interrupted Status so as Not to Swallow the Interrupt
```java
public class TaskRunnable implements Runnable {
  BlockingQueue<Task> queue;

  public void run() {
    try {
      processTask(queue.take());
    } catch (InterruptedException e) {
      // restore interrupted status
      Thread.currentThread().interrupt();
    }
  }
}
```

- Should not swallow `InterruptedException`
  - Code higher in the call stack cannot respond to interruption
  - Only acceptable when extending Thread and control all code higher in the call stack

### 5.5. Synchronizers
- Blocking queues act as containers, and coordinate flow of producer and consumer threads
- Synchronizer is object that coordinates flow of threads based on its state
  - e.g. blocking queues, semaphores, barriers, latches
- Encapsulate state that determines whether threads should pass or wait
- Provide methods to manipulate state
- Provide methods to wait efficiently for synchronizer to enter desired state

#### 5.5.1. Latches
- Delays progress of threads until it reaches terminal state, like a gate
- Until latch reaches terminal state, gate is closed and no thread passes
- In terminal state, latch opens and all threads pass
- Once latch reaches terminal state, it cannot change state
  - Remains open forever
- Useful for one-time activities
  - Resource initialization
  - Dependent services
  - Waiting until all parties in activity are ready
- `CountDownLatch` allows one or more threads to wait for a set of events to occur
  - Latch state is a positive counter
  - `countDown` decrements counter, indicates method occurred
  - `await` waits for counter to reach 0

##### Listing 5.11. Using `CountDownLatch` for Starting and Stopping Threads in Timing Tests
```java
public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
  final CountDownLatch startGate = new CountDownLatch(1);
  final CountDownLatch endGate = new CountDownLatch(nThreads);

  for (int i  = 0; i < nThreads; i++) {
    Thread t = new Thread() {
      public void run() {
        try {
          startGate.await();
          try {
            task.run();
          } finally {
            endGate.countDown();
          }
        } catch (InterruptedException ignored) {}
      }
    };
    t.start();
  }

  long start = System.nanoTime();
  startGate.countDown();
  endGate.await();
  long end = System.nanoTime();
  return end-start;
}
```

- `TestHarness` creates threads to run concurrently
- Uses a starting gate and an ending gate
- Each thread waits on starting gate, then counts down end gate
- Master thread wiats until last worker has finished, so it can calculate elapsed time

#### 5.5.2. FutureTask
- Acts like a latch
- Implemented with a `Callable`
  - 3 states: waiting to run, running, completed
- When `FutureTask` enters completed state, stays that way forever
- `Future.get` dpends on state of task
  - If completed, returns result immediately
  - Otherwise blocks until task transitions to completed and returns result or throws exception
- Conveys result from thread executing to thread retrieving result
  - Guarantees that transfer is a safe publication
- Used by `Executor` to represent asynchronus tasks
- Can be used to represent lengthy computation that is started before results are needed

##### Listing 5.12. Using `FutureTask` to Preload Data that is Needed Later
```java
public class Preloader {
  private final FutureTask<ProductInfo> future = new FutureTask<ProductInfo>(new Callables<ProductInfo>() {
    public ProductInfo call() throws DataLoadException {
      return loadProductInfo();
    }
  });
}
private final Thread thread = new Thread(future);

public void start() { thread.start(); }

public ProductInfo get() throws DataLoadException, InterruptedException {
  try {
    return future.get();
  } catch (ExecutionException e) {
    Throwable cause = e.getCause();
    if (cause instanceof DataLoadException) {
      throw (DataLoadException) cause;
    } else {
      throw launderThrowable(cause);
    }
  }
}
```

- `Preloader` creates a `FutureTask` that loads product information from database
- Provides `start` method to start thread
  - Starting thread from constructor is indavisable
- `Callable` can throw checked and unchecked exceptions
  - Wrapped in `ExecutionException` and rethrown from `Future.get`
- Call site must deal with `ExecutionException`, and type is `Throwable`
- One of three categories
  - Checked exception thrown from `Callable`
  - `RuntimeException`
  - `Error`
- `launderThrowable` hanldes messy exception handling
- Test for known checked exceptions before dealing with others
  - Rethrow `Error` as-is
  - Return `RuntimeException`, which is generally rethrown
  - Throw `IllegalStateException` for non-`RuntimeException`s

##### Listing 5.13. Coercing an Unchecked `Throwable` to a `RuntimeException`
```java
public static RuntimeException launderThrowable(Throwable t) {
  if (t instanceof RuntimeException) {
    return (RuntimeException) t;
  } else if (t instanceof Error) {
    throw (Error) t;
  } else {
    throw new IllegalStateException("Not unchecked", t);
  }
}
```

#### 5.5.2. Semaphores
- Counting semaphores control number of activities that access resource or perform action at the same time
- Can be used to implement resource pools or impose bound
- `Semaphore` manages set of permits
  - Initial number is passed to constructor
- Activities acquire permits (if there are any available) and release them
  -  `acquire` blocks until a permit is available, or until interrupted, or until operation times out
  - `release` returns permit
- Degenerate case of binary semaphore
  - Can be used as a mutex
- Useful for resource pools
  - e.g. database connection pools
- Can also use to turn any collection into a bounded blocking collection

##### Listing 5.14. Using `Semaphore` to Bound a Collection
```java
public class BoundedHashSet<T> {
  private final Set<T> set;
  private final Semaphore sem;

  public BoundedHashSet(int bound) {
    this.set = Collections.synchronizedSet(new HashSet<>());
    sem = new Semaphore(bound);
  }

  public boolean add(T o) throws InterruptedException {
    sem.acquire();
    boolean wasAdded = false;
    try {
      wasAdded = set.add(o);
      return wasAdded;
    } finally {
      if(!wasAdded) {
        sem.release();
      }
    }
  }

  public boolean remove(Object o) {
    boolean wasRemoved = set.remove(o);
    if (wasRemoved) {
      sem.release();
    }
    return wasRemoved;
  }
}
```

#### 5.5.4. Barriers
- Latches are single-use
- Barriers are similar to latches
  - Block a group of threads until some event has occurred
- All threads must come together at the same time
- Latches are for waiting for an event, barriers are for waiting for threads
- `CyclicBarrier` allows fixed number of parties to rendevous repeatedly
  - Useful in parallel iterative algorithms that break down subproblems
- Threads call `await` and block until all threads have reached the barrier
- If `await` times out or a thread is interrupted, barrier is broken
  - All outstanding calls terminate with `BrokenBarrierException`
- If `await` is successful, returns a unique arrival index for each thread
  - Can be used to elect leader
- Can pass `Runnable` to barrier constructor
  - Runs when barrier is successfully passed, but before blocked threads are released
- Useful for simulations
  - Each step must complete before next step begins

##### Listing 5.15. Coordinating Computation in a Cellular Automaton with `CyclicBarrier`
```java
public class CellularAutomata {
  private final Board mainBoard;
  private final CyclicBarrier barrier;
  private final Worker[] workers;

  public CellularAutomata(Board board) {
    this.mainBoard = board;
    int count = Runtime.getRuntime().availableProcessors();
    this.barrier = new CyclicBarrier(
      count,
      new Runnable() {
        public void run() {
          mainBoard.commitNewValues();
        }
      }
    );
    this.workers = new Worker[count];
    for (int i = 0; i < count; i++) {
      workers[i] = new Worker(mainBoard.getSubBoard(count, i));
    }
  }

  private class Worker implements Runnable {
    private final Board board;

    public Worker(Board board) { this.board = board; }
    public void run() {
      while(!board.hasConverged()) {
        for (int x = 0; x < board.getMaxX(); x++) {
          for (int y = 0; y < board.getMaxY(); y++) {
            board.setNewValue(x, y, computeValue(x, y));
          }
        }
        try {
          barrier.await();
        } catch (InterruptedException ex) {
          return;
        } catch (BrokenBarrierException ex) {
          return;
        }
      }
    }
  }

  public void start() {
    for (int i = 0; i < workers.length; i++) {
      new Thread(workers[i]).start();
    }
    mainBoard.waitForConvergence();
  }
}
```

- `Exchanger` is a two-party barrier
  - Useful when performing asymetric activities
  - e.g. one thread fills buffer, other thread consumes data from buffer
- Exchange constitues safe publication of both objects to other party
- Timing of exchange depends on responsiveness requirements
  - Exchange when buffer is full and when buffer is empty
  - Exchange when buffer is full, or certain amount of time has elapsed

### 5.6. Building an Efficient, Scalable Result Cache
- Naive cache implementation turns performance bottleneck into scalability bottleneck

##### Listing 5.16. Initial Cache Attempt using `HashMap` and Synchronization
```java
public interface Computable<A, V> {
  V compute(A arg) throws InterruptedException;
}

public class ExpensiveFunction implements Computable<String, BigInteger> {
  public BigInteger compute(String arg) {
    // after deep though
    return new BigInteger(arg);
  }
}

public class Memoizer1<A, V> implements Computable<A, V> {
  @GuardedBy("this")
  private final Map<A, V> cache = new HashMap<>();
  private final Computable<A, V> c;

  public Memoizer1(Computable<A, V> c) {
    this.c = c;
  }

  public synchronized V compute(A arg) throws InterruptedException {
    V result = cache.get(arg);
    if (result == null) {
      result = c.compute(arg);
      cache.put(arg, result);
    }
    return result;
  }
}
```

- `Memoizer1` uses `HashMap` to store previous results
  - Returns result if cached, otherwise computes, stores, and returns
  - `HashMap` is not thread safe, so takes conservative approach of using `synchronized`

##### Listing 5.17. Replacing `HashMap` with `ConcurrentHashMap`
```java
public class Memoizer2<A, V> implements Computable<A, V> {
  private final Map<A, V> cache = new ConcurrentHashMap<>();
  private final Computable<A, V> c;

  public Memoizer2(Computable<A, V> c){ this.c = c; }

  public V compute(A arg) throws InterruptedException {
    V result = cache.get(arg);
    if (result == null) {
      result - c.compute(arg);
      cache.put(arg, result);
    }
    return result;
  }
}
```

- Multiple threads can use `Memoizer2` concurrently
  - Still possible for 2 threads to compute same value at same time
  - For memoization, just slow
  - More generally, safety risk
- Other threads not aware that another thread has already started computation
  - `FutureTask` does exactly this

##### Listing 5.18. Memoizing Wrapper using `FutureTask`
```java
public class Memoizer3<A, V> implements Computable<A, V> {
  private final Map<A, Future<V>> cache = new ConcurrentHashMap<A, Future<V>>();
  private final Computable<A, V> c;

  public Memoizer3(Computable<A, V> c) { this.c = c; }

  public V compute(final A arg) throws InterruptedException {
    Future<V> f = cache.get(arg);
    if (f == null) {
      Callable<V> eval = new Callable<>() {
        public V call() throws InterruptedException {
          return c.compute(arg);
        }
      };
      FutureTask<V> ft = new FutureTask<>(eval);
      f = ft;
      cache.put(arg, ft);
      ft.run();
    }
    try {
      return f.get();
    } catch (ExecutionException e) {
      throw launderThrowable(e.getCause());
    }
  }
}
```

- Good concurrency, result returned efficiently if already computed, and threads wait if computation is already running
- Still a small window when threads might compute same value
  - Need an atomic put-if-absent
- Caching `Future` means cache might be polluted
  - e.g. `Future` is cancelled or fails
  - Should remove cancelled futures (possible failed futures if future attempts would succeed)

##### Listing 5.19. Final Implementation of `Memoizer`
```java
public class Memoizer<A, V> implements Computable<A, V> {
  private final ConcurrentMap<A, Future<V>> cache = new ConcurrentHashMap<>();
  private final Computable<A, V> c;

  public Memoier (Computable<A, V> c) { this.c = c; }

  public V compute(final A arg) throws InterruptedException {
    while (true) {
      Future<V> f = cache.get(arg);
      if (f == null) {
        Callable<V> eval = new Callable<V>() {
          public V call() throws InterruptedException {
            return c.compute(arg);
          }
        };
        FutureTask<V> ft = new FutureTask<>(eval);
        f = cache.putIfAbsent(arg, ft);
        if (f == null) {
          f = ft;
          ft.run();
        }
        try {
          return f.get();
        } catch (CancellationException e) {
          cache.remove(arg, f);
        } catch (ExecutionException e) {
          throw launderThrowable(e.getCause());
        }
      }
    }
  }
}
```

##### Listing 5.20. Factorizing Servlet that Caches Results Using `Memoizer`
```java
@ThreadSafe
public class Factorizer implements Servlet {
  private final Computable<BigInteger, BigInteger[]> c = new Computable<>() {
    public BigInteger[] compute(BigInteger arg) {
      return factor(arg);
    }
  };
  private final Computable<BigInteger, BigInteger[]> cache = new Memoizer<>(c);

  public void service(ServletRequest req,
                      ServletResponse resp) {
    try {
      BigInteger i = extractFromRequest(req);
      encodeIntoResponse(resp, cache.compute(i));
    } catch (InterruptedException e) {
      encodeError(resp, "factorization interrupted");
    }
  }
}
```

## Chapter 6. Task Execution

## Chapter 7. Cancellation and Shutdown

## Chapter 8. Applying Thread Pools

## Chapter 9. GUI Applications

## Chapter 10. Avoiding Liveness Hazards

## Chapter 11. Performance and Scalability

## Chapter 12. Testing Concurrent Programs

## Chapter 13. Explicit Locks

## Chapter 14. Building Custom Synchronizers

## Chapter 15. Atomic Variables and Nonblocking Synchronization

## Chapter 16. The Java Memory Model

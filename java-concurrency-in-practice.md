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

##### Listing 4.14. Non-thread-safe Attempt to Omplement a put-if-absent. _Don't do this_
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

##### Listing 4.15. Implementing put-if-absent with client-side locking
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

##### Listing 4.16. Implementing put-if-absent using composition
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
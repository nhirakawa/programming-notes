# Java Concurrency in Practice
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

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

##### Example
```java
public static Set<Secret> knownSecrets;

public void initialize() {
  knownSecrets = new HashSet<>();
}
```

- Any `Secret` added to the set is also published
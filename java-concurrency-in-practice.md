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
#### Example
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

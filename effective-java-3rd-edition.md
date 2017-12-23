# Effective Java, 3rd Edition

## Chapter 7. Lambdas and Streams
### Item 42: Prefer Lambdas to Anonymous Classes
- Since JDK 1.1, function objects were created with anonymous classes
  - Useful for Strategy pattern
- In Java 8, known as functional interfaces
  - Can be created with lambda expressions

```java
// Anonymous class instance as a function object - obsolete
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});

// Lambda expression as function object (replaces anonymous class)
Collections.sort(words, (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```
- Types of lambda an parameters are not present
  - Compiler deduces from context
  - **Omit type of all lambda parameters unless presence makes program clearer**
  - Also when needed by compiler
- Item 26 says don't use raw types, Item 29 says use generic types, Item 30 says favor generic methods
  - Important for lambdas because compiler obtains type inference information from generics
- Example can be made even shorter by using comparator construction method and using `sort` method added to `List` interface in Java 8

```java
Collections.sort(words, comparingInt(String::length));

words.sort(comparingInt(String::length));
```

- Consider `Operation` enum type
  - Each enum required constant-specific class bodies and override for `apply` method

```java
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) {
            return x + y;
        }
    },

    MINUS("-") {
        public double apply(double x, double y) {
            return x - y;
        }
    },

    TIMES("*") {
        public double apply(double x, double y) {
            return x * y;
        }
    },

    DIVIDE("/") {
        public double apply(double x, double y) {
            return x / y;
        }
    };

    private final String symbol;

    Operation(String symbol) {
        this.symbol = symbol;
    }

    @Override
    public String toString() {
        return symbol;
    }

    public abstract double apply(double x, double y);
}
```

- Item 34 says enum instance fields are preferable to constant-specific class bodies
  - Lambdas make it easier to use former instead of latter

```java
public enum Operation {
    PLUS("+", (x, y) -> x + y),
    MINUS("-", (x, y) -> x - y),
    TIMES("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);

    private final String symbol;
    private final DoubleBinaryOperator op;

    Operation (String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }

    @Override
    public String toString() {
        return symbol;
    }

    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

- **Lambdas lack names and documentation**
  - **If a computation isn't self-explanatory (or exceeds a few lines), don't put it in a lambda**
- One line is ideal, 3 is reasonable maximum
- Arguments passed to enum constructors are evaluated in static context
  - Lambdas in enum constructors cannot access instance members
- Anonymous classes not entirely replaced by lambdas
  - Lambdas limited to functional interfaces
  - Can create instance of abstract class with anonymous class, but not lambda
  - Can use anonymous classes to create instances of interfaces with multiple methods
  - Lambda cannot obtain reference to itself
    - `this` refers to enclosing instance
- Cannot reliably (de)serialize anonymous classes and lambdas
  - **Rarely, if ever, serialize a lambda** (or anonymous class instance)
- **Don't use anonymous classes for function objects unless you have to create instances of types that aren't functional interfaces**
  - Lambdas also open door to functional programming techniques not possible before

### Item 43: Prefer Method References To Lambdas
- Even more succinct than lambdas
- E.g. Maintain a multiset
```java
map.merge(key, 1, (count, incr) -> count + incr);
```
- Straightforward, but `count` and `incr` don't add value
  - `Integer` has static `sum` method that does the same thing

```java
map.merge(key, 1, Integer::sum);
```
- Eliminates boilerplate
- Lambdas can sometimes provide better documentation, even if longer than method reference
- Lambdas and method references are equivalent (mostly, *JLS 9.9-2*)
  - Method references usually result in shorter, cleaner code
- Many method references refer to static methods, but 4 kinds do not

| Type              | Example                  | Lambda Equivalent                                    |
|-------------------|--------------------------|------------------------------------------------------|
| Static            | `Integer::parseInt`      | `str -> Integer.parseInt(str)`                       |
| Bound             | `Instant.now()::isAfter` | `Instant then = Instant.now(); t -> then.isAfter(t)` |
| Unbound           | `String::toLowerCase`    | `str -> str.toLowerCase()`                           |
| Class Constructor | `TreeMap<K,V>::new`      | `() -> new TreeMap`                                  |
| Array Constructor | `int[]::new`             | `len -> new int[len]`                                |
- **Where method references are shorter and clearer, use them; there they aren't, stick with lambdas**

### Item 44: Favor the Use of Standard Functional Interfaces
- With lambdas, *Template Method* [Gamma 95] is less attractive
  - Modern alternative is to provide static factory or constructor that accepts function object
  - More generally, more constructors and methods that take function objects
- Can use `LinkedHashMap` as cache by overriding protected `removeEldestEntry` method
  - Allows map to grow to one hundred entries, then deletes eldest entry when a new key is added

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > 100;
}
```

- Works fine, but would be better if static factory or constructor took function object
- From `removeEldestEntry`, would expect function object to take `Map.Entry<K,V>` and return `boolean`
  - `removeEldestEntry` calls `size()` to get number of entries in map, but function object is not an instance method on map and can't capture it because map doesn't exist yet
  - Map must pass itself to function object, which takes map as input
- Functional interface might look like

```java
// Unnecessary functional interface; use a standard one instead.

@FunctionalInterface
interface EldestEntryRemovalFunction<K,V> {
    boolean remove(Map<K,V> map, Map.Entry<K,V> eldest);
}
```

- This would work, but you don't need to declare a new interface just for this
- **If one of the standard functional interfaces does the job, you should generally use it in preference to a purpose-build functional interface**
- `Predicate` interface provides methods for combining predicates
  - Could replace `EldestEntryRemovalFunction` with `BiPredicate<Map<K, V>, Map.Entry<K,V>>`
- 43 interfaces in `java.util.Function`
  - 6 basic interfaces

| Type              | Example                  | Lambda Equivalent                                    |
|-------------------|--------------------------|------------------------------------------------------|
| Static            | `Integer::parseInt`      | `str -> Integer.parseInt(str)`                       |
| Bound             | `Instant.now()::isAfter` | `Instant then = Instant.now(); t -> then.isAfter(t)` |
| Unbound           | `String::toLowerCase`    | `str -> str.toLowerCase()`                           |
| Class Constructor | `TreeMap<K,V>::new`      | `() -> new TreeMap`                                  |
| Array Constructor | `int[]::new`             | `len -> new int[len]`                                |

- Also flavors for `int`, `long`, and `double`

### Item 45: Use Streams Judiciously
### Item 46: Prefer Side-Effect-Free Functions in Streams
### Item 47: Prefer Collection to Stream as a Return Type
### Item 48: Use Caution When Making Streams Parallel

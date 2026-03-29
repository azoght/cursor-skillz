# solid-ify

Refactor code to adhere to the SOLID principles.

## Role
**You are a refactoring agent.** Investigate code for SOLID principle violations and fix them.

**Your responsibilities:**
- Understand what the SOLID principles are in every object-oriented programming language
- Look for "code smells" or signs that SOLID principles are being violated
- Use refactoring techniques to fix SOLID principle violations in code
- Provide explanation of refactorings while referencing SOLID principles

**Boundaries:** 
- Code examples are shallow and in Java only. Nevertheless, these examples should be understandable in any programming language that uses OOP. Furthermore, each code example doesn't show each possible code smell that signals the SOLID principle it demonstrates. Read with caution.
- Whenever adding a new module (class, interface, etc.), please create it in a new file instead of an existing file.
- Only add code comments as docstrings for classes or methods.

## Prerequisites
None

## Usage
```
/solid-ify [optional: directory]
```
**Examples:**
- `/solid-ify`
- `/solid-ify src` 
- `/solid-ify backend/tests`

## Instructions

### Phase 0: Code Scoping
Scan the command for a directory. 

If no directory is provided, proceed and use the entire codebase for this command.

If a directory is provided, verify that the directory exists in the codebase. If it doesn't, end this command with the message "Directory doesn't exist. Please enter a path to a directory relative to the root.". Otherwise, proceed and use just the provided directory for this command.

### Phase 1: Single Responsibility Principle

"A class should only have one reason to change."

**Example:**

```java
// Violation
public class Book {

    private String name;
    private String author;
    private String text;

    // ...

    public String replaceWordInText(String word, String replacementWord){
        return text.replaceAll(word, replacementWord);
    }

    public boolean isWordInText(String word){
        return text.contains(word);
    }

    void printTextToConsole(){
        // ...
    }
}

// Fix
public class Book {

    private String name;
    private String author;
    private String text;

    // ...

    public String replaceWordInText(String word, String replacementWord){
        return text.replaceAll(word, replacementWord);
    }

    public boolean isWordInText(String word){
        return text.contains(word);
    }
}

public class BookPrinter {

    void printTextToConsole(String text){
        // ...
    }

    void printTextToAnotherMedium(String text){
        // ...
    }
}
```

### Phase 2: Open-Closed Principle
"Software entities should be open for extension but closed for modification."

**Example:**

```java
// Violation
public class Guitar {

    private String make;
    private String model;
    private int volume;
    
    private String flameColor;

    // ...
}

// Fix
public class Guitar {

    private String make;
    private String model;
    private int volume;

    // ...
}

public class SuperCoolGuitarWithFlames extends Guitar {

    private String flameColor;

    // ...
}
```

### Phase 3: Liskov Substitution Principle

"Subtypes must be replaceable with their base types without affecting the correctness of the program."

**Example:**

```java
// Violation
public interface Car {
    void turnOnEngine();
    void accelerate();
}

public class MotorCar implements Car {

    private Engine engine;

    // ...

    public void turnOnEngine() {
        engine.on();
    }

    public void accelerate() {
        engine.powerOn(1000);
    }
}

public class ElectricCar implements Car {

    public void turnOnEngine() {
        throw new AssertionError("I don't have an engine!");
    }

    public void accelerate() {
        // ...
    }
}

// Fix
public interface Car {
    void accelerate();
}

public interface CarWithEngine extends Car {
    void turnOnEngine();
}

public class MotorCar implements CarWithEngine {

    private Engine engine;

    // ...

    public void turnOnEngine() {
        engine.on();
    }

    public void accelerate() {
        engine.powerOn(1000);
    }
}

public class ElectricCar implements Car {

    public void accelerate() {
        // ...
    }
}
```

### Phase 4: Interface Segregation Principle
"Clients should not be forced to depend on interfaces they do not use."

**Example:**

```java
// Violation
public interface BearKeeper {
    void washTheBear();
    void feedTheBear();
    void petTheBear();
}

public class BearCarer implements BearKeeper {

    public void washTheBear() {
        // ...
    }

    public void feedTheBear() {
        // ...
    }

    public void petTheBear() {
        throw new AssertionError("I can't do that...");
    }
}

// Fix
public interface BearCleaner {
    void washTheBear();
}

public interface BearFeeder {
    void feedTheBear();
}

public interface BearPetter {
    void petTheBear();
}

public class BearCarer implements BearCleaner, BearFeeder {

    public void washTheBear() {
        // ...
    }

    public void feedTheBear() {
        // ...
    }
}

public class CrazyPerson implements BearPetter {

    public void petTheBear() {
        // ...
    }
}
```

### Phase 5: Dependency Inversion Principle
"High-level modules should not depend on low-level modules, both should depend on abstractions."

**Example:**

```java
// Violation
public class Windows98Machine {

    private final StandardKeyboard keyboard;
    private final Monitor monitor;

    public Windows98Machine() {
        monitor = new Monitor();
        keyboard = new StandardKeyboard();
    }

}

// Fix
public interface Keyboard { }

public class StandardKeyboard implements Keyboard { }

public class Windows98Machine{

    private final Keyboard keyboard;
    private final Monitor monitor;

    public Windows98Machine(Keyboard keyboard, Monitor monitor) {
        this.keyboard = keyboard;
        this.monitor = monitor;
    }
}
```

## Output
**End your response with:**
```md
✅ SOLID refactoring complete!

**Files Changed:**
- [First File Changed] : [Description of Changes]
- [Second File Changed] : [Description of Changes]
...
```

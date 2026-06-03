# Technical Interview Preparation Guide

## SOLID Principles

### 1. Single Responsibility Principle (SRP)

- **Definition**: A class should have only one reason to change
- **Example**: 
  ```java
  // Good: Separate classes for different responsibilities
  class UserService { /* user operations only */ }
  class EmailService { /* email operations only */ }
  class ValidationService { /* validation logic only */ }
  ```

### 2. Open/Closed Principle (OCP)

- **Definition**: Software entities should be open for extension but closed for modification
- **Example**: Using interfaces and abstract classes
  ```java
  interface PaymentProcessor {
      void processPayment();
  }

  class CreditCardProcessor implements PaymentProcessor { /* implementation */ }
  class PayPalProcessor implements PaymentProcessor { /* implementation */ }
  ```

### 3. Liskov Substitution Principle (LSP)

- **Definition**: Subtypes must be substitutable for their base types
Child Class should not break parent class type defination and behavior.
LSP is a foundation principle of solid principle state that if a program or module is using base class then derived class should be able to extend their base class without changing their original implementation.

- **Example**: Any subclass should work where parent class is expected
  ```java
  List<String> list = new ArrayList<>(); // ArrayList can substitute List
  ```

### 4. Interface Segregation Principle (ISP)

- **Definition**: Clients should not be forced to depend on interfaces they don't use
- **Example**: Small, focused interfaces instead of large, monolithic ones
  ```java
  interface Printer { void print(); }
  interface Scanner { void scan(); }
  // Instead of one large interface with both methods
  ```

### 5. Dependency Inversion Principle (DIP)

- **Definition**: High-level modules should not depend on low-level modules; both should depend on abstractions
- **Example**: Depend on interfaces, not concrete implementations
  ```java
  class UserService {
      private final UserRepository repository; // Interface, not concrete class
      public UserService(UserRepository repository) { this.repository = repository; }
  }
  ```

## OOP Principles in Java

### 1. Encapsulation

- **Definition**: Bundling data and methods that operate on that data within a single unit
- **Example**: Private fields with public getters/setters
  ```java
  public class Person {
      private String name;
      private int age;
      
      public String getName() { return name; }
      public void setName(String name) { this.name = name; }
  }
  ```

### 2. Inheritance

- **Definition**: Mechanism that allows a class to inherit properties and methods from another class
- **Example**: 
  ```java
  class Animal {
      protected String name;
      public void eat() { System.out.println("Eating..."); }
  }

  class Dog extends Animal {
      public void bark() { System.out.println("Woof!"); }
  }
  ```

### 3. Polymorphism

- **Definition**: Ability to present the same interface for different underlying forms
- **Types**:
  - **Compile-time**: Method overloading
  - **Runtime**: Method overriding
- **Example**:
  ```java
  // Method overloading
  public int add(int a, int b) { return a + b; }
  public double add(double a, double b) { return a + b; }

  // Method overriding
  @Override
  public void makeSound() { System.out.println("Woof!"); }
  ```

### 4. Abstraction

- **Definition**: Hiding complex implementation details and showing only necessary features
- **Example**: Abstract classes and interfaces
  ```java
  abstract class Shape {
      abstract double calculateArea();
  }

  interface Drawable {
      void draw();
  }
  ```

## Service Deployment & Discovery

### Server Naming Convention

When multiple services are deployed on the same server:

```
{service-name}-{environment}-{instance-number}
```

**Examples**:

- `user-service-prod-01`
- `order-service-prod-01`
- `payment-service-prod-01`

### Service Discovery & Health Checks

**How discovery server knows service status**:

1. **Health Check Endpoints**:
  ```java
   @RestController
   public class HealthController {
       @GetMapping("/health")
       public ResponseEntity<String> health() {
           // Check database, external services, etc.
           return ResponseEntity.ok("UP");
       }
   }
  ```
2. **Heartbeat Mechanism**: Services send periodic status updates
3. **Active Monitoring**: Discovery server actively probes services
4. **Circuit Breaker Pattern**: Detect failing services automatically

**Tools**:

- **Eureka**: Netflix service discovery
- **Consul**: HashiCorp service mesh
- **Kubernetes**: Built-in service discovery

## Exception Handling in Spring Boot

### 1. Global Exception Handler

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception ex) {
        ErrorResponse error = new ErrorResponse("Internal Server Error", ex.getMessage());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse("Not Found", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

### 2. Custom Exceptions

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

### 3. Try-Catch Blocks

```java
@Service
public class UserService {
    public User getUserById(Long id) {
        try {
            return userRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("User not found"));
        } catch (DataAccessException e) {
            throw new ServiceException("Database error occurred", e);
        }
    }
}
```

### 4. Response Entity with Error

```java
@GetMapping("/users/{id}")
public ResponseEntity<?> getUser(@PathVariable Long id) {
    try {
        User user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    } catch (ResourceNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("User not found", e.getMessage()));
    }
}
```

## Problem Solving Solutions

### 1. Move Zeros

**Problem**: Move all zeros to the end while maintaining relative order of non-zero elements.

```java
public void moveZeroes(int[] nums) {
    int nonZeroIndex = 0;
    
    // Move all non-zero elements to the front
    for (int i = 0; i < nums.length; i++) {
        if (nums[i] != 0) {
            nums[nonZeroIndex++] = nums[i];
        }
    }
    
    // Fill remaining positions with zeros
    while (nonZeroIndex < nums.length) {
        nums[nonZeroIndex++] = 0;
    }
}
```

### 2. Odd-Even Pattern

**Problem**: Print numbers in odd-even pattern (1, 3, 5, 2, 4, 6...)

```java
public void printOddEven(int n) {
    int odd = 1, even = 2;
    
    for (int i = 0; i < n; i++) {
        if (i % 2 == 0) {
            System.out.print(odd + " ");
            odd += 2;
        } else {
            System.out.print(even + " ");
            even += 2;
        }
    }
}
```

### 3. Isomorphic Strings

**Problem**: Check if two strings are isomorphic (characters can be mapped to each other).

```java
public boolean isIsomorphic(String s, String t) {
    if (s.length() != t.length()) return false;
    
    Map<Character, Character> sToT = new HashMap<>();
    Map<Character, Character> tToS = new HashMap<>();
    
    for (int i = 0; i < s.length(); i++) {
        char sChar = s.charAt(i);
        char tChar = t.charAt(i);
        
        if (sToT.containsKey(sChar)) {
            if (sToT.get(sChar) != tChar) return false;
        } else {
            if (tToS.containsKey(tChar)) return false;
            sToT.put(sChar, tChar);
            tToS.put(tChar, sChar);
        }
    }
    return true;
}
```

### 4. Total Traffic in Traffic Signal

**Problem**: Calculate total traffic passing through a traffic signal.

```java
public int calculateTotalTraffic(int[] vehicles, int greenTime, int redTime) {
    int totalTraffic = 0;
    int currentTime = 0;
    int vehicleIndex = 0;
    
    while (vehicleIndex < vehicles.length) {
        // Green light - vehicles can pass
        int greenStart = currentTime;
        int greenEnd = currentTime + greenTime;
        
        while (vehicleIndex < vehicles.length && 
               vehicles[vehicleIndex] <= greenEnd) {
            totalTraffic++;
            vehicleIndex++;
        }
        
        currentTime = greenEnd + redTime; // Add red light time
    }
    
    return totalTraffic;
}
```

### 5. Pattern Count in String

**Problem**: Count occurrences of a pattern in a string.

```java
public int countPattern(String text, String pattern) {
    if (text.length() < pattern.length()) return 0;
    
    int count = 0;
    for (int i = 0; i <= text.length() - pattern.length(); i++) {
        if (text.substring(i, i + pattern.length()).equals(pattern)) {
            count++;
        }
    }
    return count;
}

// Using KMP algorithm for better performance
public int countPatternKMP(String text, String pattern) {
    int[] lps = computeLPS(pattern);
    int i = 0, j = 0, count = 0;
    
    while (i < text.length()) {
        if (pattern.charAt(j) == text.charAt(i)) {
            i++;
            j++;
        }
        
        if (j == pattern.length()) {
            count++;
            j = lps[j - 1];
        } else if (i < text.length() && pattern.charAt(j) != text.charAt(i)) {
            if (j != 0) {
                j = lps[j - 1];
            } else {
                i++;
            }
        }
    }
    return count;
}

private int[] computeLPS(String pattern) {
    int[] lps = new int[pattern.length()];
    int len = 0, i = 1;
    
    while (i < pattern.length()) {
        if (pattern.charAt(i) == pattern.charAt(len)) {
            len++;
            lps[i] = len;
            i++;
        } else {
            if (len != 0) {
                len = lps[len - 1];
            } else {
                lps[i] = 0;
                i++;
            }
        }
    }
    return lps;
}
```

## Additional Tips for Interviews

### 1. Code Quality

- Write clean, readable code
- Use meaningful variable names
- Add comments for complex logic
- Handle edge cases

### 2. Problem-Solving Approach

- Understand the problem clearly
- Ask clarifying questions
- Think out loud
- Start with brute force, then optimize
- Consider time and space complexity

### 3. Communication

- Explain your thought process
- Discuss trade-offs
- Ask questions when unclear
- Show enthusiasm and curiosity

### 4. Practice Areas

- Data structures (Arrays, Lists, Trees, Graphs)
- Algorithms (Sorting, Searching, Dynamic Programming)
- System design concepts
- Database design and queries
- API design principle



There were several parts: First part: Ice breaking Q: Introduce yourself. Q: Why Tekarsh? Second part: The submitted task and my resume related questions - Q: Why I used a certain wait in one of my code file, Q: About my project's stacks, experience, biggest challenge I faced while working as an automation engineer/QA Third part: Problem solving and programming language related - I have been given two problems - i) Finding the second largest element from a given array, ii) Printing the indexes of least two numbers from a given array. Q: Difference between async and await keywords. Last part: Regular interview questions - Q: Expected salary? Q: Notice period?
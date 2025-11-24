# Data Structures & Algorithms

## Quick Reference Snippets

---

## Common Data Structures

```java
// Collections Framework
List<T>        // ArrayList, LinkedList
Set<T>         // HashSet, TreeSet
Map<K, V>      // HashMap, TreeMap
Queue<T>       // PriorityQueue, ArrayDeque
```

---

## Algorithm Patterns

```java
// Two Pointers
int left = 0, right = array.length - 1;

// Sliding Window
for (int end = 0; end < array.length; end++) {
    // Window logic
}

// Binary Search
int left = 0, right = array.length - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    // Search logic
}
```

---


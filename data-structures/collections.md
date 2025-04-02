---
title: "Data Structures for C# Developers"
date_created: 2025-04-02
date_updated: 2025-04-02
authors: ["Repository Maintainers"]
tags: ["data structures", "collections", "algorithms", "performance", "reference"]
difficulty: "intermediate"
---

# Data Structures for C# Developers

## Overview

Data structures are fundamental building blocks in software development that allow for efficient data organization, storage, and manipulation. For C# developers, understanding the built-in data structures in the .NET framework as well as custom implementations is essential for writing performant, scalable, and maintainable applications. This guide explores key data structures available in C#, their implementation details, performance characteristics, and usage patterns.

## Core .NET Collection Types

The .NET Framework provides a comprehensive set of collection types in the `System.Collections.Generic` namespace that are the foundation for most data structure needs.

### Lists

`List<T>` is a dynamically-sized array that provides random access and efficient operations at the end of the collection.

```csharp
// Creating and using a List
List<string> names = new List<string>();
names.Add("Alice");
names.Add("Bob");
names.Add("Charlie");

// Random access
string secondName = names[1]; // "Bob"

// Insert at specific position
names.Insert(1, "Anna"); // ["Alice", "Anna", "Bob", "Charlie"]

// Remove item
names.Remove("Bob"); // ["Alice", "Anna", "Charlie"]

// Performance characteristics
// - Access: O(1)
// - Search: O(n)
// - Insert/Remove at end: O(1) amortized
// - Insert/Remove at middle: O(n)
```

#### Implementation Details

`List<T>` is built on an array that automatically resizes when capacity is reached:

```csharp
public class SimpleList<T>
{
    private T[] _items;
    private int _size;
    private const int DefaultCapacity = 4;

    public SimpleList()
    {
        _items = new T[DefaultCapacity];
        _size = 0;
    }

    public void Add(T item)
    {
        if (_size == _items.Length)
        {
            // Double the capacity when full
            int newCapacity = _items.Length * 2;
            Array.Resize(ref _items, newCapacity);
        }
        _items[_size++] = item;
    }

    public T this[int index]
    {
        get
        {
            if (index < 0 || index >= _size)
                throw new IndexOutOfRangeException();
            return _items[index];
        }
        set
        {
            if (index < 0 || index >= _size)
                throw new IndexOutOfRangeException();
            _items[index] = value;
        }
    }

    public int Count => _size;
}
```

### Dictionaries

`Dictionary<TKey, TValue>` provides key-value pair storage with fast lookup by key.

```csharp
// Creating and using a Dictionary
Dictionary<string, int> ages = new Dictionary<string, int>();
ages.Add("Alice", 25);
ages.Add("Bob", 30);
ages["Charlie"] = 22; // Alternative syntax

// Lookup by key
int bobsAge = ages["Bob"]; // 30

// Check if key exists
if (ages.ContainsKey("David"))
{
    // This won't execute
}

// Safe retrieval with TryGetValue
if (ages.TryGetValue("Alice", out int aliceAge))
{
    Console.WriteLine($"Alice's age: {aliceAge}");
}

// Performance characteristics
// - Lookup, Insert, Delete: Average O(1), Worst O(n)
```

#### Implementation Details

`Dictionary<TKey, TValue>` is implemented as a hash table with separate chaining for collision resolution:

```csharp
public class SimpleDictionary<TKey, TValue>
{
    private struct Entry
    {
        public int HashCode;
        public TKey Key;
        public TValue Value;
        public int Next; // Index of next entry in chain
    }

    private Entry[] _entries;
    private int[] _buckets;
    private int _count;
    private const int DefaultCapacity = 4;

    public SimpleDictionary()
    {
        _entries = new Entry[DefaultCapacity];
        _buckets = new int[DefaultCapacity];
        for (int i = 0; i < _buckets.Length; i++)
            _buckets[i] = -1; // -1 indicates empty bucket
    }

    public void Add(TKey key, TValue value)
    {
        if (key == null)
            throw new ArgumentNullException(nameof(key));

        int hashCode = key.GetHashCode() & 0x7FFFFFFF; // Remove sign bit
        int bucketIndex = hashCode % _buckets.Length;

        // Check for duplicate key
        for (int i = _buckets[bucketIndex]; i >= 0; i = _entries[i].Next)
        {
            if (_entries[i].HashCode == hashCode && Equals(_entries[i].Key, key))
                throw new ArgumentException("Key already exists");
        }

        // Resize if needed
        if (_count == _entries.Length)
        {
            Resize();
            bucketIndex = hashCode % _buckets.Length;
        }

        // Add new entry
        int index = _count++;
        _entries[index].HashCode = hashCode;
        _entries[index].Key = key;
        _entries[index].Value = value;
        _entries[index].Next = _buckets[bucketIndex];
        _buckets[bucketIndex] = index;
    }

    private void Resize()
    {
        // Implementation omitted for brevity
    }

    public bool TryGetValue(TKey key, out TValue value)
    {
        if (key == null)
            throw new ArgumentNullException(nameof(key));

        value = default;
        
        if (_count == 0)
            return false;

        int hashCode = key.GetHashCode() & 0x7FFFFFFF;
        int bucketIndex = hashCode % _buckets.Length;

        for (int i = _buckets[bucketIndex]; i >= 0; i = _entries[i].Next)
        {
            if (_entries[i].HashCode == hashCode && Equals(_entries[i].Key, key))
            {
                value = _entries[i].Value;
                return true;
            }
        }

        return false;
    }
}
```

### Sets

`HashSet<T>` provides a collection of unique items with efficient lookup, add, and remove operations.

```csharp
// Creating and using a HashSet
HashSet<string> uniqueNames = new HashSet<string>();
uniqueNames.Add("Alice");
uniqueNames.Add("Bob");
uniqueNames.Add("Alice"); // No effect, already exists

// Checking membership
bool containsBob = uniqueNames.Contains("Bob"); // true

// Set operations
HashSet<string> otherNames = new HashSet<string> { "Bob", "Charlie", "David" };
uniqueNames.UnionWith(otherNames); // ["Alice", "Bob", "Charlie", "David"]

HashSet<string> intersection = new HashSet<string>(uniqueNames);
intersection.IntersectWith(otherNames); // ["Bob", "Charlie", "David"]

// Performance characteristics
// - Contains, Add, Remove: Average O(1), Worst O(n)
```

### Queues

`Queue<T>` implements a first-in, first-out (FIFO) collection.

```csharp
// Creating and using a Queue
Queue<string> customerQueue = new Queue<string>();
customerQueue.Enqueue("Customer 1");
customerQueue.Enqueue("Customer 2");
customerQueue.Enqueue("Customer 3");

// Process next customer
string nextCustomer = customerQueue.Dequeue(); // "Customer 1"

// Check who's next without removing
string peekedCustomer = customerQueue.Peek(); // "Customer 2"

// Check if queue contains a specific item
bool hasCustomer3 = customerQueue.Contains("Customer 3"); // true

// Performance characteristics
// - Enqueue, Dequeue, Peek: O(1)
// - Contains: O(n)
```

### Stacks

`Stack<T>` implements a last-in, first-out (LIFO) collection.

```csharp
// Creating and using a Stack
Stack<string> browserHistory = new Stack<string>();
browserHistory.Push("Homepage");
browserHistory.Push("Products");
browserHistory.Push("Product Details");

// Go back one page
string previousPage = browserHistory.Pop(); // "Product Details"

// Check current page without navigation
string currentPage = browserHistory.Peek(); // "Products"

// Performance characteristics
// - Push, Pop, Peek: O(1)
// - Contains: O(n)
```

### Linked Lists

`LinkedList<T>` implements a doubly-linked list providing efficient insertions and removals.

```csharp
// Creating and using a LinkedList
LinkedList<string> linkedList = new LinkedList<string>();
linkedList.AddLast("Item 1");
linkedList.AddLast("Item 3");

// Insert between items
LinkedListNode<string> node = linkedList.Find("Item 3");
linkedList.AddBefore(node, "Item 2");

// Traverse the list
foreach (string item in linkedList)
{
    Console.WriteLine(item);
}

// Performance characteristics
// - AddFirst, AddLast, RemoveFirst, RemoveLast: O(1)
// - Find, AddBefore, AddAfter, Remove: O(n)
```

## Specialized Collections

### SortedDictionary and SortedList

Both provide sorted key-value pairs but with different performance trade-offs.

```csharp
// SortedDictionary
SortedDictionary<string, int> sortedScores = new SortedDictionary<string, int>();
sortedScores.Add("Charlie", 85);
sortedScores.Add("Alice", 95);
sortedScores.Add("Bob", 90);

// Keys are automatically sorted
foreach (var kvp in sortedScores)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
// Output:
// Alice: 95
// Bob: 90
// Charlie: 85

// SortedList
SortedList<string, int> sortedListScores = new SortedList<string, int>();
sortedListScores.Add("Charlie", 85);
sortedListScores.Add("Alice", 95);
sortedListScores.Add("Bob", 90);

// Access by index is available (unlike SortedDictionary)
string secondStudent = sortedListScores.Keys[1]; // "Bob"

// Performance comparison
// SortedDictionary:
// - Add, Remove, Search: O(log n)
// - Memory: More overhead
// - Better for frequent insertions/deletions

// SortedList:
// - Add, Remove: O(n) due to array resizing
// - Search: O(log n)
// - Memory: Less overhead
// - Better for fixed data, index-based access
```

### ConcurrentCollections

Thread-safe collections in the `System.Collections.Concurrent` namespace:

```csharp
// Thread-safe dictionary
ConcurrentDictionary<string, int> concurrentDict = new ConcurrentDictionary<string, int>();

// Thread-safe add or update
concurrentDict.AddOrUpdate("Key1", 
    addValue => 1,                             // Used if key doesn't exist
    (key, oldValue) => oldValue + 1);          // Used if key exists

// Get or add atomically
int value = concurrentDict.GetOrAdd("Key2", 42);

// Thread-safe queue
ConcurrentQueue<string> messageQueue = new ConcurrentQueue<string>();
messageQueue.Enqueue("Message 1");

bool success = messageQueue.TryDequeue(out string message);
if (success)
{
    Console.WriteLine($"Processed: {message}");
}

// Other concurrent collections
ConcurrentBag<T> // Unordered collection optimized for multiple threads adding items
ConcurrentStack<T> // LIFO with thread-safety
BlockingCollection<T> // Bounded collection with blocking operations
```

### Immutable Collections

Collections that cannot be modified after creation, in the `System.Collections.Immutable` namespace:

```csharp
// Immutable list
ImmutableList<string> immutableList = ImmutableList.Create<string>();
ImmutableList<string> newList = immutableList.Add("Item 1");
ImmutableList<string> newerList = newList.Add("Item 2");

// Original list remains unchanged
Console.WriteLine(immutableList.Count); // 0
Console.WriteLine(newList.Count);      // 1
Console.WriteLine(newerList.Count);    // 2

// Immutable dictionary
ImmutableDictionary<string, int> immutableDict = ImmutableDictionary.Create<string, int>();
ImmutableDictionary<string, int> newDict = immutableDict.Add("Key1", 1);

// Builders for efficient batch construction
ImmutableList<string>.Builder builder = ImmutableList.CreateBuilder<string>();
builder.Add("Item 1");
builder.Add("Item 2");
builder.Add("Item 3");
ImmutableList<string> finalList = builder.ToImmutable();

// Available immutable collections:
// ImmutableArray<T>
// ImmutableList<T>
// ImmutableDictionary<TKey, TValue>
// ImmutableHashSet<T>
// ImmutableSortedDictionary<TKey, TValue>
// ImmutableSortedSet<T>
// ImmutableQueue<T>
// ImmutableStack<T>
```

## Custom Data Structures

### Trees

Binary trees, B-trees, and other hierarchical structures.

#### Binary Search Tree

```csharp
public class BinarySearchTree<T> where T : IComparable<T>
{
    private class Node
    {
        public T Value { get; set; }
        public Node Left { get; set; }
        public Node Right { get; set; }

        public Node(T value)
        {
            Value = value;
            Left = null;
            Right = null;
        }
    }

    private Node _root;

    public BinarySearchTree()
    {
        _root = null;
    }

    public void Insert(T value)
    {
        _root = InsertRecursive(_root, value);
    }

    private Node InsertRecursive(Node node, T value)
    {
        if (node == null)
            return new Node(value);

        if (value.CompareTo(node.Value) < 0)
            node.Left = InsertRecursive(node.Left, value);
        else if (value.CompareTo(node.Value) > 0)
            node.Right = InsertRecursive(node.Right, value);

        return node;
    }

    public bool Contains(T value)
    {
        return ContainsRecursive(_root, value);
    }

    private bool ContainsRecursive(Node node, T value)
    {
        if (node == null)
            return false;

        if (value.CompareTo(node.Value) == 0)
            return true;

        return value.CompareTo(node.Value) < 0
            ? ContainsRecursive(node.Left, value)
            : ContainsRecursive(node.Right, value);
    }

    public void InOrderTraversal(Action<T> action)
    {
        InOrderTraversalRecursive(_root, action);
    }

    private void InOrderTraversalRecursive(Node node, Action<T> action)
    {
        if (node == null)
            return;

        InOrderTraversalRecursive(node.Left, action);
        action(node.Value);
        InOrderTraversalRecursive(node.Right, action);
    }
}

// Usage
var bst = new BinarySearchTree<int>();
bst.Insert(50);
bst.Insert(30);
bst.Insert(70);
bst.Insert(20);
bst.Insert(40);

Console.WriteLine(bst.Contains(30)); // true
Console.WriteLine(bst.Contains(60)); // false

// Traverse in order (sorted)
bst.InOrderTraversal(value => Console.Write($"{value} ")); // 20 30 40 50 70
```

### Graphs

```csharp
public class Graph<T>
{
    private Dictionary<T, List<T>> _adjacencyList;

    public Graph()
    {
        _adjacencyList = new Dictionary<T, List<T>>();
    }

    public void AddVertex(T vertex)
    {
        if (!_adjacencyList.ContainsKey(vertex))
        {
            _adjacencyList[vertex] = new List<T>();
        }
    }

    public void AddEdge(T source, T destination)
    {
        if (!_adjacencyList.ContainsKey(source))
            AddVertex(source);

        if (!_adjacencyList.ContainsKey(destination))
            AddVertex(destination);

        _adjacencyList[source].Add(destination);
        // For undirected graph, add the reverse edge too
        // _adjacencyList[destination].Add(source);
    }

    public List<T> GetNeighbors(T vertex)
    {
        if (_adjacencyList.ContainsKey(vertex))
            return _adjacencyList[vertex];

        return new List<T>();
    }

    public void BreadthFirstTraversal(T startVertex, Action<T> action)
    {
        if (!_adjacencyList.ContainsKey(startVertex))
            return;

        HashSet<T> visited = new HashSet<T>();
        Queue<T> queue = new Queue<T>();

        visited.Add(startVertex);
        queue.Enqueue(startVertex);

        while (queue.Count > 0)
        {
            T vertex = queue.Dequeue();
            action(vertex);

            foreach (T neighbor in _adjacencyList[vertex])
            {
                if (!visited.Contains(neighbor))
                {
                    visited.Add(neighbor);
                    queue.Enqueue(neighbor);
                }
            }
        }
    }

    public void DepthFirstTraversal(T startVertex, Action<T> action)
    {
        if (!_adjacencyList.ContainsKey(startVertex))
            return;

        HashSet<T> visited = new HashSet<T>();
        DepthFirstRecursive(startVertex, visited, action);
    }

    private void DepthFirstRecursive(T vertex, HashSet<T> visited, Action<T> action)
    {
        visited.Add(vertex);
        action(vertex);

        foreach (T neighbor in _adjacencyList[vertex])
        {
            if (!visited.Contains(neighbor))
            {
                DepthFirstRecursive(neighbor, visited, action);
            }
        }
    }
}

// Usage
var graph = new Graph<string>();
graph.AddVertex("A");
graph.AddVertex("B");
graph.AddVertex("C");
graph.AddVertex("D");
graph.AddEdge("A", "B");
graph.AddEdge("A", "C");
graph.AddEdge("B", "D");
graph.AddEdge("C", "D");

Console.WriteLine("BFS traversal:");
graph.BreadthFirstTraversal("A", v => Console.Write($"{v} ")); // A B C D

Console.WriteLine("\nDFS traversal:");
graph.DepthFirstTraversal("A", v => Console.Write($"{v} ")); // A B D C
```

### Tries

Efficient for string operations and prefix searches:

```csharp
public class Trie
{
    private class TrieNode
    {
        public Dictionary<char, TrieNode> Children { get; } = new Dictionary<char, TrieNode>();
        public bool IsEndOfWord { get; set; }
    }

    private readonly TrieNode _root = new TrieNode();

    public void Insert(string word)
    {
        TrieNode current = _root;

        foreach (char c in word)
        {
            if (!current.Children.ContainsKey(c))
            {
                current.Children[c] = new TrieNode();
            }
            current = current.Children[c];
        }

        current.IsEndOfWord = true;
    }

    public bool Search(string word)
    {
        TrieNode node = FindNode(word);
        return node != null && node.IsEndOfWord;
    }

    public bool StartsWith(string prefix)
    {
        return FindNode(prefix) != null;
    }

    private TrieNode FindNode(string key)
    {
        TrieNode current = _root;

        foreach (char c in key)
        {
            if (!current.Children.ContainsKey(c))
            {
                return null;
            }
            current = current.Children[c];
        }

        return current;
    }

    public List<string> GetAllWordsWithPrefix(string prefix)
    {
        List<string> results = new List<string>();
        TrieNode prefixNode = FindNode(prefix);

        if (prefixNode != null)
        {
            CollectWords(prefixNode, prefix, results);
        }

        return results;
    }

    private void CollectWords(TrieNode node, string prefix, List<string> words)
    {
        if (node.IsEndOfWord)
        {
            words.Add(prefix);
        }

        foreach (var kvp in node.Children)
        {
            CollectWords(kvp.Value, prefix + kvp.Key, words);
        }
    }
}

// Usage
var trie = new Trie();
trie.Insert("apple");
trie.Insert("application");
trie.Insert("banana");
trie.Insert("app");

Console.WriteLine(trie.Search("apple"));     // true
Console.WriteLine(trie.Search("apples"));    // false
Console.WriteLine(trie.StartsWith("app"));   // true
Console.WriteLine(trie.StartsWith("ban"));   // true

var appWords = trie.GetAllWordsWithPrefix("app");
foreach (var word in appWords)
{
    Console.WriteLine(word);  // app, apple, application
}
```

### Heaps

Priority queues and efficient operations for minimum/maximum value:

```csharp
// Using built-in PriorityQueue (available in .NET 6+)
PriorityQueue<string, int> priorityQueue = new PriorityQueue<string, int>();
priorityQueue.Enqueue("Low Priority Task", 3);
priorityQueue.Enqueue("High Priority Task", 1);
priorityQueue.Enqueue("Medium Priority Task", 2);

// Dequeue in priority order
string nextTask = priorityQueue.Dequeue(); // "High Priority Task"
nextTask = priorityQueue.Dequeue();        // "Medium Priority Task"
nextTask = priorityQueue.Dequeue();        // "Low Priority Task"

// Custom MinHeap implementation for earlier .NET versions
public class MinHeap<T> where T : IComparable<T>
{
    private List<T> _items;

    public MinHeap()
    {
        _items = new List<T>();
    }

    public int Count => _items.Count;

    public void Insert(T item)
    {
        _items.Add(item);
        int i = _items.Count - 1;
        BubbleUp(i);
    }

    public T ExtractMin()
    {
        if (_items.Count == 0)
            throw new InvalidOperationException("Heap is empty");

        T min = _items[0];
        _items[0] = _items[_items.Count - 1];
        _items.RemoveAt(_items.Count - 1);

        if (_items.Count > 0)
            BubbleDown(0);

        return min;
    }

    public T Peek()
    {
        if (_items.Count == 0)
            throw new InvalidOperationException("Heap is empty");

        return _items[0];
    }

    private void BubbleUp(int index)
    {
        int parent = (index - 1) / 2;

        if (index > 0 && _items[index].CompareTo(_items[parent]) < 0)
        {
            // Swap with parent
            T temp = _items[parent];
            _items[parent] = _items[index];
            _items[index] = temp;

            // Recursively bubble up
            BubbleUp(parent);
        }
    }

    private void BubbleDown(int index)
    {
        int leftChild = 2 * index + 1;
        int rightChild = 2 * index + 2;
        int smallest = index;

        if (leftChild < _items.Count && _items[leftChild].CompareTo(_items[smallest]) < 0)
            smallest = leftChild;

        if (rightChild < _items.Count && _items[rightChild].CompareTo(_items[smallest]) < 0)
            smallest = rightChild;

        if (smallest != index)
        {
            // Swap with smallest child
            T temp = _items[smallest];
            _items[smallest] = _items[index];
            _items[index] = temp;

            // Recursively bubble down
            BubbleDown(smallest);
        }
    }
}

// Usage
var minHeap = new MinHeap<int>();
minHeap.Insert(5);
minHeap.Insert(3);
minHeap.Insert(8);
minHeap.Insert(1);

while (minHeap.Count > 0)
{
    Console.WriteLine(minHeap.ExtractMin()); // 1, 3, 5, 8
}
```

## Performance Comparisons

### Time Complexity

| Collection | Add/Insert | Remove | Contains/Find | Access by Index |
|------------|------------|--------|---------------|-----------------|
| List<T>    | O(1)* | O(n) | O(n) | O(1) |
| LinkedList<T> | O(1) | O(1)** | O(n) | O(n) |
| Dictionary<K,V> | O(1)* | O(1)* | O(1)* | N/A |
| SortedDictionary<K,V> | O(log n) | O(log n) | O(log n) | N/A |
| SortedList<K,V> | O(n) | O(n) | O(log n) | O(1) |
| HashSet<T> | O(1)* | O(1)* | O(1)* | N/A |
| SortedSet<T> | O(log n) | O(log n) | O(log n) | N/A |
| Queue<T> | O(1) | O(1) | O(n) | N/A |
| Stack<T> | O(1) | O(1) | O(n) | N/A |
| PriorityQueue<T> | O(log n) | O(log n) | O(n) | N/A |

\* Amortized time, worst case can be O(n)  
\** If you already have the node reference, otherwise O(n) to find the node

### Space Complexity

| Collection | Space Complexity |
|------------|------------------|
| List<T>    | O(n) |
| LinkedList<T> | O(n) |
| Dictionary<K,V> | O(n) |
| SortedDictionary<K,V> | O(n) |
| SortedList<K,V> | O(n) |
| HashSet<T> | O(n) |
| SortedSet<T> | O(n) |
| Queue<T> | O(n) |
| Stack<T> | O(n) |
| PriorityQueue<T> | O(n) |

### Memory Overhead

| Collection | Memory Overhead |
|------------|-----------------|
| List<T> | Low |
| Array<T> | Very Low |
| LinkedList<T> | High (references) |
| Dictionary<K,V> | Medium-High |
| SortedDictionary<K,V> | High (tree structure) |
| SortedList<K,V> | Medium |
| HashSet<T> | Medium |
| Queue<T>/Stack<T> | Low |

## Memory Efficiency and Optimization

### Struct vs Class-based Collections

For large collections of small data, using struct-based collections can significantly reduce memory overhead and improve cache locality:

```csharp
// Class-based (reference type) - More memory overhead
List<MyReferenceType> refList = new List<MyReferenceType>();

// Struct-based (value type) - Less memory overhead for small data
List<MyValueType> valueList = new List<MyValueType>();

// Example: Struct for a 2D point
public struct Point2D
{
    public float X;
    public float Y;
}

// More efficient than a class for large collections
List<Point2D> points = new List<Point2D>(10000);
```

### Using Spans and Memory for Zero-Copy Operations

`Span<T>` and `Memory<T>` enable working with portions of arrays or other memory without copying:

```csharp
// Traditional approach (creates a copy)
byte[] array = new byte[1000];
byte[] slice = array.Skip(500).Take(100).ToArray();

// Using Span (zero-copy)
Span<byte> span = array.AsSpan(500, 100);

// Apply operations directly
span.Fill(0);
span.CopyTo(targetSpan);

// Convert between strings and spans without allocations
ReadOnlySpan<char> nameSpan = "John Smith".AsSpan();
if (nameSpan.Length > 10)
{
    nameSpan = nameSpan.Slice(0, 10);
}
```

### Custom Collections and Pooling

For high-performance scenarios, consider object pooling to reduce garbage collection:

```csharp
// Example using ArrayPool
using System.Buffers;

void ProcessLargeData(Stream dataStream)
{
    // Rent a buffer from the shared pool
    byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
    
    try
    {
        int bytesRead;
        while ((bytesRead = dataStream.Read(buffer, 0, buffer.Length)) > 0)
        {
            // Process data...
            ProcessBuffer(buffer.AsSpan(0, bytesRead));
        }
    }
    finally
    {
        // Return the buffer to the pool
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

## Best Practices

### Choosing the Right Collection

- **Performance-critical operations**: Identify which operations (add, remove, search, iterate) are most critical for your scenario
- **Frequency of updates**: Mutable collections for frequent updates, immutable collections for mostly read scenarios
- **Concurrency requirements**: Use concurrent collections for multi-threaded access
- **Memory constraints**: Consider overhead and memory efficiency
- **Ordering requirements**: Some collections preserve insertion order, others sort automatically

```csharp
// Examples of choosing the right collection
// Scenario: Frequent lookups, infrequent modifications
Dictionary<string, User> usersByUsername = new Dictionary<string, User>();

// Scenario: Need both key-lookup and ordering
SortedDictionary<DateTime, Event> eventsByDate = new SortedDictionary<DateTime, Event>();

// Scenario: Frequent addition/removal at both ends
LinkedList<Request> requestQueue = new LinkedList<Request>();

// Scenario: Multi-threaded processing queue
ConcurrentQueue<WorkItem> workItems = new ConcurrentQueue<WorkItem>();

// Scenario: Manage unique values with set operations
HashSet<string> allowedCategories = new HashSet<string>();
```

### Capacity Planning

Pre-size collections when you know the approximate size to avoid resizing costs:

```csharp
// Pre-size a List to avoid resizing
List<Customer> customers = new List<Customer>(10000);

// Pre-size a Dictionary
Dictionary<int, Product> productsById = new Dictionary<int, Product>(5000);
```

### Enumeration and Modification

Be careful about modifying collections during enumeration:

```csharp
// This will throw an exception
Dictionary<string, int> scores = new Dictionary<string, int>();
scores.Add("Alice", 95);
scores.Add("Bob", 80);

foreach (var kvp in scores)
{
    if (kvp.Value < 90)
    {
        // WRONG: Modifying collection during enumeration
        // scores.Remove(kvp.Key);
    }
}

// Correct approach: Create a list of keys to remove first
List<string> keysToRemove = scores
    .Where(kvp => kvp.Value < 90)
    .Select(kvp => kvp.Key)
    .ToList();

foreach (string key in keysToRemove)
{
    scores.Remove(key);
}

// Alternative: Use ToList() to create a snapshot for enumeration
foreach (var kvp in scores.ToList())
{
    if (kvp.Value < 90)
    {
        scores.Remove(kvp.Key);
    }
}
```

### Handling Concurrency

Use appropriate collection types for multi-threaded scenarios:

```csharp
// Thread-safe with explicit locking
private readonly Dictionary<string, User> _users = new Dictionary<string, User>();
private readonly object _lock = new object();

public void AddUser(User user)
{
    lock (_lock)
    {
        _users[user.Username] = user;
    }
}

public User GetUser(string username)
{
    lock (_lock)
    {
        return _users.TryGetValue(username, out User user) ? user : null;
    }
}

// Better: Use ConcurrentDictionary
private readonly ConcurrentDictionary<string, User> _concurrentUsers = 
    new ConcurrentDictionary<string, User>();

public void AddUserConcurrent(User user)
{
    _concurrentUsers.AddOrUpdate(
        user.Username,
        // Add function
        _ => user,
        // Update function
        (_, _) => user);
}

public User GetUserConcurrent(string username)
{
    return _concurrentUsers.TryGetValue(username, out User user) ? user : null;
}
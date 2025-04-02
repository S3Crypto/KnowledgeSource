---
title: "C# Generics"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["c#", "generics", "type-safety", "performance", "advanced"]
difficulty: "intermediate"
---

# C# Generics

## Overview

Generics are a powerful feature in C# that allow you to define type-safe data structures and algorithms without committing to specific data types. Introduced in C# 2.0, generics enable you to write code that works with any data type while maintaining type safety, improving performance, and reducing code duplication. This document covers the fundamentals of generics in C#, advanced usage patterns, performance considerations, and best practices.

## Core Concepts

### The Need for Generics

Before generics, developers had to choose between:

1. **Type-specific implementations**: Creating separate implementations for each data type, leading to code duplication.

2. **Using `object`**: Using the `object` type to handle any data type, which required boxing/unboxing for value types and lacked compile-time type safety.

Generics solve these problems by providing a way to create reusable code that works with any type while maintaining type safety and performance.

### Generic Types

Generic types allow you to define classes, interfaces, structures, and delegates that can work with any type specified by the consumer:

```csharp
// Generic class definition
public class GenericList<T>
{
    private T[] _items;
    private int _count;
    
    public GenericList(int capacity)
    {
        _items = new T[capacity];
        _count = 0;
    }
    
    public void Add(T item)
    {
        if (_count < _items.Length)
        {
            _items[_count++] = item;
        }
        else
        {
            throw new InvalidOperationException("List is full");
        }
    }
    
    public T GetItem(int index)
    {
        if (index < 0 || index >= _count)
        {
            throw new IndexOutOfRangeException();
        }
        
        return _items[index];
    }
}

// Usage - with string
var stringList = new GenericList<string>(10);
stringList.Add("Hello");
stringList.Add("World");
string item = stringList.GetItem(0); // "Hello"

// Usage - with int
var intList = new GenericList<int>(10);
intList.Add(42);
intList.Add(123);
int number = intList.GetItem(0); // 42
```

### Generic Methods

Generic methods allow you to define methods that can operate on different types without creating multiple overloads:

```csharp
// Generic method
public static T FindMax<T>(T first, T second) where T : IComparable<T>
{
    if (first.CompareTo(second) > 0)
    {
        return first;
    }
    return second;
}

// Usage with different types
int maxInt = FindMax<int>(10, 20); // 20
string maxString = FindMax<string>("apple", "banana"); // "banana"
double maxDouble = FindMax<double>(3.14, 2.71); // 3.14

// Type inference - type argument can be omitted
int maxInt2 = FindMax(10, 20); // Compiler infers T as int
```

### Type Constraints

Type constraints allow you to restrict which types can be used with your generic type or method, enabling you to perform specific operations on the generic type parameters:

```csharp
// Constraint to reference types
public class ReferenceCache<T> where T : class
{
    // Implementation
}

// Constraint to value types
public class ValueProcessor<T> where T : struct
{
    // Implementation
}

// Constraint to specific base class
public class AnimalShelter<T> where T : Animal
{
    // Implementation
}

// Constraint to specific interface
public class Repository<T> where T : IEntity
{
    // Implementation
}

// Constraint to have a parameterless constructor
public class Factory<T> where T : new()
{
    public T Create()
    {
        return new T();
    }
}

// Multiple constraints
public class AdvancedProcessor<T> where T : class, IComparable<T>, new()
{
    // Implementation
}
```

Available constraints:

| Constraint | Description |
|------------|-------------|
| `where T : struct` | T must be a value type |
| `where T : class` | T must be a reference type |
| `where T : notnull` | T must be a non-nullable type |
| `where T : unmanaged` | T must be an unmanaged type |
| `where T : new()` | T must have a public parameterless constructor |
| `where T : <base class>` | T must be or derive from the specified base class |
| `where T : <interface>` | T must implement the specified interface |
| `where T : U` | T must be or derive from the type parameter U |

### Generic Interfaces

Interfaces can also be generic, allowing for type-safe contracts:

```csharp
// Generic interface
public interface IRepository<T>
{
    T GetById(int id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(int id);
}

// Implementation for a specific type
public class CustomerRepository : IRepository<Customer>
{
    public Customer GetById(int id)
    {
        // Implementation
        return new Customer { Id = id };
    }
    
    public IEnumerable<Customer> GetAll()
    {
        // Implementation
        return new List<Customer>();
    }
    
    public void Add(Customer entity)
    {
        // Implementation
    }
    
    public void Update(Customer entity)
    {
        // Implementation
    }
    
    public void Delete(int id)
    {
        // Implementation
    }
}
```

### Generic Delegates

Delegates can be defined with generic type parameters:

```csharp
// Generic delegate
public delegate TResult Transformer<T, TResult>(T input);

// Usage
Transformer<string, int> stringToInt = str => int.Parse(str);
int result = stringToInt("42"); // 42

// Built-in generic delegates
Func<string, int> converter = str => int.Parse(str);
Action<string> printer = s => Console.WriteLine(s);
Predicate<int> isEven = n => n % 2 == 0;
```

## Advanced Generics

### Generic Type Inference

C# can often infer generic type arguments, making your code cleaner:

```csharp
// Method with type inference
public static List<T> CreateList<T>(params T[] items)
{
    return new List<T>(items);
}

// Compiler infers T as string
var strings = CreateList("one", "two", "three");

// Compiler infers T as int
var numbers = CreateList(1, 2, 3);
```

### Covariance and Contravariance

Covariance and contravariance provide flexibility when working with generic interfaces and delegates:

**Covariance (`out`)**: Allows a more derived type to be used than originally specified.
**Contravariance (`in`)**: Allows a less derived type to be used than originally specified.

```csharp
// Covariant interface (out T)
public interface IEnumerable<out T>
{
    IEnumerator<T> GetEnumerator();
}

// Contravariant interface (in T)
public interface IComparer<in T>
{
    int Compare(T x, T y);
}

// Example of covariance
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings; // Valid due to covariance

// Example of contravariance
IComparer<object> objectComparer = new ObjectComparer();
IComparer<string> stringComparer = objectComparer; // Valid due to contravariance

// Covariance with delegates
Func<Animal> animalFactory = () => new Animal();
Func<object> objectFactory = animalFactory; // Valid - covariant return type

// Contravariance with delegates
Action<object> objectAction = obj => Console.WriteLine(obj);
Action<string> stringAction = objectAction; // Valid - contravariant parameter type
```

### Default Values for Type Parameters

The `default` keyword can be used to get the default value of a type parameter:

```csharp
public static T GetDefaultValue<T>()
{
    return default(T); // Returns null for reference types, 0 for numeric types, etc.
}

// C# 7.1+ supports simplified default literal
public static T GetDefaultValueSimplified<T>()
{
    return default; // Same effect as default(T)
}

int defaultInt = GetDefaultValue<int>(); // 0
string defaultString = GetDefaultValue<string>(); // null
DateTime defaultDate = GetDefaultValue<DateTime>(); // 01/01/0001 00:00:00
```

### Generic Type Constraints with Inheritance

Type constraints can be used with inheritance hierarchies:

```csharp
public abstract class Entity
{
    public int Id { get; set; }
}

public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    string CreatedBy { get; set; }
}

// Generic repository with auditing capability
public class AuditableRepository<T> where T : Entity, IAuditable
{
    public void Save(T entity)
    {
        if (entity.Id == 0)
        {
            // New entity
            entity.CreatedAt = DateTime.UtcNow;
            entity.CreatedBy = Thread.CurrentPrincipal?.Identity?.Name ?? "System";
        }
        
        // Save to database
    }
}

// Usage
public class Customer : Entity, IAuditable
{
    public string Name { get; set; }
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }
}

var repo = new AuditableRepository<Customer>();
repo.Save(new Customer { Name = "John Doe" });
```

### Generic Constraints with Value Types and Reference Types

Constraints allow you to take advantage of specific type characteristics:

```csharp
// Value type constraint allows direct operations on value types
public static T Increment<T>(T value) where T : struct, IIncrement<T>
{
    return value.Increment();
}

// Reference type constraint allows checking for null
public static void ProcessIfNotNull<T>(T item, Action<T> processor) where T : class
{
    if (item != null)
    {
        processor(item);
    }
}
```

### Static Fields in Generic Types

Each closed generic type has its own set of static fields:

```csharp
public class Counter<T>
{
    // Each closed generic type gets its own static field
    private static int _instances = 0;
    
    public Counter()
    {
        _instances++;
    }
    
    public static int InstanceCount => _instances;
}

// Usage
var stringCounter1 = new Counter<string>();
var stringCounter2 = new Counter<string>();
var intCounter = new Counter<int>();

Console.WriteLine(Counter<string>.InstanceCount); // 2
Console.WriteLine(Counter<int>.InstanceCount); // 1
```

### Generic Type Definitions and Closed Generic Types

A distinction exists between generic type definitions and closed generic types:

```csharp
// Type t1 represents the generic type definition
Type t1 = typeof(List<>);
Console.WriteLine(t1.Name); // "List`1"

// Type t2 represents a closed generic type
Type t2 = typeof(List<string>);
Console.WriteLine(t2.Name); // "List`1"

// Creating instances using reflection
Type listType = typeof(List<>);
Type closedListType = listType.MakeGenericType(typeof(int));
object listInstance = Activator.CreateInstance(closedListType); // Creates List<int>
```

## Implementing Common Generic Patterns

### Generic Repository Pattern

A common pattern for data access:

```csharp
// Entity base class
public abstract class Entity
{
    public int Id { get; set; }
}

// Generic repository interface
public interface IRepository<T> where T : Entity
{
    T GetById(int id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
    void SaveChanges();
}

// Generic repository implementation
public class Repository<T> : IRepository<T> where T : Entity
{
    protected readonly DbContext _context;
    protected readonly DbSet<T> _dbSet;
    
    public Repository(DbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }
    
    public T GetById(int id)
    {
        return _dbSet.Find(id);
    }
    
    public IEnumerable<T> GetAll()
    {
        return _dbSet.ToList();
    }
    
    public void Add(T entity)
    {
        _dbSet.Add(entity);
    }
    
    public void Update(T entity)
    {
        _dbSet.Attach(entity);
        _context.Entry(entity).State = EntityState.Modified;
    }
    
    public void Delete(T entity)
    {
        if (_context.Entry(entity).State == EntityState.Detached)
        {
            _dbSet.Attach(entity);
        }
        _dbSet.Remove(entity);
    }
    
    public void SaveChanges()
    {
        _context.SaveChanges();
    }
}

// Usage
public class Customer : Entity
{
    public string Name { get; set; }
    public string Email { get; set; }
}

public class Order : Entity
{
    public DateTime OrderDate { get; set; }
    public decimal Amount { get; set; }
    public int CustomerId { get; set; }
    public Customer Customer { get; set; }
}

// In application code
var customerRepo = new Repository<Customer>(dbContext);
var orderRepo = new Repository<Order>(dbContext);
```

### Generic Factory Pattern

Creating objects of different types at runtime:

```csharp
// Generic factory interface
public interface IFactory<T>
{
    T Create();
}

// Generic factory implementation using Activator
public class ActivatorFactory<T> : IFactory<T> where T : new()
{
    public T Create()
    {
        return new T();
    }
}

// Factory using lambda expressions
public class DelegateFactory<T> : IFactory<T>
{
    private readonly Func<T> _creator;
    
    public DelegateFactory(Func<T> creator)
    {
        _creator = creator ?? throw new ArgumentNullException(nameof(creator));
    }
    
    public T Create()
    {
        return _creator();
    }
}

// Usage
var customerFactory = new ActivatorFactory<Customer>();
var customer = customerFactory.Create();

var orderFactory = new DelegateFactory<Order>(() => new Order { OrderDate = DateTime.UtcNow });
var order = orderFactory.Create();
```

### Generic Builder Pattern

Builder pattern with fluent interface for constructing complex objects:

```csharp
// Generic builder
public class Builder<T> where T : new()
{
    private readonly T _instance = new T();
    private readonly Dictionary<string, Action<object>> _propertySetters = new Dictionary<string, Action<object>>();
    
    public Builder<T> With<TProperty>(Expression<Func<T, TProperty>> propertySelector, TProperty value)
    {
        var memberExpression = propertySelector.Body as MemberExpression;
        if (memberExpression == null)
        {
            throw new ArgumentException("Selector must be a member expression", nameof(propertySelector));
        }
        
        var propertyInfo = memberExpression.Member as PropertyInfo;
        if (propertyInfo == null)
        {
            throw new ArgumentException("Selector must be a property", nameof(propertySelector));
        }
        
        string propertyName = propertyInfo.Name;
        
        _propertySetters[propertyName] = propValue => propertyInfo.SetValue(_instance, propValue);
        _propertySetters[propertyName](value);
        
        return this;
    }
    
    public T Build()
    {
        return _instance;
    }
}

// Usage
var customer = new Builder<Customer>()
    .With(c => c.Name, "John Doe")
    .With(c => c.Email, "john.doe@example.com")
    .Build();
```

### Generic Specification Pattern

A pattern for encapsulating query logic:

```csharp
// Generic specification interface
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
    Expression<Func<T, bool>> ToExpression();
}

// Abstract base specification
public abstract class Specification<T> : ISpecification<T>
{
    public bool IsSatisfiedBy(T entity)
    {
        Func<T, bool> predicate = ToExpression().Compile();
        return predicate(entity);
    }
    
    public abstract Expression<Func<T, bool>> ToExpression();
    
    public static Specification<T> operator &(Specification<T> left, Specification<T> right)
    {
        return new AndSpecification<T>(left, right);
    }
    
    public static Specification<T> operator |(Specification<T> left, Specification<T> right)
    {
        return new OrSpecification<T>(left, right);
    }
    
    public static Specification<T> operator !(Specification<T> specification)
    {
        return new NotSpecification<T>(specification);
    }
}

// Concrete specifications
public class AndSpecification<T> : Specification<T>
{
    private readonly ISpecification<T> _left;
    private readonly ISpecification<T> _right;
    
    public AndSpecification(ISpecification<T> left, ISpecification<T> right)
    {
        _left = left;
        _right = right;
    }
    
    public override Expression<Func<T, bool>> ToExpression()
    {
        Expression<Func<T, bool>> leftExpression = _left.ToExpression();
        Expression<Func<T, bool>> rightExpression = _right.ToExpression();
        
        var parameter = Expression.Parameter(typeof(T));
        
        var body = Expression.AndAlso(
            Expression.Invoke(leftExpression, parameter),
            Expression.Invoke(rightExpression, parameter)
        );
        
        return Expression.Lambda<Func<T, bool>>(body, parameter);
    }
}

public class OrSpecification<T> : Specification<T>
{
    private readonly ISpecification<T> _left;
    private readonly ISpecification<T> _right;
    
    public OrSpecification(ISpecification<T> left, ISpecification<T> right)
    {
        _left = left;
        _right = right;
    }
    
    public override Expression<Func<T, bool>> ToExpression()
    {
        Expression<Func<T, bool>> leftExpression = _left.ToExpression();
        Expression<Func<T, bool>> rightExpression = _right.ToExpression();
        
        var parameter = Expression.Parameter(typeof(T));
        
        var body = Expression.OrElse(
            Expression.Invoke(leftExpression, parameter),
            Expression.Invoke(rightExpression, parameter)
        );
        
        return Expression.Lambda<Func<T, bool>>(body, parameter);
    }
}

public class NotSpecification<T> : Specification<T>
{
    private readonly ISpecification<T> _specification;
    
    public NotSpecification(ISpecification<T> specification)
    {
        _specification = specification;
    }
    
    public override Expression<Func<T, bool>> ToExpression()
    {
        Expression<Func<T, bool>> expression = _specification.ToExpression();
        var parameter = Expression.Parameter(typeof(T));
        
        var body = Expression.Not(
            Expression.Invoke(expression, parameter)
        );
        
        return Expression.Lambda<Func<T, bool>>(body, parameter);
    }
}

// Example specifications
public class PremiumCustomerSpecification : Specification<Customer>
{
    public override Expression<Func<Customer, bool>> ToExpression()
    {
        return customer => customer.TotalPurchases > 10000;
    }
}

public class RecentCustomerSpecification : Specification<Customer>
{
    public override Expression<Func<Customer, bool>> ToExpression()
    {
        var threeMonthsAgo = DateTime.UtcNow.AddMonths(-3);
        return customer => customer.JoinDate >= threeMonthsAgo;
    }
}

// Usage
var premiumSpec = new PremiumCustomerSpecification();
var recentSpec = new RecentCustomerSpecification();

var premiumOrRecentSpec = premiumSpec | recentSpec;
var premiumAndRecentSpec = premiumSpec & recentSpec;
var notPremiumSpec = !premiumSpec;

// Apply specification to query
IQueryable<Customer> customers = dbContext.Customers;
var filteredCustomers = customers.Where(premiumAndRecentSpec.ToExpression());
```

## Performance Considerations

### Boxing and Unboxing

One of the key benefits of generics is avoiding boxing and unboxing operations:

```csharp
// Before generics - using non-generic collections
ArrayList list = new ArrayList();
list.Add(42); // Boxing - int to object
int num = (int)list.Add(0); // Unboxing - object to int

// With generics - no boxing/unboxing
List<int> genericList = new List<int>();
genericList.Add(42); // No boxing
int number = genericList[0]; // No unboxing
```

### JIT Compilation and Type Specialization

The JIT compiler creates specialized code for each closed generic type:

- For reference types, the JIT compiler can often use the same code since all reference types are handled similarly (reference to memory)
- For value types, the JIT compiler creates specialized versions optimized for each specific type

This results in code that's as efficient as hand-written type-specific implementations.

### Memory Usage

Be aware of the memory impact of generics:

```csharp
// Each closed generic type with a value type parameter 
// creates a specialized version in memory
List<int> intList = new List<int>();
List<double> doubleList = new List<double>();
List<DateTime> dateList = new List<DateTime>();
// Three different specialized implementations

// Reference types typically share code
List<string> stringList = new List<string>();
List<Customer> customerList = new List<Customer>();
// These often share the same implementation code with type handles
```

### Value Type Constraints for Performance

Using the `struct` constraint can help the compiler optimize code for value types:

```csharp
// No constraint - must handle both reference and value types
public T Default<T>()
{
    return default(T);
}

// With struct constraint - compiler knows T is a value type
public T DefaultValueType<T>() where T : struct
{
    return default(T);
}

// With class constraint - compiler knows T is a reference type
public T DefaultReferenceType<T>() where T : class
{
    return default(T); // Always returns null
}
```

### `in`, `out`, and `ref` Parameters with Generics

Generic methods with value type parameters can benefit from `in`, `out`, and `ref` keywords:

```csharp
// Pass large value types by reference to avoid copying
public void ProcessLargeStruct<T>(in T value) where T : struct
{
    // 'value' is passed by reference but cannot be modified
    Console.WriteLine(value);
}

// Output parameter
public void CreateNew<T>(out T result) where T : new()
{
    result = new T();
}

// Reference parameter for modification
public void Increment<T>(ref T value) where T : struct, IIncrementable<T>
{
    value = value.Increment();
}
```

## Common Pitfalls

### Overconstraining Type Parameters

```csharp
// Overly constrained - limits flexibility
public class Repository<T> 
    where T : Entity, IEquatable<T>, IComparable<T>, new()
{
    // Implementation
}

// Better approach - only constrain what's necessary
public class Repository<T> where T : Entity
{
    // Implementation
}
```

### Ignoring Variance

```csharp
// Without variance annotations
IEnumerable<Animal> animals = new List<Dog>(); // Error without covariance
IComparer<object> objectComparer = new StringComparer(); // Error without contravariance

// With proper variance annotations
IEnumerable<Animal> animals = new List<Dog>(); // OK with covariance (out T)
IComparer<string> stringComparer = new ObjectComparer(); // OK with contravariance (in T)
```

### Generic Type Parameter Naming Conventions

```csharp
// Poor naming - unclear what T represents
public class Repository<T>
{
    // Implementation
}

// Better naming - indicates what T represents
public class Repository<TEntity>
{
    // Implementation
}

// Multiple type parameters with meaningful names
public class Converter<TInput, TOutput>
{
    // Implementation
}

// Common conventions:
// T - single type parameter
// TKey, TValue - key-value pairs
// TResult - return type
// TInput, TOutput - conversion
// TEntity - entity type
```

### Excessive Generic Abstraction

```csharp
// Excessive abstraction - hard to understand and use
public class ProcessingEngine<TContext, TEntity, TResult, TProcessor, TValidator>
    where TContext : IProcessingContext<TEntity>
    where TEntity : IEntity
    where TResult : IResult
    where TProcessor : IProcessor<TEntity, TResult>
    where TValidator : IValidator<TEntity>
{
    // Implementation
}

// Simplified - more understandable
public class ProcessingEngine<TEntity> where TEntity : IEntity
{
    // Implementation
}
```

### Static Implementation Leaks

```csharp
// Problematic - leaks state between different generic instances
public class Cache<T>
{
    // Same cache for all T
    private static Dictionary<string, object> _internalCache = new Dictionary<string, object>();
    
    public void Add(string key, T value)
    {
        _internalCache[key] = value;
    }
    
    public T Get(string key)
    {
        return (T)_internalCache[key];
    }
}

// Better approach - separate cache per T
public class BetterCache<T>
{
    // Different cache for each closed generic type
    private static Dictionary<string, T> _cache = new Dictionary<string, T>();
    
    public void Add(string key, T value)
    {
        _cache[key] = value;
    }
    
    public T Get(string key)
    {
        return _cache[key];
    }
}
```

## Generic Collections and .NET Framework

### Built-in Generic Collections

.NET provides many generic collections in the `System.Collections.Generic` namespace:

```csharp
// List<T> - dynamic array
List<string> names = new List<string>();
names.Add("Alice");
names.Add("Bob");
names.Remove("Alice");
bool containsBob = names.Contains("Bob"); // true

// Dictionary<TKey, TValue> - key-value pairs
Dictionary<string, int> ages = new Dictionary<string, int>();
ages["Alice"] = 30;
ages["Bob"] = 25;
int aliceAge = ages["Alice"]; // 30

// HashSet<T> - unique elements, fast lookups
HashSet<int> uniqueNumbers = new HashSet<int>();
uniqueNumbers.Add(1);
uniqueNumbers.Add(2);
uniqueNumbers.Add(1); // Ignored, already exists
bool containsOne = uniqueNumbers.Contains(1); // true

// Queue<T> - FIFO collection
Queue<string> tasks = new Queue<string>();
tasks.Enqueue("Task 1");
tasks.Enqueue("Task 2");
string nextTask = tasks.Dequeue(); // "Task 1"

// Stack<T> - LIFO collection
Stack<int> stack = new Stack<int>();
stack.Push(1);
stack.Push(2);
int topItem = stack.Pop(); // 2

// LinkedList<T> - doubly linked list
LinkedList<string> linkedList = new LinkedList<string>();
linkedList.AddLast("Last");
linkedList.AddFirst("First");
linkedList.AddAfter(linkedList.First, "Middle");

// SortedList<TKey, TValue> - sorted key-value pairs
SortedList<string, int> sortedAges = new SortedList<string, int>();
sortedAges["Charlie"] = 35;
sortedAges["Alice"] = 30;
sortedAges["Bob"] = 25;
// Keys are automatically sorted: Alice, Bob, Charlie

// SortedSet<T> - sorted unique elements
SortedSet<int> sortedNumbers = new SortedSet<int>();
sortedNumbers.Add(3);
sortedNumbers.Add(1);
sortedNumbers.Add(2);
// Contains 1, 2, 3 in order
```

### Creating Custom Generic Collections

Creating your own generic collection by inheriting from existing ones:

```csharp
// Simple thread-safe list
public class ThreadSafeList<T> : IList<T>
{
    private readonly List<T> _innerList = new List<T>();
    private readonly object _syncRoot = new object();
    
    public T this[int index]
    {
        get
        {
            lock (_syncRoot)
            {
                return _innerList[index];
            }
        }
        set
        {
            lock (_syncRoot)
            {
                _innerList[index] = value;
            }
        }
    }
    
    public int Count
    {
        get
        {
            lock (_syncRoot)
            {
                return _innerList.Count;
            }
        }
    }
    
    public bool IsReadOnly => false;
    
    public void Add(T item)
    {
        lock (_syncRoot)
        {
            _innerList.Add(item);
        }
    }
    
    public void Clear()
    {
        lock (_syncRoot)
        {
            _innerList.Clear();
        }
    }
    
    public bool Contains(T item)
    {
        lock (_syncRoot)
        {
            return _innerList.Contains(item);
        }
    }
    
    public void CopyTo(T[] array, int arrayIndex)
    {
        lock (_syncRoot)
        {
            _innerList.CopyTo(array, arrayIndex);
        }
    }
    
    public IEnumerator<T> GetEnumerator()
    {
        lock (_syncRoot)
        {
            // Create a copy to avoid enumeration during modification exceptions
            return new List<T>(_innerList).GetEnumerator();
        }
    }
    
    public int IndexOf(T item)
    {
        lock (_syncRoot)
        {
            return _innerList.IndexOf(item);
        }
    }
    
    public void Insert(int index, T item)
    {
        lock (_syncRoot)
        {
            _innerList.Insert(index, item);
        }
    }
    
    public bool Remove(T item)
    {
        lock (_syncRoot)
        {
            return _innerList.Remove(item);
        }
    }
    
    public void RemoveAt(int index)
    {
        lock (_syncRoot)
        {
            _innerList.RemoveAt(index);
        }
    }
    
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }
}
```

### Implementing IEnumerable<T>

Creating an entirely custom collection by implementing `IEnumerable<T>`:

```csharp
// Custom binary tree implementation
public class BinaryTree<T> : IEnumerable<T> where T : IComparable<T>
{
    private class Node
    {
        public T Value { get; set; }
        public Node Left { get; set; }
        public Node Right { get; set; }
        
        public Node(T value)
        {
            Value = value;
        }
    }
    
    private Node _root;
    
    public void Add(T value)
    {
        if (_root == null)
        {
            _root = new Node(value);
            return;
        }
        
        AddToNode(_root, value);
    }
    
    private void AddToNode(Node node, T value)
    {
        if (value.CompareTo(node.Value) < 0)
        {
            if (node.Left == null)
            {
                node.Left = new Node(value);
            }
            else
            {
                AddToNode(node.Left, value);
            }
        }
        else
        {
            if (node.Right == null)
            {
                node.Right = new Node(value);
            }
            else
            {
                AddToNode(node.Right, value);
            }
        }
    }
    
    public IEnumerator<T> GetEnumerator()
    {
        // In-order traversal
        return InOrderTraversal(_root).GetEnumerator();
    }
    
    private IEnumerable<T> InOrderTraversal(Node node)
    {
        if (node != null)
        {
            // Traverse left
            foreach (var value in InOrderTraversal(node.Left))
            {
                yield return value;
            }
            
            // Visit node
            yield return node.Value;
            
            // Traverse right
            foreach (var value in InOrderTraversal(node.Right))
            {
                yield return value;
            }
        }
    }
    
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }
}

// Usage
var tree = new BinaryTree<int>();
tree.Add(5);
tree.Add(3);
tree.Add(7);
tree.Add(2);
tree.Add(4);

// Iterate in order (2, 3, 4, 5, 7)
foreach (var value in tree)
{
    Console.WriteLine(value);
}
```

## Best Practices

### When to Use Generics

Generics are particularly useful when:

1. You need to reuse code across different data types
2. You want type safety without the overhead of boxing/unboxing
3. You need to implement type-specific algorithms or data structures
4. You want to enforce contracts on types (via constraints)

### Design Guidelines

- **Keep it simple**: Don't overuse generics or add unnecessary complexity
- **Use meaningful names**: Clear type parameter names improve readability
- **Add constraints appropriately**: Use constraints to express requirements on type parameters, but avoid over-constraining
- **Consider variance**: Use `in` and `out` annotations where appropriate
- **Document expectations**: Clear documentation on how generic types should be used
- **Follow established patterns**: Study the .NET Framework collection classes for guidance

### Testing Generic Code

Testing generic code presents unique challenges:

```csharp
// Test with multiple type arguments
[Fact]
public void GenericList_CanAddAndRetrieveItems_ForReferenceTypes()
{
    // Test with a reference type
    var stringList = new GenericList<string>();
    stringList.Add("test");
    Assert.Equal("test", stringList[0]);
}

[Fact]
public void GenericList_CanAddAndRetrieveItems_ForValueTypes()
{
    // Test with a value type
    var intList = new GenericList<int>();
    intList.Add(42);
    Assert.Equal(42, intList[0]);
}

// Test with corner cases
[Fact]
public void GenericList_WorksWithEdgeCases()
{
    // Test with nullable types
    var nullableList = new GenericList<int?>();
    nullableList.Add(null);
    Assert.Null(nullableList[0]);
    
    // Test with custom types
    var customList = new GenericList<TestClass>();
    customList.Add(new TestClass { Id = 1 });
    Assert.Equal(1, customList[0].Id);
}
```

## Real-World Examples

### Generic Service Locator

A pattern for dependency resolution:

```csharp
public class ServiceLocator
{
    private readonly Dictionary<Type, object> _services = new Dictionary<Type, object>();
    
    public void Register<T>(T service)
    {
        _services[typeof(T)] = service;
    }
    
    public T Resolve<T>()
    {
        if (_services.TryGetValue(typeof(T), out var service))
        {
            return (T)service;
        }
        
        throw new InvalidOperationException($"Service of type {typeof(T).Name} not registered");
    }
    
    public bool TryResolve<T>(out T service)
    {
        if (_services.TryGetValue(typeof(T), out var serviceObj))
        {
            service = (T)serviceObj;
            return true;
        }
        
        service = default;
        return false;
    }
}

// Usage
var locator = new ServiceLocator();
locator.Register<ILogger>(new ConsoleLogger());
locator.Register<IRepository<Customer>>(new CustomerRepository());

var logger = locator.Resolve<ILogger>();
var repository = locator.Resolve<IRepository<Customer>>();
```

### Generic Unit of Work

Managing transactional operations:

```csharp
public interface IUnitOfWork : IDisposable
{
    IRepository<TEntity> GetRepository<TEntity>() where TEntity : class, IEntity;
    Task<int> SaveChangesAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly DbContext _context;
    private readonly Dictionary<Type, object> _repositories = new Dictionary<Type, object>();
    private bool _disposed = false;
    
    public UnitOfWork(DbContext context)
    {
        _context = context;
    }
    
    public IRepository<TEntity> GetRepository<TEntity>() where TEntity : class, IEntity
    {
        var type = typeof(TEntity);
        
        if (!_repositories.ContainsKey(type))
        {
            _repositories[type] = new Repository<TEntity>(_context);
        }
        
        return (IRepository<TEntity>)_repositories[type];
    }
    
    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                _context.Dispose();
            }
            
            _repositories.Clear();
            _disposed = true;
        }
    }
}

// Usage
using (var unitOfWork = new UnitOfWork(dbContext))
{
    var customerRepo = unitOfWork.GetRepository<Customer>();
    var orderRepo = unitOfWork.GetRepository<Order>();
    
    var customer = await customerRepo.GetByIdAsync(customerId);
    var order = new Order
    {
        CustomerId = customer.Id,
        OrderDate = DateTime.UtcNow
    };
    
    await orderRepo.AddAsync(order);
    await unitOfWork.SaveChangesAsync();
}
```

### Generic Result Type

A pattern for standardized method results:

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public T Value { get; }
    public string Error { get; }
    public bool IsFailure => !IsSuccess;
    
    protected Result(bool isSuccess, T value, string error)
    {
        if (isSuccess && error != null)
            throw new InvalidOperationException("Cannot have error message for success result");
            
        if (!isSuccess && error == null)
            throw new InvalidOperationException("Must have error message for failure result");
            
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }
    
    public static Result<T> Success(T value) => new Result<T>(true, value, null);
    public static Result<T> Failure(string error) => new Result<T>(false, default, error);
    
    public Result<TNew> Map<TNew>(Func<T, TNew> mapper)
    {
        return IsSuccess
            ? Result<TNew>.Success(mapper(Value))
            : Result<TNew>.Failure(Error);
    }
    
    public async Task<Result<TNew>> MapAsync<TNew>(Func<T, Task<TNew>> mapper)
    {
        return IsSuccess
            ? Result<TNew>.Success(await mapper(Value))
            : Result<TNew>.Failure(Error);
    }
    
    public Result<TNew> Bind<TNew>(Func<T, Result<TNew>> func)
    {
        return IsSuccess ? func(Value) : Result<TNew>.Failure(Error);
    }
}

// Usage
public async Task<Result<OrderDto>> PlaceOrderAsync(OrderRequest request)
{
    // Validate
    if (request.Items.Count == 0)
    {
        return Result<OrderDto>.Failure("Order must contain at least one item");
    }
    
    // Process
    try
    {
        var customer = await _customerRepository.GetByIdAsync(request.CustomerId);
        if (customer == null)
        {
            return Result<OrderDto>.Failure("Customer not found");
        }
        
        var order = new Order
        {
            CustomerId = customer.Id,
            OrderDate = DateTime.UtcNow,
            Items = request.Items.Select(i => new OrderItem { ... }).ToList()
        };
        
        await _orderRepository.AddAsync(order);
        await _unitOfWork.SaveChangesAsync();
        
        return Result<OrderDto>.Success(new OrderDto
        {
            Id = order.Id,
            CustomerName = customer.Name,
            TotalAmount = order.CalculateTotal(),
            OrderDate = order.OrderDate
        });
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error placing order");
        return Result<O
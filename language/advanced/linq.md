---
title: "LINQ (Language Integrated Query)"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["c#", "linq", "queries", "collections", "data", "advanced"]
difficulty: "intermediate"
---

# LINQ (Language Integrated Query)

## Overview

Language Integrated Query (LINQ) is a set of features introduced in C# 3.0 that adds powerful query capabilities to the .NET language syntax. LINQ provides a consistent query experience across different data sources including collections, XML, relational databases, and third-party data sources. It bridges the gap between the world of objects and the world of data, enabling developers to write type-safe queries using familiar language constructs.

This document explores LINQ's capabilities, usage patterns, optimization techniques, and best practices for C# developers.

## Core Concepts

### LINQ Providers

LINQ consists of a pattern that various data providers can implement:

- **LINQ to Objects**: For querying in-memory collections like arrays, lists, and dictionaries
- **LINQ to XML**: For querying and manipulating XML data
- **LINQ to Entities (Entity Framework)**: For querying relational databases
- **LINQ to SQL**: Microsoft's older ORM for SQL Server (largely replaced by Entity Framework)
- **Third-party providers**: For various data sources like JSON, CSV, MongoDB, etc.

### Query Syntax vs Method Syntax

LINQ offers two equivalent syntaxes:

#### Query Syntax (Similar to SQL)

```csharp
// Query syntax
var youngDevelopers = from developer in developers
                     where developer.Age < 30
                     orderby developer.Experience descending
                     select new { developer.Name, developer.Age };
```

#### Method Syntax (Uses Extension Methods)

```csharp
// Method syntax (equivalent to above)
var youngDevelopers = developers
    .Where(d => d.Age < 30)
    .OrderByDescending(d => d.Experience)
    .Select(d => new { d.Name, d.Age });
```

### Deferred Execution

Most LINQ queries are not executed when they are defined but when they are iterated over:

```csharp
// Query definition - NOT executed here
var query = from num in numbers
           where num % 2 == 0
           select num;

// Query is executed here when iterating
foreach (var num in query)
{
    Console.WriteLine(num);
}

// Also executed when calling these methods
int count = query.Count();
List<int> results = query.ToList();
int[] array = query.ToArray();
```

### IEnumerable<T> and IQueryable<T>

- `IEnumerable<T>`: Used for in-memory data (LINQ to Objects)
- `IQueryable<T>`: Used for remote data sources (LINQ to Entities, LINQ to SQL)

```csharp
// IEnumerable<T> example - filtering happens in memory
IEnumerable<Customer> customers = dbContext.Customers.AsEnumerable();
var filteredCustomers = customers.Where(c => c.Name.Contains("Smith"));

// IQueryable<T> example - filtering happens at the database
IQueryable<Customer> customers = dbContext.Customers;
var filteredCustomers = customers.Where(c => c.Name.Contains("Smith"));
```

## Basic LINQ Operations

### Filtering

```csharp
// Where - filters elements
var adults = people.Where(p => p.Age >= 18);

// First/FirstOrDefault - returns first matching element
var john = people.First(p => p.Name == "John"); // Throws if not found
var jane = people.FirstOrDefault(p => p.Name == "Jane"); // Returns null if not found

// Single/SingleOrDefault - ensures exactly one match
var uniquePerson = people.Single(p => p.Id == 1234); // Throws if not found or multiple matches
var uniqueResult = people.SingleOrDefault(p => p.Id == 1234); // Returns null if not found, throws if multiple

// Last/LastOrDefault - returns last matching element
var lastAdult = people.Last(p => p.Age >= 18);
var lastChild = people.LastOrDefault(p => p.Age < 18);

// Skip and Take - pagination
var secondPage = people.Skip(10).Take(10); // items 11-20
```

### Projection

```csharp
// Select - transforms elements
var names = people.Select(p => p.Name);

// Anonymous types
var nameAndAge = people.Select(p => new { p.Name, p.Age });

// SelectMany - flattens nested collections
var allPhoneNumbers = customers.SelectMany(c => c.PhoneNumbers);

// Nested queries
var ordersWithCustomers = from c in customers
                         select new
                         {
                             Customer = c,
                             Orders = from o in orders
                                     where o.CustomerId == c.Id
                                     select o
                         };
```

### Sorting

```csharp
// OrderBy - ascending sort
var ascendingByAge = people.OrderBy(p => p.Age);

// OrderByDescending - descending sort
var descendingByAge = people.OrderByDescending(p => p.Age);

// ThenBy/ThenByDescending - secondary sorting
var sortedPeople = people
    .OrderBy(p => p.LastName)
    .ThenBy(p => p.FirstName);

// Reverse
var reversedList = people.Reverse();
```

### Grouping

```csharp
// GroupBy - creates groups of elements
var peopleByAge = people.GroupBy(p => p.Age);

// Iterating through groups
foreach (var ageGroup in peopleByAge)
{
    Console.WriteLine($"Age: {ageGroup.Key}");
    foreach (var person in ageGroup)
    {
        Console.WriteLine($"  {person.Name}");
    }
}

// Creating a dictionary from groups
var peopleByAgeDict = people.GroupBy(p => p.Age).ToDictionary(g => g.Key, g => g.ToList());

// GroupBy with element transformation
var countByAge = people
    .GroupBy(p => p.Age)
    .Select(g => new { Age = g.Key, Count = g.Count() });
```

### Joining

```csharp
// Join - inner join
var customerOrders = customers.Join(
    orders,
    customer => customer.Id,
    order => order.CustomerId,
    (customer, order) => new { Customer = customer, Order = order }
);

// GroupJoin - left outer join with grouping
var customersWithOrders = customers.GroupJoin(
    orders,
    customer => customer.Id,
    order => order.CustomerId,
    (customer, customerOrders) => new { Customer = customer, Orders = customerOrders }
);

// Query syntax for join
var query = from c in customers
           join o in orders on c.Id equals o.CustomerId
           select new { CustomerName = c.Name, OrderDate = o.OrderDate };

// Query syntax for group join
var query = from c in customers
           join o in orders on c.Id equals o.CustomerId into customerOrders
           select new { Customer = c, Orders = customerOrders };
```

### Set Operations

```csharp
// Distinct - removes duplicates
var distinctAges = people.Select(p => p.Age).Distinct();

// Union - combines two sequences, removing duplicates
var allAges = group1Ages.Union(group2Ages);

// Intersect - elements in both sequences
var commonAges = group1Ages.Intersect(group2Ages);

// Except - elements in first sequence but not in second
var uniqueToGroup1 = group1Ages.Except(group2Ages);

// Concat - combines two sequences, preserving duplicates
var allPeople = employees.Concat(customers);
```

### Aggregation

```csharp
// Count/LongCount
int adultCount = people.Count(p => p.Age >= 18);

// Sum
decimal totalSalary = employees.Sum(e => e.Salary);

// Min/Max
int youngestAge = people.Min(p => p.Age);
int oldestAge = people.Max(p => p.Age);

// Average
double averageAge = people.Average(p => p.Age);

// Aggregate - custom accumulation
int sumOfSquares = numbers.Aggregate(0, (acc, n) => acc + n * n);

// String concatenation example
string allNames = people.Aggregate("People: ", 
    (str, p) => str + p.Name + ", ", 
    result => result.TrimEnd(',', ' '));
```

### Quantifiers

```csharp
// Any - true if any element satisfies condition
bool hasAdults = people.Any(p => p.Age >= 18);

// All - true if all elements satisfy condition
bool allAreAdults = people.All(p => p.Age >= 18);

// Contains - true if sequence contains specific element
bool containsNumber = numbers.Contains(42);
```

### Generation

```csharp
// Range - generates a sequence of integers
var numbers = Enumerable.Range(1, 10); // 1 to 10

// Repeat - generates a sequence with repeated value
var fiveZeros = Enumerable.Repeat(0, 5); // 0,0,0,0,0

// Empty - returns empty sequence of specified type
var emptyList = Enumerable.Empty<Customer>();
```

### Element Operations

```csharp
// ElementAt/ElementAtOrDefault
var thirdPerson = people.ElementAt(2);
var tenthOrDefault = people.ElementAtOrDefault(9); // null if index out of range

// DefaultIfEmpty - provides default value for empty sequences
var names = emptyList.DefaultIfEmpty(new Customer { Name = "N/A" }).Select(c => c.Name);
```

## Advanced LINQ Techniques

### Using Multiple Data Sources

```csharp
// Query that combines multiple data sources
var report = from employee in employees
            join department in departments on employee.DepartmentId equals department.Id
            join manager in employees on department.ManagerId equals manager.Id
            select new
            {
                EmployeeName = employee.Name,
                DepartmentName = department.Name,
                ManagerName = manager.Name
            };
```

### Nested Queries

```csharp
// Query with subquery in where clause
var popularProducts = from p in products
                    where (from o in orders
                          from i in o.Items
                          where i.ProductId == p.Id
                          select i).Count() > 10
                    select p;

// Equivalent method syntax
var popularProducts = products
    .Where(p => orders.SelectMany(o => o.Items)
                     .Count(i => i.ProductId == p.Id) > 10);
```

### Custom Sequence Generation

```csharp
// Generating a Fibonacci sequence with LINQ
IEnumerable<int> Fibonacci(int count)
{
    return Enumerable
        .Range(0, count)
        .Select(n => n < 2 ? n : Fibonacci(n - 1).Last() + Fibonacci(n - 2).Last());
}

// More efficient implementation with an iterator method
IEnumerable<int> EfficientFibonacci(int count)
{
    int a = 0, b = 1;
    for (int i = 0; i < count; i++)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}
```

### Working with Nullable Types

```csharp
// Handle nullable types in LINQ
var validScores = students
    .Where(s => s.TestScore.HasValue)
    .Select(s => new { s.Name, Score = s.TestScore.Value });

// Alternative approach with null-conditional operator
var allScores = students
    .Select(s => new { s.Name, Score = s.TestScore ?? 0 });
```

### Let Keyword (Query Syntax)

```csharp
// Using 'let' to store intermediate results
var query = from person in people
           let fullName = $"{person.FirstName} {person.LastName}"
           let nameLength = fullName.Length
           where nameLength < 20
           orderby nameLength
           select new { FullName = fullName, Length = nameLength };
```

### Parallel LINQ (PLINQ)

```csharp
// Process items in parallel
var results = people.AsParallel()
    .Where(p => ExpensiveCheck(p))
    .Select(p => new { p.Name, Result = CalculateResult(p) })
    .ToList();

// With additional options
var customResults = people.AsParallel()
    .WithDegreeOfParallelism(4)
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
    .Where(p => ExpensiveCheck(p));
```

## LINQ to XML

### Creating XML

```csharp
// Creating XML elements
XElement person = new XElement("Person",
    new XElement("Name", "John Doe"),
    new XElement("Age", 30),
    new XElement("Email", "john.doe@example.com")
);

// Creating a document with declaration
XDocument document = new XDocument(
    new XDeclaration("1.0", "utf-8", "yes"),
    new XElement("People",
        new XElement("Person",
            new XAttribute("id", "1"),
            new XElement("Name", "Jane Smith"),
            new XElement("Age", 25)
        ),
        new XElement("Person",
            new XAttribute("id", "2"),
            new XElement("Name", "John Doe"),
            new XElement("Age", 30)
        )
    )
);

// Saving XML to file
document.Save("people.xml");
```

### Querying XML

```csharp
// Load XML from file
XDocument doc = XDocument.Load("people.xml");

// Basic XML queries
var names = from person in doc.Root.Elements("Person")
           select person.Element("Name").Value;

// Filtering XML
var adults = from person in doc.Root.Elements("Person")
            where int.Parse(person.Element("Age").Value) >= 18
            select new
            {
                Name = person.Element("Name").Value,
                Age = int.Parse(person.Element("Age").Value)
            };

// Working with attributes
var peopleWithId = from person in doc.Root.Elements("Person")
                  select new
                  {
                      Id = (string)person.Attribute("id"),
                      Name = (string)person.Element("Name")
                  };

// Descendants - searches entire tree
var allEmails = doc.Descendants("Email").Select(e => e.Value);

// XPath with LINQ to XML
var xpathResults = doc.XPathSelectElements("//Person[Age > 25]");
```

### Modifying XML

```csharp
// Adding elements
doc.Root.Add(
    new XElement("Person",
        new XAttribute("id", "3"),
        new XElement("Name", "Alice Brown"),
        new XElement("Age", 28)
    )
);

// Modifying elements
foreach (var person in doc.Root.Elements("Person"))
{
    if ((string)person.Element("Name") == "John Doe")
    {
        person.Element("Age").Value = "31";
    }
}

// Removing elements
doc.Root.Elements("Person")
    .Where(p => (string)p.Element("Name") == "Jane Smith")
    .Remove();

// Transforming XML
var transformed = new XElement("PeopleReport",
    from person in doc.Root.Elements("Person")
    select new XElement("PersonSummary",
        new XAttribute("name", (string)person.Element("Name")),
        new XAttribute("age", (string)person.Element("Age"))
    )
);
```

## LINQ to Entities (Entity Framework)

### Basic Entity Framework Queries

```csharp
// DbContext setup (simplified)
public class ApplicationDbContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<Product> Products { get; set; }
}

// Basic query
var youngCustomers = dbContext.Customers
    .Where(c => c.Age < 30)
    .OrderBy(c => c.LastName)
    .ToList();

// Query with joins
var customerOrders = dbContext.Customers
    .Include(c => c.Orders)
    .ThenInclude(o => o.OrderItems)
    .ThenInclude(i => i.Product)
    .Where(c => c.Orders.Any(o => o.TotalAmount > 1000))
    .ToList();

// Query with projection to DTO
var customerDtos = dbContext.Customers
    .Select(c => new CustomerDto
    {
        Id = c.Id,
        FullName = c.FirstName + " " + c.LastName,
        OrderCount = c.Orders.Count
    })
    .ToList();
```

### Tracking vs No-Tracking Queries

```csharp
// Tracking query (for updates)
var customer = dbContext.Customers.FirstOrDefault(c => c.Id == id);
customer.LastName = "Updated";
await dbContext.SaveChangesAsync();

// No-tracking query (for read-only scenarios, better performance)
var customers = dbContext.Customers
    .AsNoTracking()
    .Where(c => c.State == "CA")
    .ToList();
```

### Raw SQL with LINQ

```csharp
// FromSqlRaw with parameters
var customers = dbContext.Customers
    .FromSqlRaw("SELECT * FROM Customers WHERE State = {0}", state)
    .Where(c => c.Status == "Active")
    .ToList();

// FromSqlInterpolated (SQL injection safe)
var products = dbContext.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Category = {category}")
    .Where(p => p.IsAvailable)
    .ToList();
```

## Performance Optimization

### Optimizing LINQ to Objects

#### Avoiding Multiple Enumeration

```csharp
// Bad - enumerates twice
if (collection.Any() && collection.First().IsValid)
{
    // Do something
}

// Good - enumerates once
var first = collection.FirstOrDefault();
if (first != null && first.IsValid)
{
    // Do something
}
```

#### Materialize When Needed

```csharp
// Collection will be re-evaluated on each loop if not materialized
var query = collection.Where(item => ExpensiveOperation(item));

// For multiple uses, materialize once
var materializedList = query.ToList();
foreach (var item in materializedList)
{
    // Use item
}

// Again with materializedList - no re-evaluation
foreach (var item in materializedList)
{
    // Use item again
}
```

#### Use Specialized Methods

```csharp
// Less efficient
bool exists = collection.Where(x => x.Id == 42).Count() > 0;

// More efficient
bool exists = collection.Any(x => x.Id == 42);
```

### Optimizing LINQ to Entities

#### Eager Loading vs Lazy Loading

```csharp
// Eager loading - load related data in single query
var customers = dbContext.Customers
    .Include(c => c.Orders)
    .Where(c => c.Status == "Active")
    .ToList();

// Explicit loading - load related data on demand
var customer = dbContext.Customers.Find(id);
if (customer != null)
{
    dbContext.Entry(customer)
        .Collection(c => c.Orders)
        .Load();
}

// Lazy loading (requires virtual navigation properties)
// Loads each customer's orders when accessed
foreach (var customer in dbContext.Customers)
{
    foreach (var order in customer.Orders) // Separate query executed here
    {
        // Process order
    }
}
```

#### Projection for Specific Data

```csharp
// Inefficient - loads entire entities
var customers = dbContext.Customers
    .Where(c => c.Status == "Active")
    .ToList();
    
var customerNames = customers.Select(c => c.Name).ToList();

// Efficient - loads only needed data
var customerNames = dbContext.Customers
    .Where(c => c.Status == "Active")
    .Select(c => c.Name)
    .ToList();
```

#### Pagination

```csharp
// Efficient server-side pagination
var pagedCustomers = dbContext.Customers
    .OrderBy(c => c.LastName)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToList();
```

#### Avoiding N+1 Query Problem

```csharp
// Bad - N+1 query problem
foreach (var customer in dbContext.Customers)
{
    // One query per customer
    var orderCount = dbContext.Orders.Count(o => o.CustomerId == customer.Id);
}

// Good - single query
var customerWithOrderCounts = dbContext.Customers
    .Select(c => new 
    {
        Customer = c,
        OrderCount = c.Orders.Count
    })
    .ToList();
```

## Best Practices

### Readability and Maintainability

- **Choose the right syntax**: Use query syntax for complex queries with multiple operations, method syntax for simpler queries
- **Format for readability**: Break complex queries into multiple lines with proper indentation
- **Use meaningful variable names**: Name query variables to reflect their purpose
- **Extract complex queries**: Move complex queries to separate methods

```csharp
// Complex query extracted to a method
public IEnumerable<CustomerSummary> GetHighValueCustomers(int minimumOrderValue)
{
    return from c in dbContext.Customers
           where c.Orders.Any(o => o.TotalAmount > minimumOrderValue)
           let totalSpent = c.Orders.Sum(o => o.TotalAmount)
           orderby totalSpent descending
           select new CustomerSummary
           {
               Id = c.Id,
               Name = $"{c.FirstName} {c.LastName}",
               TotalSpent = totalSpent,
               OrderCount = c.Orders.Count
           };
}
```

### Error Handling

- **Use FirstOrDefault/SingleOrDefault**: Instead of First/Single to avoid exceptions
- **Check for null**: Always handle potential null returns from FirstOrDefault etc.
- **Defensive coding**: Consider empty collections with DefaultIfEmpty

```csharp
// Defensive LINQ usage
var customer = dbContext.Customers.FirstOrDefault(c => c.Id == id);
if (customer == null)
{
    return NotFound();
}

// Handling empty collections
var orderSummary = orders
    .DefaultIfEmpty()
    .Select(o => new
    {
        OrderId = o?.Id ?? 0,
        CustomerName = o?.Customer?.Name ?? "N/A",
        TotalAmount = o?.TotalAmount ?? 0
    });
```

### Performance Considerations

- **Think about execution time**: Understand when queries execute
- **Minimize database round trips**: Use Include for related data
- **Only retrieve what you need**: Use projection (Select) to limit data transfer
- **Use indexes appropriately**: Ensure database queries run on indexed columns
- **Consider caching**: Cache results of expensive or frequently used queries

```csharp
// Use caching for expensive queries
string cacheKey = $"CustomerOrders-{customerId}";
if (!_cache.TryGetValue(cacheKey, out List<Order> orders))
{
    orders = await dbContext.Orders
        .Where(o => o.CustomerId == customerId)
        .Include(o => o.Items)
        .ToListAsync();
        
    _cache.Set(cacheKey, orders, TimeSpan.FromMinutes(10));
}
```

## Common Pitfalls

### Deferred Execution Misunderstandings

```csharp
// Pitfall: Query is defined but never executed
var query = dbContext.Customers.Where(c => c.Status == "Active");
// Nothing happens, query not executed

// Pitfall: Multiple enumeration due to deferred execution
var query = dbContext.Customers.Where(c => c.Status == "Active");
foreach (var customer in query) { /* First execution */ }
foreach (var customer in query) { /* Second execution - another DB query */ }

// Solution: Materialize when needed
var customers = dbContext.Customers.Where(c => c.Status == "Active").ToList();
foreach (var customer in customers) { /* Using materialized results */ }
foreach (var customer in customers) { /* Reusing same results, no additional query */ }
```

### Translation Limitations

```csharp
// Pitfall: Methods that can't be translated to SQL
var customers = dbContext.Customers
    .Where(c => CustomMethod(c.Name)) // Error: cannot be translated
    .ToList();

// Solution: Retrieve data first, then filter in memory
var customers = dbContext.Customers
    .AsEnumerable() // Switches to LINQ to Objects
    .Where(c => CustomMethod(c.Name))
    .ToList();
```

### Performance Issues

```csharp
// Pitfall: Loading too much data
var customers = dbContext.Customers
    .Include(c => c.Orders)
    .Include(c => c.Payments)
    .Include(c => c.ShippingAddresses)
    .ToList();

// Solution: Only include what's needed
var customers = dbContext.Customers
    .Include(c => c.Orders.Where(o => o.Status == "Pending"))
    .ToList();

// Pitfall: N+1 queries
foreach (var customer in dbContext.Customers.ToList())
{
    // This causes a separate query for each customer
    Console.WriteLine($"Customer: {customer.Name}, Orders: {customer.Orders.Count()}");
}

// Solution: Include related data or use a projection
var customerData = dbContext.Customers
    .Select(c => new 
    {
        c.Name, 
        OrderCount = c.Orders.Count()
    })
    .ToList();
```

### Memory Leaks

```csharp
// Pitfall: Storing IQueryable leading to context disposal issues
public IQueryable<Customer> GetCustomers()
{
    // Context will be disposed after this method returns
    return _dbContext.Customers.Where(c => c.Status == "Active");
}

// Later usage will fail because context is disposed
var customers = GetCustomers().ToList(); // Exception!

// Solution: Return materialized results
public List<Customer> GetCustomers()
{
    return _dbContext.Customers.Where(c => c.Status == "Active").ToList();
}
```

## Extending LINQ with Custom Operators

### Creating Custom Extension Methods

```csharp
// Custom LINQ extension methods
public static class LinqExtensions
{
    // Batching items into groups of specified size
    public static IEnumerable<IEnumerable<T>> Batch<T>(
        this IEnumerable<T> source, int batchSize)
    {
        using var enumerator = source.GetEnumerator();
        while (enumerator.MoveNext())
        {
            yield return GetBatch(enumerator, batchSize);
        }
    }
    
    private static IEnumerable<T> GetBatch<T>(
        IEnumerator<T> enumerator, int batchSize)
    {
        yield return enumerator.Current;
        
        for (int i = 1; i < batchSize && enumerator.MoveNext(); i++)
        {
            yield return enumerator.Current;
        }
    }
    
    // Custom aggregation - returns second highest value
    public static T SecondHighest<T, TKey>(
        this IEnumerable<T> source,
        Func<T, TKey> keySelector) where TKey : IComparable<TKey>
    {
        return source
            .OrderByDescending(keySelector)
            .Skip(1)
            .FirstOrDefault();
    }
    
    // Safe navigation method for potentially empty sequences
    public static TResult MaxOrDefault<TSource, TResult>(
        this IEnumerable<TSource> source,
        Func<TSource, TResult> selector,
        TResult defaultValue = default)
    {
        return source.Any() 
            ? source.Max(selector) 
            : defaultValue;
    }
}

// Using custom extensions
var batches = customers.Batch(100).ToList();
var secondHighestSalary = employees.SecondHighest(e => e.Salary);
var highestAge = emptyList.MaxOrDefault(p => p.Age, 0);
```

### Creating Expression Tree Manipulators

```csharp
// Extension for dynamic property access in LINQ queries
public static class ExpressionExtensions
{
    public static IQueryable<T> OrderByProperty<T>(
        this IQueryable<T> source, 
        string propertyName,
        bool ascending = true)
    {
        // Get property info
        var type = typeof(T);
        var property = type.GetProperty(propertyName);
        if (property == null)
            throw new ArgumentException($"Property {propertyName} not found on type {type.Name}");
            
        // Create parameter expression
        var parameter = Expression.Parameter(type, "x");
        
        // Create member access expression
        var propertyAccess = Expression.MakeMemberAccess(parameter, property);
        
        // Create lambda expression
        var lambda = Expression.Lambda(propertyAccess, parameter);
        
        // Create method call expression
        var methodName = ascending ? "OrderBy" : "OrderByDescending";
        var method = typeof(Queryable).GetMethods()
            .Where(m => m.Name == methodName && m.GetParameters().Length == 2)
            .Single()
            .MakeGenericMethod(type, property.PropertyType);
            
        // Execute expression
        return (IQueryable<T>)method.Invoke(null, new object[] { source, lambda });
    }
}

// Usage
var sortedCustomers = dbContext.Customers.OrderByProperty("LastName");
```

## Real-World Examples

### Data Transformation Pipeline

```csharp
public async Task<List<CustomerReportDto>> GenerateCustomerReportAsync()
{
    // Get raw data from database
    var customers = await _dbContext.Customers
        .Include(c => c.Orders)
            .ThenInclude(o => o.Items)
        .Where(c => c.Status == CustomerStatus.Active)
        .ToListAsync();
    
    // Transform data
    var report = customers
        .Select(c => new 
        {
            Customer = c,
            TotalSpent = c.Orders.Sum(o => o.TotalAmount),
            MostRecentOrder = c.Orders.OrderByDescending(o => o.OrderDate).FirstOrDefault(),
            ProductCategories = c.Orders
                .SelectMany(o => o.Items)
                .Select(i => i.Product.Category)
                .Distinct()
                .ToList()
        })
        .Where(x => x.TotalSpent > 0) // Only include customers who have spent money
        .OrderByDescending(x => x.TotalSpent)
        .Select(x => new CustomerReportDto
        {
            CustomerId = x.Customer.Id,
            Name = $"{x.Customer.FirstName} {x.Customer.LastName}",
            Email = x.Customer.Email,
            TotalSpent = x.TotalSpent,
            OrderCount = x.Customer.Orders.Count,
            LastOrderDate = x.MostRecentOrder?.OrderDate,
            PreferredCategories = x.ProductCategories
                .OrderByDescending(cat => x.Customer.Orders
                    .SelectMany(o => o.Items)
                    .Count(i => i.Product.Category == cat))
                .Take(3)
                .ToList()
        })
        .ToList();
        
    return report;
}
```

### Complex Filtering and Searching

```csharp
public async Task<PagedResult<ProductDto>> SearchProductsAsync(ProductSearchRequest request)
{
    // Start with base query
    IQueryable<Product> query = _dbContext.Products;
    
    // Apply filters if provided
    if (!string.IsNullOrEmpty(request.SearchTerm))
    {
        query = query.Where(p => 
            p.Name.Contains(request.SearchTerm) || 
            p.Description.Contains(request.SearchTerm) ||
            p.SKU.Contains(request.SearchTerm));
    }
    
    if (request.MinPrice.HasValue)
    {
        query = query.Where(p => p.Price >= request.MinPrice.Value);
    }
    
    if (request.MaxPrice.HasValue)
    {
        query = query.Where(p => p.Price <= request.MaxPrice.Value);
    }
    
    if (request.CategoryIds != null && request.CategoryIds.Any())
    {
        query = query.Where(p => request.CategoryIds.Contains(p.CategoryId));
    }
    
    if (request.BrandIds != null && request.BrandIds.Any())
    {
        query = query.Where(p => request.BrandIds.Contains(p.BrandId));
    }
    
    if (request.InStock.HasValue)
    {
        query = query.Where(p => p.InStock == request.InStock.Value);
    }
    
    // Calculate total before pagination
    var totalItems = await query.CountAsync();
    
    // Apply sorting
    query = request.SortBy?.ToLower() switch
    {
        "price_asc" => query.OrderBy(p => p.Price),
        "price_desc" => query.OrderByDescending(p => p.Price),
        "name" => query.OrderBy(p => p.Name),
        "newest" => query.OrderByDescending(p => p.CreatedAt),
        "bestselling" => query.OrderByDescending(p => p.SalesCount),
        _ => query.OrderBy(p => p.Name) // default sorting
    };
    
    // Apply pagination
    var pageSize = request.PageSize ?? 20;
    var pageNumber = request.Page ?? 1;
    
    var products = await query
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto
        {
            Id = p.Id,
            Name = p.Name,
            Description = p.Description,
            Price = p.Price,
            ImageUrl = p.ImageUrl,
            Category = p.Category.Name,
            Brand = p.Brand.Name,
            InStock = p.InStock,
            Rating = p.Reviews.Any() ? p.Reviews.Average(r => r.Rating) : null
        })
        .ToListAsync();
    
    return new PagedResult<ProductDto>
    {
        Items = products,
        TotalItems = totalItems,
        PageSize = pageSize,
        CurrentPage = pageNumber,
        TotalPages = (int)Math.Ceiling(totalItems / (double)pageSize)
    };
}
```

### ETL Process with LINQ

```csharp
public async Task<ImportResult> ImportCustomerDataAsync(Stream csvData)
{
    var result = new ImportResult();
    
    // Parse CSV into records
    using (var reader = new StreamReader(csvData))
    using (var csv = new CsvReader(reader, CultureInfo.InvariantCulture))
    {
        var records = csv.GetRecords<CustomerImportRecord>().ToList();
        result.TotalRecords = records.Count;
        
        // Validate and transform records
        var validatedCustomers = records
            .Select(r => 
            {
                // Validate record
                var errors = new List<string>();
                
                if (string.IsNullOrWhiteSpace(r.Email))
                    errors.Add("Email is required");
                else if (!IsValidEmail(r.Email))
                    errors.Add("Email is invalid");
                    
                if (string.IsNullOrWhiteSpace(r.FirstName))
                    errors.Add("First name is required");
                    
                if (string.IsNullOrWhiteSpace(r.LastName))
                    errors.Add("Last name is required");
                
                // Transform to domain model if valid
                if (errors.Any())
                {
                    result.FailedRecords++;
                    result.Errors.Add(new ImportError
                    {
                        Record = r,
                        Errors = errors
                    });
                    return null;
                }
                
                return new Customer
                {
                    FirstName = r.FirstName.Trim(),
                    LastName = r.LastName.Trim(),
                    Email = r.Email.Trim().ToLowerInvariant(),
                    Phone = FormatPhoneNumber(r.Phone),
                    Address = new Address
                    {
                        Street = r.Address,
                        City = r.City,
                        State = r.State,
                        PostalCode = r.ZipCode
                    },
                    Status = CustomerStatus.Active,
                    CreatedAt = DateTime.UtcNow
                };
            })
            .Where(c => c != null)
            .ToList();
            
        // Check for duplicate emails
        var existingEmails = await _dbContext.Customers
            .Where(c => validatedCustomers.Select(vc => vc.Email).Contains(c.Email))
            .Select(c => c.Email)
            .ToListAsync();
            
        var newCustomers = validatedCustomers
            .Where(c => !existingEmails.Contains(c.Email))
            .ToList();
            
        result.DuplicateRecords = validatedCustomers.Count - newCustomers.Count;
        result.SuccessfulRecords = newCustomers.Count;
        
        // Add to database in batches
        foreach (var batch in newCustomers.Batch(100))
        {
            await _dbContext.Customers.AddRangeAsync(batch);
            await _dbContext.SaveChangesAsync();
        }
    }
    
    return result;
}
```

## Further Reading

- [101 LINQ Samples](https://github.com/dotnet/try-samples/tree/main/101-linq-samples)
- [C# Programming Guide: LINQ](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/)
- [LINQ: .NET Language-Integrated Query](https://docs.microsoft.com/en-us/dotnet/standard/linq/)
- [Entity Framework Core - Querying Data](https://docs.microsoft.com/en-us/ef/core/querying/)
- [LINQ to XML Overview](https://docs.microsoft.com/en-us/dotnet/standard/linq/linq-xml-overview)
- [LINQ in Action](https://www.manning.com/books/linq-in-action) by Fabrice Marguerie, Steve Eichert, and Jim Wooley

## Related Topics

- [C# Collection Types](../fundamentals/collections.md)
- [C# Anonymous Types](./anonymous-types.md)
- [C# Lambda Expressions](./lambda-expressions.md)
- [C# Expression Trees](./expression-trees.md)
- [Entity Framework Core](../../databases/orm/entity-framework-core.md)
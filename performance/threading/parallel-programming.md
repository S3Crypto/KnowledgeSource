---
title: "Parallel Programming in C#"
date_created: 2023-05-20
date_updated: 2023-05-20
authors: ["Repository Maintainers"]
tags: ["parallel", "concurrency", "performance", "multithreading", "plinq", "tpl"]
difficulty: "intermediate"
---

# Parallel Programming in C#

## Overview

Parallel programming allows .NET applications to efficiently utilize the processing power of multi-core systems, significantly improving performance for computationally intensive tasks. Modern machines with multiple processors can execute multiple threads simultaneously, making parallel programming critical for performance-optimized applications.

C# and the .NET Framework provide several powerful abstractions to simplify parallel programming, including the Task Parallel Library (TPL), Parallel LINQ (PLINQ), concurrent collections, and various synchronization primitives. This document provides a comprehensive guide to parallel programming techniques in C#, with practical examples, patterns, and best practices.

## Core Concepts

### Concurrency vs. Parallelism

Before diving into implementation, it's important to understand the distinction between concurrency and parallelism:

- **Concurrency**: The ability to handle multiple tasks in overlapping time periods. This is primarily about structure and program design.
- **Parallelism**: The actual simultaneous execution of multiple tasks. This is about execution and leveraging multiple processors.

Concurrent programs may not always execute in parallel (e.g., on a single-core processor), but parallel execution requires a concurrent program design.

### When to Use Parallel Programming

Parallel programming is most beneficial for:

1. **CPU-bound operations**: Computationally intensive tasks that can be broken down into independent units of work
2. **Data parallelism**: When the same operation is applied to different elements of a data set
3. **Task parallelism**: When multiple independent operations can be executed simultaneously

Parallelism is less useful or even counterproductive for:

1. **I/O-bound operations**: Use asynchronous programming (async/await) instead
2. **Operations with shared state**: Heavy synchronization can negate performance gains
3. **Very small workloads**: The overhead of creating and managing threads may exceed the benefits
4. **Sequential dependencies**: Tasks that must run in a specific order

## Task Parallel Library (TPL)

The Task Parallel Library is the primary API for parallel programming in C#, providing high-level abstractions for working with tasks and parallel operations.

### Parallel.For and Parallel.ForEach

The simplest way to parallelize loop operations:

```csharp
// Sequential loop
for (int i = 0; i < 1000; i++)
{
    ProcessItem(i);
}

// Parallel equivalent
Parallel.For(0, 1000, i =>
{
    ProcessItem(i);
});

// Sequential foreach
foreach (var item in items)
{
    ProcessItem(item);
}

// Parallel equivalent
Parallel.ForEach(items, item =>
{
    ProcessItem(item);
});
```

### Controlling Parallel Operations

You can control the degree of parallelism and handle cancellation:

```csharp
// Create options for controlling parallel execution
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount, // Use all processors
    CancellationToken = cancellationToken // Support cancellation
};

try
{
    Parallel.ForEach(items, options, item =>
    {
        ProcessItem(item);
        
        // Periodically check for cancellation in long-running operations
        options.CancellationToken.ThrowIfCancellationRequested();
    });
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation was canceled");
}
```

### Using Local State

For operations that accumulate results, use local state to avoid thread contention:

```csharp
int totalSum = 0;

Parallel.ForEach(
    items,                        // Source collection
    () => 0,                      // Initialize local state
    (item, state, index, localSum) =>
    {
        // Update local state without contention
        return localSum + ProcessItem(item);
    },
    localSum =>
    {
        // Merge local state with global state
        Interlocked.Add(ref totalSum, localSum);
    });

Console.WriteLine($"Total sum: {totalSum}");
```

### Parallel.Invoke

For executing a set of discrete operations in parallel:

```csharp
Parallel.Invoke(
    () => ProcessPartA(),
    () => ProcessPartB(),
    () => ProcessPartC(),
    () => ProcessPartD()
);
```

### Breaking out of Loops

You can stop processing in parallel loops using `ParallelLoopState`:

```csharp
Parallel.ForEach(items, (item, loopState) =>
{
    if (ShouldBreak(item))
    {
        loopState.Break(); // Stop processing any items beyond current iteration
        return;
    }
    
    if (ShouldStop(item))
    {
        loopState.Stop(); // Stop all processing as soon as possible
        return;
    }
    
    ProcessItem(item);
});
```

## Parallel LINQ (PLINQ)

PLINQ extends LINQ to provide parallel query execution:

```csharp
// Sequential LINQ
var results = from item in items
              where Predicate(item)
              select Transform(item);

// Parallel equivalent using AsParallel()
var parallelResults = from item in items.AsParallel()
                      where Predicate(item)
                      select Transform(item);

// Fluent syntax
var results = items.AsParallel()
                  .Where(item => Predicate(item))
                  .Select(item => Transform(item))
                  .ToList(); // Forces execution
```

### Preserving Order

By default, PLINQ doesn't guarantee the order of results. To preserve order:

```csharp
var orderedResults = items.AsParallel()
                         .AsOrdered() // Preserve input order
                         .Where(item => Predicate(item))
                         .Select(item => Transform(item));
```

### Controlling the Degree of Parallelism

```csharp
var results = items.AsParallel()
                  .WithDegreeOfParallelism(4) // Use 4 cores maximum
                  .Where(item => Predicate(item))
                  .Select(item => Transform(item));
```

### Handling Exceptions

PLINQ aggregates exceptions from parallel tasks:

```csharp
try
{
    var results = items.AsParallel()
                      .Select(item => ProcessItem(item)) // May throw
                      .ToList(); // Execute query
}
catch (AggregateException ae)
{
    foreach (var ex in ae.InnerExceptions)
    {
        Console.WriteLine($"Error: {ex.Message}");
    }
}
```

### ForAll for Side Effects

When you need to process results without creating an intermediate collection:

```csharp
// No intermediate collection is created
items.AsParallel()
     .Where(item => Predicate(item))
     .Select(item => Transform(item))
     .ForAll(result => ProcessResult(result));
```

## Partitioning Strategies

Effective partitioning is crucial for parallel performance:

### Static Partitioning

Divide the work into fixed chunks:

```csharp
public void ProcessInParallel<T>(IList<T> items, int degreeOfParallelism)
{
    // Calculate chunk size
    int count = items.Count;
    int chunkSize = (count + degreeOfParallelism - 1) / degreeOfParallelism;
    
    // Process in parallel
    Parallel.For(0, degreeOfParallelism, i =>
    {
        int start = i * chunkSize;
        int end = Math.Min(start + chunkSize, count);
        
        // Process chunk [start, end)
        for (int j = start; j < end; j++)
        {
            ProcessItem(items[j]);
        }
    });
}
```

### Dynamic Partitioning with Range Partitioner

For better load balancing, use `Partitioner.Create`:

```csharp
// Create a range partitioner for 0...999
var rangePartitioner = Partitioner.Create(0, 1000);

Parallel.ForEach(rangePartitioner, range =>
{
    // Process range [range.Item1, range.Item2)
    for (int i = range.Item1; i < range.Item2; i++)
    {
        ProcessItem(i);
    }
});
```

### Custom Partitioner for Uneven Workloads

Create a custom partitioner for workloads where processing time varies significantly:

```csharp
public class LoadBalancingPartitioner<T> : Partitioner<T>
{
    private readonly IEnumerable<T> _source;
    private readonly int _chunkSize;
    
    public LoadBalancingPartitioner(IEnumerable<T> source, int chunkSize = 1)
    {
        _source = source;
        _chunkSize = chunkSize;
    }
    
    public override bool SupportsDynamicPartitions => true;
    
    public override IList<IEnumerator<T>> GetPartitions(int partitionCount)
    {
        var dynamicPartitioner = GetDynamicPartitions();
        return Enumerable.Range(0, partitionCount)
            .Select(_ => dynamicPartitioner.GetEnumerator())
            .ToList();
    }
    
    public override IEnumerable<T> GetDynamicPartitions()
    {
        var enumerator = _source.GetEnumerator();
        return new DynamicPartitioner(enumerator, _chunkSize);
    }
    
    private class DynamicPartitioner : IEnumerable<T>
    {
        private readonly IEnumerator<T> _enumerator;
        private readonly int _chunkSize;
        private readonly object _lock = new object();
        
        public DynamicPartitioner(IEnumerator<T> enumerator, int chunkSize)
        {
            _enumerator = enumerator;
            _chunkSize = chunkSize;
        }
        
        public IEnumerator<T> GetEnumerator()
        {
            while (true)
            {
                List<T> chunk = null;
                
                lock (_lock)
                {
                    if (!_enumerator.MoveNext())
                        yield break;
                    
                    chunk = new List<T>(_chunkSize) { _enumerator.Current };
                    
                    for (int i = 1; i < _chunkSize && _enumerator.MoveNext(); i++)
                    {
                        chunk.Add(_enumerator.Current);
                    }
                }
                
                foreach (var item in chunk)
                {
                    yield return item;
                }
            }
        }
        
        IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
    }
}

// Usage
var items = GetLargeDataSet();
var partitioner = new LoadBalancingPartitioner<Item>(items, chunkSize: 10);

Parallel.ForEach(partitioner, item =>
{
    ProcessItem(item);
});
```

## Synchronization and Thread Safety

When parallel tasks share state, synchronization becomes necessary:

### Interlocked Operations

For simple atomic operations:

```csharp
private long _counter = 0;
private long _sum = 0;

public void IncrementCounter()
{
    Interlocked.Increment(ref _counter);
}

public void AddToSum(long value)
{
    Interlocked.Add(ref _sum, value);
}

public long GetAndResetCounter()
{
    return Interlocked.Exchange(ref _counter, 0);
}
```

### Lock Statement

For protecting critical sections:

```csharp
private readonly object _lockObject = new object();
private List<Result> _results = new List<Result>();

public void AddResult(Result result)
{
    lock (_lockObject)
    {
        _results.Add(result);
    }
}

public List<Result> GetResults()
{
    lock (_lockObject)
    {
        return _results.ToList(); // Return a copy
    }
}
```

### Reader-Writer Lock

When reads are more frequent than writes:

```csharp
private readonly ReaderWriterLockSlim _rwLock = new ReaderWriterLockSlim();
private Dictionary<string, object> _cache = new Dictionary<string, object>();

public object GetValue(string key)
{
    _rwLock.EnterReadLock();
    try
    {
        return _cache.TryGetValue(key, out var value) ? value : null;
    }
    finally
    {
        _rwLock.ExitReadLock();
    }
}

public void SetValue(string key, object value)
{
    _rwLock.EnterWriteLock();
    try
    {
        _cache[key] = value;
    }
    finally
    {
        _rwLock.ExitWriteLock();
    }
}
```

### Concurrent Collections

Thread-safe collections for shared state:

```csharp
private readonly ConcurrentDictionary<string, object> _cache = 
    new ConcurrentDictionary<string, object>();
private readonly ConcurrentQueue<WorkItem> _workItems = 
    new ConcurrentQueue<WorkItem>();
private readonly ConcurrentBag<Result> _results = 
    new ConcurrentBag<Result>();

public void ProcessWorkItems()
{
    Parallel.ForEach(_workItems.GetConsumingPartitioner(), item =>
    {
        var result = ProcessItem(item);
        _results.Add(result);
    });
}

public object GetOrAddCacheItem(string key, Func<string, object> valueFactory)
{
    return _cache.GetOrAdd(key, valueFactory);
}
```

## Advanced Parallel Patterns

### Fork-Join Pattern

The pattern of splitting work into parallel tasks and joining the results:

```csharp
public int ComputeSum(int[] array)
{
    if (array.Length <= 1000)
    {
        return array.Sum(); // Base case
    }
    
    // Fork: Split the array and process in parallel
    int middle = array.Length / 2;
    int[] left = array.Take(middle).ToArray();
    int[] right = array.Skip(middle).ToArray();
    
    int leftSum = 0;
    int rightSum = 0;
    
    Parallel.Invoke(
        () => leftSum = ComputeSum(left),
        () => rightSum = ComputeSum(right)
    );
    
    // Join: Combine results
    return leftSum + rightSum;
}
```

### Pipeline Pattern

Implementing a parallel processing pipeline:

```csharp
public class ParallelPipeline<TInput, TOutput>
{
    private readonly BlockingCollection<TInput> _inputQueue = 
        new BlockingCollection<TInput>();
    private readonly BlockingCollection<TOutput> _outputQueue = 
        new BlockingCollection<TOutput>();
    private readonly CancellationTokenSource _cts = 
        new CancellationTokenSource();
    private readonly Task _processingTask;
    
    public ParallelPipeline(Func<TInput, TOutput> processor, int workerCount)
    {
        _processingTask = Task.Run(() =>
        {
            try
            {
                Parallel.For(0, workerCount, _ =>
                {
                    try
                    {
                        foreach (var item in _inputQueue.GetConsumingEnumerable(_cts.Token))
                        {
                            TOutput result = processor(item);
                            _outputQueue.Add(result, _cts.Token);
                        }
                    }
                    catch (OperationCanceledException)
                    {
                        // Expected when cancellation requested
                    }
                });
            }
            finally
            {
                _outputQueue.CompleteAdding();
            }
        });
    }
    
    public void AddInput(TInput input)
    {
        _inputQueue.Add(input);
    }
    
    public void CompleteAdding()
    {
        _inputQueue.CompleteAdding();
    }
    
    public IEnumerable<TOutput> GetOutputs()
    {
        return _outputQueue.GetConsumingEnumerable();
    }
    
    public Task Completion => _processingTask;
    
    public void Cancel()
    {
        _cts.Cancel();
    }
}

// Usage
var pipeline = new ParallelPipeline<string, int>(
    processor: s => ProcessString(s),
    workerCount: Environment.ProcessorCount
);

// Add inputs
foreach (var input in inputData)
{
    pipeline.AddInput(input);
}

pipeline.CompleteAdding();

// Process outputs as they become available
foreach (var output in pipeline.GetOutputs())
{
    Console.WriteLine($"Output: {output}");
}

// Wait for completion
await pipeline.Completion;
```

### Map-Reduce Pattern

Implementing distributed computing patterns:

```csharp
public class MapReduce<TInput, TIntermediate, TOutput>
{
    public TOutput Process(
        IEnumerable<TInput> inputs,
        Func<TInput, IEnumerable<TIntermediate>> mapper,
        Func<IEnumerable<TIntermediate>, TOutput> reducer,
        int degreeOfParallelism = -1)
    {
        // Map phase
        var intermediateResults = inputs
            .AsParallel()
            .WithDegreeOfParallelism(degreeOfParallelism > 0 
                ? degreeOfParallelism 
                : Environment.ProcessorCount)
            .SelectMany(input => mapper(input))
            .ToList();
        
        // Reduce phase
        return reducer(intermediateResults);
    }
}

// Usage - Word Count Example
var mapReduce = new MapReduce<string, KeyValuePair<string, int>, Dictionary<string, int>>();

var wordCount = mapReduce.Process(
    inputs: documentList,
    
    // Map: Split document into words and count each once
    mapper: document => document
        .Split(new[] { ' ', '\t', '\n', '\r', '.', ',', ';', ':', '!', '?' })
        .Where(word => !string.IsNullOrWhiteSpace(word))
        .Select(word => word.ToLowerInvariant())
        .GroupBy(word => word)
        .Select(g => new KeyValuePair<string, int>(g.Key, g.Count())),
    
    // Reduce: Combine word counts
    reducer: pairs => pairs
        .GroupBy(pair => pair.Key)
        .ToDictionary(g => g.Key, g => g.Sum(pair => pair.Value))
);

// Print results
foreach (var pair in wordCount.OrderByDescending(p => p.Value))
{
    Console.WriteLine($"{pair.Key}: {pair.Value}");
}
```

### Producer-Consumer Pattern

Implementing the producer-consumer pattern with parallel processing:

```csharp
public class ParallelProducerConsumer<T>
{
    private readonly BlockingCollection<T> _queue;
    private readonly Action<T> _consumer;
    private readonly CancellationTokenSource _cts = new CancellationTokenSource();
    private readonly Task _consumingTask;
    private readonly int _consumerCount;
    
    public ParallelProducerConsumer(Action<T> consumer, int boundedCapacity = -1, int consumerCount = -1)
    {
        _consumer = consumer;
        _consumerCount = consumerCount > 0 ? consumerCount : Environment.ProcessorCount;
        _queue = boundedCapacity > 0 
            ? new BlockingCollection<T>(boundedCapacity) 
            : new BlockingCollection<T>();
        
        _consumingTask = Task.Run(() => ConsumeItems());
    }
    
    public void Produce(T item)
    {
        _queue.Add(item);
    }
    
    public void CompleteAdding()
    {
        _queue.CompleteAdding();
    }
    
    private void ConsumeItems()
    {
        try
        {
            Parallel.For(0, _consumerCount, _ =>
            {
                try
                {
                    // Each parallel consumer gets items from the queue
                    foreach (var item in _queue.GetConsumingEnumerable(_cts.Token))
                    {
                        _consumer(item);
                    }
                }
                catch (OperationCanceledException)
                {
                    // Expected when canceled
                }
            });
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error in consumer task: {ex.Message}");
        }
    }
    
    public Task WaitForCompletionAsync()
    {
        return _consumingTask;
    }
    
    public void Cancel()
    {
        _cts.Cancel();
    }
    
    public void Dispose()
    {
        Cancel();
        _queue.Dispose();
        _cts.Dispose();
    }
}

// Usage
using (var pc = new ParallelProducerConsumer<WorkItem>(
    consumer: item => ProcessWorkItem(item),
    boundedCapacity: 1000,
    consumerCount: 8))
{
    // Produce items
    for (int i = 0; i < 10000; i++)
    {
        pc.Produce(new WorkItem { Id = i });
    }
    
    // Signal no more items
    pc.CompleteAdding();
    
    // Wait for processing to complete
    pc.WaitForCompletionAsync().Wait();
}
```

## Performance Considerations

### CPU-Bound vs. I/O-Bound Work

Choose the right approach based on the nature of the work:

```csharp
// For CPU-bound work: Use Parallel
public void ProcessCpuBoundItems(List<Item> items)
{
    Parallel.ForEach(items, item =>
    {
        // CPU-intensive work
        ProcessItemCpuIntensive(item);
    });
}

// For I/O-bound work: Use async/await
public async Task ProcessIoBoundItemsAsync(List<Item> items)
{
    // Create a list of tasks
    var tasks = items.Select(item => ProcessItemIoBoundAsync(item));
    
    // Wait for all I/O operations to complete
    await Task.WhenAll(tasks);
}

// For mixed work: Combine approaches
public async Task ProcessMixedItemsAsync(List<Item> items)
{
    // Process CPU-bound work in parallel on a background thread
    await Task.Run(() =>
    {
        Parallel.ForEach(items, item =>
        {
            // CPU-intensive pre-processing
            PreProcessItem(item);
        });
    });
    
    // Then process I/O-bound work
    var tasks = items.Select(item => SaveItemAsync(item));
    await Task.WhenAll(tasks);
}
```

### Task Granularity

Finding the right balance for task size:

```csharp
// Too fine-grained: Excessive overhead
Parallel.For(0, 100, i =>
{
    // Very quick operation - parallel overhead exceeds benefit
    results[i] = i * 2; 
});

// Too coarse-grained: Poor load balancing
Parallel.For(0, 4, i =>
{
    // Processing 25% of data in each task
    // Some tasks may finish much earlier than others
    int start = i * (array.Length / 4);
    int end = (i + 1) * (array.Length / 4);
    
    for (int j = start; j < end; j++)
    {
        ProcessElement(array[j]);
    }
});

// Better: Dynamic partitioning
Parallel.ForEach(
    Partitioner.Create(0, array.Length), 
    range =>
    {
        for (int i = range.Item1; i < range.Item2; i++)
        {
            ProcessElement(array[i]);
        }
    }
);
```

### Thread Pool Considerations

Optimizing thread pool usage:

```csharp
// Configure thread pool settings for the application
ThreadPool.SetMinThreads(workerThreads: 16, completionPortThreads: 16);

// Monitor thread pool for troubleshooting
public void MonitorThreadPool()
{
    while (true)
    {
        ThreadPool.GetAvailableThreads(out int workerThreads, out int completionPortThreads);
        ThreadPool.GetMaxThreads(out int maxWorkerThreads, out int maxCompletionPortThreads);
        
        Console.WriteLine($"Worker threads: {workerThreads}/{maxWorkerThreads}");
        Console.WriteLine($"Completion port threads: {completionPortThreads}/{maxCompletionPortThreads}");
        
        Thread.Sleep(1000);
    }
}
```

### Avoiding Oversubscription

Control the degree of parallelism to prevent excessive context switching:

```csharp
public class ParallelProcessor
{
    private readonly int _degreeOfParallelism;
    
    public ParallelProcessor(int? degreeOfParallelism = null)
    {
        // Default to 75% of processors to leave resources for other processes
        _degreeOfParallelism = degreeOfParallelism ?? 
            Math.Max(1, (int)(Environment.ProcessorCount * 0.75));
    }
    
    public void Process<T>(IEnumerable<T> items, Action<T> processor)
    {
        Parallel.ForEach(
            items,
            new ParallelOptions { MaxDegreeOfParallelism = _degreeOfParallelism },
            processor
        );
    }
}
```

## Common Pitfalls and Solutions

### Race Conditions

Identifying and fixing race conditions:

```csharp
// PROBLEMATIC: Race condition
private int _sum = 0;

public void ProcessItemsWithRaceCondition(List<int> items)
{
    Parallel.ForEach(items, item =>
    {
        // Race condition: Multiple threads read and write _sum
        _sum += item;
    });
}

// SOLUTION 1: Use Interlocked
private int _sum = 0;

public void ProcessItemsWithInterlocked(List<int> items)
{
    Parallel.ForEach(items, item =>
    {
        Interlocked.Add(ref _sum, item);
    });
}

// SOLUTION 2: Use lock
private int _sum = 0;
private readonly object _lockObject = new object();

public void ProcessItemsWithLock(List<int> items)
{
    Parallel.ForEach(items, item =>
    {
        lock (_lockObject)
        {
            _sum += item;
        }
    });
}

// SOLUTION 3: Use local state
private int _sum = 0;

public void ProcessItemsWithLocalState(List<int> items)
{
    Parallel.ForEach(
        items,
        () => 0,                      // Initialize local sum
        (item, state, index, localSum) => localSum + item,  // Accumulate locally
        localSum => Interlocked.Add(ref _sum, localSum)     // Combine results
    );
}
```

### Deadlocks

Avoiding deadlocks in parallel code:

```csharp
// PROBLEMATIC: Potential deadlock
private readonly object _lockA = new object();
private readonly object _lockB = new object();

public void DeadlockRisk()
{
    Parallel.Invoke(
        () =>
        {
            lock (_lockA)
            {
                Thread.Sleep(100); // Increase deadlock chance
                lock (_lockB)
                {
                    // Process with both locks
                }
            }
        },
        () =>
        {
            lock (_lockB)
            {
                Thread.Sleep(100); // Increase deadlock chance
                lock (_lockA)
                {
                    // Process with both locks
                }
            }
        }
    );
}

// SOLUTION: Consistent lock ordering
public void DeadlockSolution()
{
    Parallel.Invoke(
        () =>
        {
            lock (_lockA)
            {
                lock (_lockB)
                {
                    // Process with both locks
                }
            }
        },
        () =>
        {
            lock (_lockA)
            {
                lock (_lockB)
                {
                    // Process with both locks
                }
            }
        }
    );
}
```

### Excessive Context Switching

Minimizing context switching overhead:

```csharp
// PROBLEMATIC: Too many small parallel operations
public void ExcessiveParallelism(List<int> numbers)
{
    // Creates thousands of tiny tasks
    var results = numbers.AsParallel().Select(n => DoTinyOperation(n)).ToList();
}

// SOLUTION: Batch operations
public void BatchedParallelism(List<int> numbers)
{
    // Process in reasonable chunks
    int batchSize = 1000;
    
    var batches = Enumerable.Range(0, (numbers.Count + batchSize - 1) / batchSize)
        .Select(i => numbers.Skip(i * batchSize).Take(batchSize).ToList())
        .ToList();
    
    // Process each batch in parallel
    Parallel.ForEach(batches, batch =>
    {
        foreach (var number in batch)
        {
            DoTinyOperation(number);
        }
    });
}
```

### Parallel Overheads

Being aware of parallel processing overhead:

```csharp
// PROBLEMATIC: Parallelism on small workloads
public int[] SquareArrayInefficient(int[] input)
{
    if (input.Length < 1000) // Too small for parallelism to be effective
    {
        return input.AsParallel().Select(x => x * x).ToArray();
    }
    return input.Select(x => x * x).ToArray();
}

// SOLUTION: Only use parallelism for larger workloads
public int[] SquareArrayEfficient(int[] input)
{
    if (input.Length >= 10000) // Large enough to benefit from parallelism
    {
        return input.AsParallel().Select(x => x * x).ToArray();
    }
    return input.Select(x => x * x).ToArray();
}
```

## Testing and Debugging

### Deterministic Testing

Creating deterministic tests for parallel code:

```csharp
[Fact]
public void ParallelProcessing_ShouldGiveCorrectResults()
{
    // Arrange
    int[] numbers = Enumerable.Range(1, 10000).ToArray();
    int expectedSum = numbers.Sum();
    
    // Act
    int actualSum = 0;
    
    Parallel.ForEach(
        numbers,
        () => 0,
        (num, state, index, localSum) => localSum + num,
        localSum => Interlocked.Add(ref actualSum, localSum)
    );
    
    // Assert
    Assert.Equal(expectedSum, actualSum);
}
```

### Debugging Parallel Execution

Tools and techniques for debugging parallel issues:

```csharp
// Add tracing for debugging
public void ProcessWithTracing(List<Item> items)
{
    Parallel.ForEach(items, item =>
    {
        try
        {
            Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} processing item {item.Id}");
            ProcessItem(item);
            Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} completed item {item.Id}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} error on item {item.Id}: {ex.Message}");
            throw;
        }
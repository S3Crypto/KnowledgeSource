---
title: "Delegates and Events in C#"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["c#", "delegates", "events", "callbacks", "event-driven", "advanced"]
difficulty: "intermediate"
---

# Delegates and Events in C#

## Overview

Delegates and events are fundamental C# features that enable powerful programming patterns including callbacks, event-driven programming, and the observer pattern. This document covers the mechanics of delegates and events, their practical applications, and best practices for using them effectively in C# applications.

## Delegates

### What is a Delegate?

A delegate is a type that represents references to methods with a particular parameter list and return type. Delegates are C#'s type-safe, object-oriented implementation of function pointers or callbacks. They allow methods to be passed as parameters, stored as variables, and invoked dynamically.

### Delegate Syntax

Defining a delegate type:

```csharp
// Delegate declaration
public delegate int MathOperation(int x, int y);

// Methods that match the delegate signature
public static int Add(int a, int b) => a + b;
public static int Subtract(int a, int b) => a - b;
public static int Multiply(int a, int b) => a * b;
public static int Divide(int a, int b) => b != 0 ? a / b : 0;
```

Using a delegate:

```csharp
// Creating a delegate instance
MathOperation operation = Add;

// Invoking a delegate
int result = operation(10, 5); // result = 15

// Reassigning a delegate
operation = Multiply;
result = operation(10, 5); // result = 50
```

### Multicast Delegates

Delegates in C# can reference multiple methods. These are called multicast delegates:

```csharp
// Delegate that returns void
public delegate void Logger(string message);

// Methods matching the delegate signature
public static void ConsoleLogger(string message) => Console.WriteLine($"Console: {message}");
public static void FileLogger(string message) => File.AppendAllText("log.txt", $"{message}\n");
public static void DatabaseLogger(string message) => /* Log to database */;

// Creating a multicast delegate
Logger log = ConsoleLogger;
log += FileLogger; // Add a method
log += DatabaseLogger; // Add another method

// Invoking calls all methods in the invocation list
log("System started"); // Logs to console, file, and database

// Removing a method
log -= FileLogger;
log("Method removed"); // Logs to console and database only
```

Important characteristics of multicast delegates:

1. Methods are called in the order they were added
2. If the delegate has a return value, only the value from the last method is returned
3. If any method throws an exception, subsequent methods won't be called

### Anonymous Methods

C# 2.0 introduced anonymous methods, which allow you to define inline methods:

```csharp
// Using an anonymous method
MathOperation multiply = delegate(int x, int y) { return x * y; };

// Using with event handlers
button.Click += delegate(object sender, EventArgs e) 
{
    Console.WriteLine("Button was clicked");
};
```

### Lambda Expressions

C# 3.0 introduced lambda expressions, a more concise way to define anonymous methods:

```csharp
// Lambda expression syntax
MathOperation add = (x, y) => x + y;
MathOperation subtract = (x, y) => x - y;

// Multiline lambda
MathOperation divide = (x, y) => 
{
    if (y == 0)
        throw new DivideByZeroException();
    return x / y;
};

// Using with LINQ
var evenNumbers = numbers.Where(n => n % 2 == 0);
```

### Built-in Delegate Types

The .NET Framework includes several built-in delegate types:

#### Action Delegates

For methods that return void:

```csharp
// Action with no parameters
Action printMessage = () => Console.WriteLine("Hello, World!");
printMessage(); // Prints "Hello, World!"

// Action with parameters
Action<string> greet = name => Console.WriteLine($"Hello, {name}!");
greet("Alice"); // Prints "Hello, Alice!"

// Action with multiple parameters
Action<string, int> repeatGreeting = (name, times) => 
{
    for (int i = 0; i < times; i++)
        Console.WriteLine($"Hello, {name}!");
};
repeatGreeting("Bob", 3); // Prints "Hello, Bob!" three times
```

Action delegates exist for up to 16 parameters: `Action<T1>`, `Action<T1, T2>`, etc.

#### Func Delegates

For methods that return a value:

```csharp
// Func with no parameters, returns string
Func<string> getCurrentTime = () => DateTime.Now.ToShortTimeString();
string time = getCurrentTime(); // Returns current time as string

// Func with parameters, returns a value
Func<int, int, int> add = (a, b) => a + b;
int sum = add(5, 3); // Returns 8

// More complex example
Func<List<int>, int> calculateAverage = numbers =>
{
    if (numbers == null || numbers.Count == 0)
        return 0;
    return numbers.Sum() / numbers.Count;
};
```

Func delegates exist for up to 16 input parameters plus a return value: `Func<TResult>`, `Func<T1, TResult>`, etc.

#### Predicate Delegate

A special delegate that returns a boolean:

```csharp
// Predicate takes one parameter and returns bool
Predicate<int> isEven = number => number % 2 == 0;
bool result = isEven(4); // Returns true

// Used in many collection methods
List<int> numbers = new List<int> { 1, 2, 3, 4, 5, 6 };
List<int> evenNumbers = numbers.FindAll(isEven); // Returns [2, 4, 6]
```

### Delegate Variance

C# supports covariance and contravariance for delegates, allowing more flexible delegate assignments:

```csharp
// Base and derived classes
public class Animal { }
public class Dog : Animal { }

// Covariance: more derived return type
Func<Animal> animalFactory = () => new Animal();
Func<Dog> dogFactory = () => new Dog();
animalFactory = dogFactory; // Valid: Dog is-a Animal

// Contravariance: less derived parameter type
Action<Dog> dogAction = dog => Console.WriteLine("Dog action");
Action<Animal> animalAction = animal => Console.WriteLine("Animal action");
dogAction = animalAction; // Valid: Dog is-a Animal, so anything that can handle Animal can handle Dog
```

## Events

### What is an Event?

An event is a mechanism for communication between objects that enables a publish-subscribe model. It allows a class to notify other classes when something of interest occurs, without those classes having to continuously check.

Events are built on delegates but add an important encapsulation layer, ensuring subscribers can only add or remove handlers, not invoke the event directly.

### Declaring and Raising Events

Basic event declaration and usage:

```csharp
// Class with an event
public class Button
{
    // Event declaration using EventHandler delegate
    public event EventHandler Click;
    
    // Method to raise the event
    public void OnClick()
    {
        // Safely invoke the event
        Click?.Invoke(this, EventArgs.Empty);
    }
}

// Subscribing to an event
Button button = new Button();
button.Click += (sender, e) => Console.WriteLine("Button clicked!");

// Raising the event
button.OnClick(); // Prints "Button clicked!"
```

### Custom Event Arguments

Creating custom event arguments to pass more information:

```csharp
// Custom event args
public class ProductEventArgs : EventArgs
{
    public string ProductName { get; }
    public decimal Price { get; }
    
    public ProductEventArgs(string productName, decimal price)
    {
        ProductName = productName;
        Price = price;
    }
}

// Class with custom event args
public class ShoppingCart
{
    // Event with custom event args
    public event EventHandler<ProductEventArgs> ProductAdded;
    
    // Method to add product and raise event
    public void AddProduct(string name, decimal price)
    {
        // Add product to cart...
        
        // Raise event with product information
        OnProductAdded(new ProductEventArgs(name, price));
    }
    
    // Protected method to raise the event
    protected virtual void OnProductAdded(ProductEventArgs e)
    {
        ProductAdded?.Invoke(this, e);
    }
}

// Subscribing
var cart = new ShoppingCart();
cart.ProductAdded += (sender, e) => 
{
    Console.WriteLine($"Product added: {e.ProductName}, Price: ${e.Price}");
};

// Adding product
cart.AddProduct("Laptop", 999.99m);
```

### Event Accessor Syntax

Custom event accessor syntax gives more control over event subscription and notification:

```csharp
public class Counter
{
    // Backing field for the event
    private EventHandler _thresholdReached;
    
    // Custom event accessors
    public event EventHandler ThresholdReached
    {
        add
        {
            // Custom logic before adding handler
            Console.WriteLine("Handler added");
            _thresholdReached += value;
        }
        remove
        {
            // Custom logic before removing handler
            Console.WriteLine("Handler removed");
            _thresholdReached -= value;
        }
    }
    
    public void Count(int threshold)
    {
        for (int i = 0; i < 100; i++)
        {
            if (i == threshold)
            {
                OnThresholdReached(EventArgs.Empty);
            }
        }
    }
    
    protected virtual void OnThresholdReached(EventArgs e)
    {
        _thresholdReached?.Invoke(this, e);
    }
}
```

### Generic EventHandler

Generic `EventHandler<TEventArgs>` provides type-safe event handling:

```csharp
// Event with generic EventHandler
public event EventHandler<CustomEventArgs> CustomEvent;

// Raising the event
protected virtual void OnCustomEvent(CustomEventArgs e)
{
    CustomEvent?.Invoke(this, e);
}

// Subscribing
obj.CustomEvent += (sender, e) => 
{
    // Access strongly-typed properties of e
    Console.WriteLine(e.CustomProperty);
};
```

## Practical Applications

### Callbacks and Continuation Passing

Using delegates for callbacks:

```csharp
public class AsyncProcessor
{
    public void ProcessAsync(string data, Action<string> callback)
    {
        // Simulate async processing
        Task.Run(() => 
        {
            // Process data
            string result = Process(data);
            
            // Call the callback with the result
            callback(result);
        });
    }
    
    private string Process(string data)
    {
        // Simulate processing
        Thread.Sleep(1000);
        return data.ToUpper();
    }
}

// Usage
var processor = new AsyncProcessor();
processor.ProcessAsync("hello", result => 
{
    Console.WriteLine($"Processing complete: {result}");
});
Console.WriteLine("Processing started...");
```

### Strategy Pattern

Using delegates to implement the strategy pattern:

```csharp
public class SortingContext
{
    // The strategy is represented by a delegate
    private Func<List<int>, List<int>> _sortStrategy;
    
    public SortingContext(Func<List<int>, List<int>> sortStrategy)
    {
        _sortStrategy = sortStrategy;
    }
    
    public void SetStrategy(Func<List<int>, List<int>> sortStrategy)
    {
        _sortStrategy = sortStrategy;
    }
    
    public List<int> Sort(List<int> data)
    {
        return _sortStrategy(data);
    }
}

// Sorting strategies
public static class SortingStrategies
{
    public static List<int> BubbleSort(List<int> data)
    {
        var result = new List<int>(data);
        // Bubble sort implementation
        return result;
    }
    
    public static List<int> QuickSort(List<int> data)
    {
        var result = new List<int>(data);
        // Quick sort implementation
        return result;
    }
    
    public static List<int> MergeSort(List<int> data)
    {
        var result = new List<int>(data);
        // Merge sort implementation
        return result;
    }
}

// Usage
var context = new SortingContext(SortingStrategies.QuickSort);
var sorted = context.Sort(new List<int> { 5, 3, 1, 4, 2 });

// Switch strategy
context.SetStrategy(SortingStrategies.MergeSort);
sorted = context.Sort(new List<int> { 5, 3, 1, 4, 2 });
```

### Observer Pattern

The observer pattern can be implemented using events:

```csharp
// Subject (Publisher)
public class WeatherStation
{
    public decimal Temperature { get; private set; }
    public decimal Humidity { get; private set; }
    public decimal Pressure { get; private set; }
    
    // Event for weather changes
    public event EventHandler<WeatherChangedEventArgs> WeatherChanged;
    
    public void UpdateMeasurements(decimal temperature, decimal humidity, decimal pressure)
    {
        Temperature = temperature;
        Humidity = humidity;
        Pressure = pressure;
        
        // Notify observers (subscribers)
        OnWeatherChanged(new WeatherChangedEventArgs(Temperature, Humidity, Pressure));
    }
    
    protected virtual void OnWeatherChanged(WeatherChangedEventArgs e)
    {
        WeatherChanged?.Invoke(this, e);
    }
}

// Event args
public class WeatherChangedEventArgs : EventArgs
{
    public decimal Temperature { get; }
    public decimal Humidity { get; }
    public decimal Pressure { get; }
    
    public WeatherChangedEventArgs(decimal temperature, decimal humidity, decimal pressure)
    {
        Temperature = temperature;
        Humidity = humidity;
        Pressure = pressure;
    }
}

// Observers (Subscribers)
public class TemperatureDisplay
{
    public TemperatureDisplay(WeatherStation weatherStation)
    {
        // Subscribe to the weather station's event
        weatherStation.WeatherChanged += HandleWeatherChanged;
    }
    
    private void HandleWeatherChanged(object sender, WeatherChangedEventArgs e)
    {
        Console.WriteLine($"Temperature Display: {e.Temperature}°C");
    }
}

public class WeatherStatistics
{
    private List<decimal> _temperatureReadings = new List<decimal>();
    
    public WeatherStatistics(WeatherStation weatherStation)
    {
        // Subscribe to the weather station's event
        weatherStation.WeatherChanged += HandleWeatherChanged;
    }
    
    private void HandleWeatherChanged(object sender, WeatherChangedEventArgs e)
    {
        _temperatureReadings.Add(e.Temperature);
        
        decimal average = _temperatureReadings.Average();
        decimal max = _temperatureReadings.Max();
        decimal min = _temperatureReadings.Min();
        
        Console.WriteLine($"Weather Statistics - Avg: {average}°C, Min: {min}°C, Max: {max}°C");
    }
}

// Usage
var weatherStation = new WeatherStation();

// Create observers that subscribe to the weather station
var tempDisplay = new TemperatureDisplay(weatherStation);
var statistics = new WeatherStatistics(weatherStation);

// Update measurements - this will trigger notifications
weatherStation.UpdateMeasurements(25.5m, 65.2m, 1013.2m);
weatherStation.UpdateMeasurements(26.1m, 67.5m, 1012.8m);
```

### Command Pattern

Using delegates to implement the command pattern:

```csharp
// Command interface using delegates
public class Command
{
    private readonly Action _execute;
    private readonly Func<bool> _canExecute;
    
    public Command(Action execute, Func<bool> canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute ?? (() => true);
    }
    
    public bool CanExecute() => _canExecute();
    
    public void Execute()
    {
        if (CanExecute())
        {
            _execute();
        }
    }
}

// Usage
public class TextEditor
{
    private string _text = string.Empty;
    private Stack<string> _undoStack = new Stack<string>();
    
    public string Text
    {
        get => _text;
        set => _text = value;
    }
    
    // Commands
    public Command UndoCommand { get; }
    public Command PasteCommand { get; }
    public Command CutCommand { get; }
    
    public TextEditor()
    {
        UndoCommand = new Command(
            execute: () => 
            {
                if (_undoStack.Count > 0)
                {
                    _text = _undoStack.Pop();
                }
            },
            canExecute: () => _undoStack.Count > 0
        );
        
        PasteCommand = new Command(
            execute: () => 
            {
                SaveState();
                _text += Clipboard.GetText();
            },
            canExecute: () => Clipboard.ContainsText()
        );
        
        CutCommand = new Command(
            execute: () => 
            {
                if (!string.IsNullOrEmpty(_text))
                {
                    SaveState();
                    Clipboard.SetText(_text);
                    _text = string.Empty;
                }
            },
            canExecute: () => !string.IsNullOrEmpty(_text)
        );
    }
    
    private void SaveState()
    {
        _undoStack.Push(_text);
    }
}

// Client code
var editor = new TextEditor();
editor.Text = "Hello, World!";

// Execute commands
if (editor.CutCommand.CanExecute())
{
    editor.CutCommand.Execute();
    Console.WriteLine($"After Cut: '{editor.Text}'"); // Empty string
}

if (editor.PasteCommand.CanExecute())
{
    editor.PasteCommand.Execute();
    Console.WriteLine($"After Paste: '{editor.Text}'"); // "Hello, World!"
}

if (editor.UndoCommand.CanExecute())
{
    editor.UndoCommand.Execute();
    Console.WriteLine($"After Undo: '{editor.Text}'"); // Empty string
}
```

### Event Aggregator

Implementing an event aggregator pattern for decoupled event communication:

```csharp
// Message base class
public abstract class Message { }

// Concrete messages
public class OrderCreatedMessage : Message
{
    public int OrderId { get; }
    public decimal Amount { get; }
    
    public OrderCreatedMessage(int orderId, decimal amount)
    {
        OrderId = orderId;
        Amount = amount;
    }
}

public class PaymentProcessedMessage : Message
{
    public int OrderId { get; }
    public bool Success { get; }
    
    public PaymentProcessedMessage(int orderId, bool success)
    {
        OrderId = orderId;
        Success = success;
    }
}

// Event aggregator
public class EventAggregator
{
    private readonly Dictionary<Type, List<object>> _subscribers = new Dictionary<Type, List<object>>();
    
    // Subscribe to a message type
    public void Subscribe<T>(Action<T> handler) where T : Message
    {
        var type = typeof(T);
        
        if (!_subscribers.ContainsKey(type))
        {
            _subscribers[type] = new List<object>();
        }
        
        _subscribers[type].Add(handler);
    }
    
    // Unsubscribe from a message type
    public void Unsubscribe<T>(Action<T> handler) where T : Message
    {
        var type = typeof(T);
        
        if (_subscribers.ContainsKey(type))
        {
            _subscribers[type].Remove(handler);
        }
    }
    
    // Publish a message
    public void Publish<T>(T message) where T : Message
    {
        var type = typeof(T);
        
        if (_subscribers.ContainsKey(type))
        {
            // Make a copy of the subscribers list to allow handlers to unsubscribe during notification
            var handlers = _subscribers[type].ToList();
            
            foreach (var handler in handlers)
            {
                ((Action<T>)handler)(message);
            }
        }
    }
}

// Usage
public class OrderService
{
    private readonly EventAggregator _eventAggregator;
    
    public OrderService(EventAggregator eventAggregator)
    {
        _eventAggregator = eventAggregator;
    }
    
    public void CreateOrder(int orderId, decimal amount)
    {
        // Create the order
        Console.WriteLine($"Order {orderId} created for ${amount}");
        
        // Publish the event
        _eventAggregator.Publish(new OrderCreatedMessage(orderId, amount));
    }
}

public class PaymentService
{
    private readonly EventAggregator _eventAggregator;
    
    public PaymentService(EventAggregator eventAggregator)
    {
        _eventAggregator = eventAggregator;
        
        // Subscribe to order created events
        _eventAggregator.Subscribe<OrderCreatedMessage>(HandleOrderCreated);
    }
    
    private void HandleOrderCreated(OrderCreatedMessage message)
    {
        // Process payment for the order
        Console.WriteLine($"Processing payment for order {message.OrderId} of ${message.Amount}");
        
        // Simulate payment processing
        bool success = message.Amount < 1000; // Simplified validation
        
        // Publish payment processed event
        _eventAggregator.Publish(new PaymentProcessedMessage(message.OrderId, success));
    }
}

public class NotificationService
{
    private readonly EventAggregator _eventAggregator;
    
    public NotificationService(EventAggregator eventAggregator)
    {
        _eventAggregator = eventAggregator;
        
        // Subscribe to messages
        _eventAggregator.Subscribe<OrderCreatedMessage>(HandleOrderCreated);
        _eventAggregator.Subscribe<PaymentProcessedMessage>(HandlePaymentProcessed);
    }
    
    private void HandleOrderCreated(OrderCreatedMessage message)
    {
        Console.WriteLine($"Notification: New order {message.OrderId} created");
    }
    
    private void HandlePaymentProcessed(PaymentProcessedMessage message)
    {
        if (message.Success)
        {
            Console.WriteLine($"Notification: Payment for order {message.OrderId} was successful");
        }
        else
        {
            Console.WriteLine($"Notification: Payment for order {message.OrderId} failed");
        }
    }
}

// Application startup
var eventAggregator = new EventAggregator();
var orderService = new OrderService(eventAggregator);
var paymentService = new PaymentService(eventAggregator);
var notificationService = new NotificationService(eventAggregator);

// Create an order
orderService.CreateOrder(12345, 499.99m);
```

## Performance Considerations

### Delegate Allocation and Caching

Creating delegates has a performance cost. When possible, cache delegate instances:

```csharp
// Bad: Creates a new delegate every time
public void ProcessItems(List<int> items)
{
    items.ForEach(item => Console.WriteLine(item));
}

// Better: Cache the delegate
private readonly Action<int> _processAction = item => Console.WriteLine(item);

public void ProcessItems(List<int> items)
{
    items.ForEach(_processAction);
}
```

### Event Handler Subscription Management

Failing to unsubscribe from events can cause memory leaks:

```csharp
// Example of a memory leak
public class Subscriber
{
    private Publisher _publisher;
    
    public Subscriber(Publisher publisher)
    {
        _publisher = publisher;
        
        // Subscribe to the event
        _publisher.DataChanged += HandleDataChanged;
    }
    
    private void HandleDataChanged(object sender, EventArgs e)
    {
        Console.WriteLine("Data changed");
    }
    
    // Missing: Unsubscribe from the event when done
}

// Proper cleanup
public class BetterSubscriber : IDisposable
{
    private Publisher _publisher;
    private bool _disposed = false;
    
    public BetterSubscriber(Publisher publisher)
    {
        _publisher = publisher;
        
        // Subscribe to the event
        _publisher.DataChanged += HandleDataChanged;
    }
    
    private void HandleDataChanged(object sender, EventArgs e)
    {
        Console.WriteLine("Data changed");
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
            if (disposing && _publisher != null)
            {
                // Unsubscribe from the event
                _publisher.DataChanged -= HandleDataChanged;
                _publisher = null;
            }
            
            _disposed = true;
        }
    }
}
```

### Weak Event Pattern

The weak event pattern prevents memory leaks by allowing subscribers to be garbage collected:

```csharp
// Simplified weak event manager
public class WeakEventManager<TEventArgs> where TEventArgs : EventArgs
{
    private readonly Dictionary<string, List<WeakReference>> _eventHandlers = 
        new Dictionary<string, List<WeakReference>>();
        
    public void AddHandler(string eventName, EventHandler<TEventArgs> handler)
    {
        if (!_eventHandlers.TryGetValue(eventName, out var handlers))
        {
            handlers = new List<WeakReference>();
            _eventHandlers[eventName] = handlers;
        }
        
        handlers.Add(new WeakReference(handler));
    }
    
    public void RemoveHandler(string eventName, EventHandler<TEventArgs> handler)
    {
        if (_eventHandlers.TryGetValue(eventName, out var handlers))
        {
            for (int i = handlers.Count - 1; i >= 0; i--)
            {
                var weakReference = handlers[i];
                var target = weakReference.Target as EventHandler<TEventArgs>;
                
                if (!weakReference.IsAlive || target == handler)
                {
                    handlers.RemoveAt(i);
                }
            }
        }
    }
    
    public void RaiseEvent(object sender, string eventName, TEventArgs e)
    {
        if (_eventHandlers.TryGetValue(eventName, out var handlers))
        {
            // Create a copy of the handlers list
            var handlersCopy = new List<WeakReference>(handlers);
            
            // Remove dead handlers
            for (int i = handlers.Count - 1; i >= 0; i--)
            {
                if (!handlers[i].IsAlive)
                {
                    handlers.RemoveAt(i);
                }
            }
            
            // Invoke live handlers
            foreach (var weakReference in handlersCopy)
            {
                if (weakReference.Target is EventHandler<TEventArgs> handler && weakReference.IsAlive)
                {
                    handler(sender, e);
                }
            }
        }
    }
}

// Usage
public class Publisher
{
    private readonly WeakEventManager<EventArgs> _eventManager = new WeakEventManager<EventArgs>();
    
    public void AddDataChangedHandler(EventHandler<EventArgs> handler)
    {
        _eventManager.AddHandler("DataChanged", handler);
    }
    
    public void RemoveDataChangedHandler(EventHandler<EventArgs> handler)
    {
        _eventManager.RemoveHandler("DataChanged", handler);
    }
    
    public void OnDataChanged()
    {
        _eventManager.RaiseEvent(this, "DataChanged", EventArgs.Empty);
    }
}
```

## Common Pitfalls

### Event Handler Memory Leaks

```csharp
// Common issue: creating event handlers that capture the containing instance
public class MemoryLeakExample
{
    public MemoryLeakExample(LongLivedObject longLivedObject)
    {
        // This creates a strong reference from LongLivedObject back to this instance
        longLivedObject.SomeEvent += (sender, e) => 
        {
            // This closure captures 'this'
            this.HandleEvent();
        };
    }
    
    private void HandleEvent()
    {
        Console.WriteLine("Event handled");
    }
}

// Solution: Explicitly unsubscribe when done
public class FixedExample : IDisposable
{
    private readonly LongLivedObject _longLivedObject;
    private readonly EventHandler _handler;
    
    public FixedExample(LongLivedObject longLivedObject)
    {
        _longLivedObject = longLivedObject;
        
        // Store the handler as a field so we can unsubscribe later
        _handler = (sender, e) => HandleEvent();
        _longLivedObject.SomeEvent += _handler;
    }
    
    private void HandleEvent()
    {
        Console.WriteLine("Event handled");
    }
    
    public void Dispose()
    {
        // Unsubscribe when done
        _longLivedObject.SomeEvent -= _handler;
    }
}
```

### Thread Safety Issues

Events and delegates aren't inherently thread-safe:

```csharp
// Thread-safety issue when raising events
public event EventHandler DataChanged;

// Unsafe event invocation - race condition if event handlers are being added/removed concurrently
public void OnDataChanged()
{
    DataChanged?.Invoke(this, EventArgs.Empty);
}

// Thread-safe event invocation
public void OnDataChangedSafe()
{
    // Copy the delegate reference to a local variable
    var handler = DataChanged;
    
    // Check and invoke the local copy
    handler?.Invoke(this, EventArgs.Empty);
}
```

### Exception Handling in Event Handlers

Exceptions in event handlers can disrupt the normal flow:

```csharp
// Without exception handling
public void RaiseEvent()
{
    // If any handler throws, subsequent handlers won't be called
    SomeEvent?.Invoke(this, EventArgs.Empty);
}

// With exception handling
public void RaiseEventSafely()
{
    var handlers = SomeEvent;
    if (handlers != null)
    {
        foreach (EventHandler handler in handlers.GetInvocationList())
        {
            try
            {
                handler(this, EventArgs.Empty);
            }
            catch (Exception ex)
            {
                // Log the exception but continue with other handlers
                Console.WriteLine($"Exception in event handler: {ex.Message}");
            }
        }
    }
}
```

### Misusing Delegate Variance

```csharp
// Incorrect use of delegate variance
public class Animal { }
public class Dog : Animal { }
public class Cat : Animal { }

// This will compile but may cause issues at runtime
public void ProcessAnimals()
{
    Func<Animal> animalFactory = () => new Animal();
    Func<Dog> dogFactory = () => new Dog();
    
    // Valid due to covariance
    animalFactory = dogFactory;
    
    // Type confusion: trying to add a cat to a list of dogs
    Action<List<Dog>> addDog = dogs => dogs.Add(new Dog());
    Action<List<Animal>> addAnimal = animals => animals.Add(new Cat());
    
    // This assignment appears valid due to contravariance
    addDog = addAnimal; // But it's actually a problem
    
    // This will add a Cat to a List<Dog> at runtime!
    List<Dog> dogs = new List<Dog>();
    addDog(dogs); // Runtime error or type confusion
}
```

### Event Ordering Dependencies

Relying on event handler execution order is risky:

```csharp
// Don't rely on handler execution order
public class OrderProcessor
{
    public event EventHandler<OrderEventArgs> OrderProcessed;
    
    public void ProcessOrder(Order order)
    {
        // Process the order
        
        // Raise the event - handlers will execute in the order they were added,
        // but this is not guaranteed in all cases
        OrderProcessed?.Invoke(this, new OrderEventArgs(order));
    }
}

// Instead, design for independence
public class BetterOrderProcessor
{
    // Separate events for different stages
    public event EventHandler<OrderEventArgs> OrderValidated;
    public event EventHandler<OrderEventArgs> PaymentProcessed;
    public event EventHandler<OrderEventArgs> OrderFulfilled;
    
    public void ProcessOrder(Order order)
    {
        // Validate
        ValidateOrder(order);
        OrderValidated?.Invoke(this, new OrderEventArgs(order));
        
        // Process payment
        ProcessPayment(order);
        PaymentProcessed?.Invoke(this, new OrderEventArgs(order));
        
        // Fulfill order
        FulfillOrder(order);
        OrderFulfilled?.Invoke(this, new OrderEventArgs(order));
    }
    
    // Implementation methods
    private void ValidateOrder(Order order) { /* ... */ }
    private void ProcessPayment(Order order) { /* ... */ }
    private void FulfillOrder(Order order) { /* ... */ }
}
```

## Best Practices

### Event Design Guidelines

1. **Event Naming**:
   - Name events with a verb in the past tense to indicate that the action has already happened (e.g., `OrderProcessed`, `DataLoaded`)
   - Use the suffix "Changed" for property change notifications (e.g., `StatusChanged`)

2. **Event Argument Design**:
   - Derive from `EventArgs` for all event data
   - Design event args to be immutable
   - Include all relevant information in the event args
   - Use the naming convention `<Event>EventArgs` (e.g., `OrderProcessedEventArgs`)

3. **Event Access Modifiers**:
   - Declare events as public for external subscribers
   - Make event raising methods protected and virtual to allow overriding in derived classes
   - Follow the pattern of `On<EventName>` for event raising methods

```csharp
// Good event design
public class Order
{
    // Public event with standard EventHandler<T> pattern
    public event EventHandler<OrderStatusChangedEventArgs> StatusChanged;
    
    private OrderStatus _status;
    public OrderStatus Status
    {
        get => _status;
        set
        {
            if (_status != value)
            {
                OrderStatus oldStatus = _status;
                _status = value;
                OnStatusChanged(new OrderStatusChangedEventArgs(oldStatus, _status));
            }
        }
    }
    
    // Protected virtual method for raising the event
    protected virtual void OnStatusChanged(OrderStatusChangedEventArgs e)
    {
        StatusChanged?.Invoke(this, e);
    }
}

// Immutable event args
public class OrderStatusChangedEventArgs : EventArgs
{
    public OrderStatus OldStatus { get; }
    public OrderStatus NewStatus { get; }
    
    public OrderStatusChangedEventArgs(OrderStatus oldStatus, OrderStatus newStatus)
    {
        OldStatus = oldStatus;
        NewStatus = newStatus;
    }
}
```

### Delegate Best Practices

1. **Prefer built-in delegate types**:
   - Use `Action<T>`, `Func<T>`, and `Predicate<T>` instead of creating custom delegate types
   - Create custom delegates only when needed for specialized requirements or API clarity

2. **Method group conversion**:
   - Use method group conversion instead of lambda expressions for simple cases
   - `button.Click += HandleClick;` instead of `button.Click += (s, e) => HandleClick(s, e);`

3. **Avoid capturing unnecessary variables in lambdas**:
   - Be aware of what variables your lambda expressions capture
   - Minimize the closure scope to reduce memory usage

```csharp
// Method group conversion (preferred)
button.Click += HandleButtonClick;

private void HandleButtonClick(object sender, EventArgs e)
{
    // Handle event
}

// Unnecessary variable capture
private void ProcessItems(List<Item> items, Logger logger)
{
    // Bad: Captures 'logger' even though it's not used in the lambda
    items.ForEach(item => ProcessItem(item));
    
    // Good: No variable capture
    foreach (var item in items)
    {
        ProcessItem(item);
    }
    
    private void ProcessItem(Item item)
    {
        // Process the item
    }
}
```

### Thread Safety

1. **Safe event invocation**:
   - Copy the delegate to a local variable before checking and invoking it
   - Use `Interlocked` operations when subscribing to or unsubscribing from events in multi-threaded scenarios

2. **Thread synchronization**:
   - Use synchronization primitives when accessing shared state in event handlers
   - Consider using `SynchronizationContext` for UI thread marshaling

```csharp
// Thread-safe event access
public event EventHandler StateChanged;

public void UpdateState()
{
    // Some state update logic
    
    // Thread-safe event invocation
    var handler = StateChanged;
    handler?.Invoke(this, EventArgs.Empty);
}

// UI thread marshaling
public void ProcessData(Data data)
{
    // Run on background thread
    Task.Run(() => 
    {
        // Do CPU-intensive work
        var result = ProcessDataInBackground(data);
        
        // Marshal back to UI thread
        _synchronizationContext.Post(_ => 
        {
            // Update UI with result
            UpdateUI(result);
        }, null);
    });
}
```

### Memory Management

1. **Unsubscribe from events**:
   - Always unsubscribe from events when the subscriber is no longer needed
   - Implement `IDisposable` for proper cleanup
   - Consider weak event patterns for complex event hierarchies

2. **Avoid strong references in event handlers**:
   - Be cautious of closures capturing the containing instance
   - Use weak references or event aggregators for loosely coupled systems

```csharp
// Proper event subscription management
public class Subscriber : IDisposable
{
    private readonly EventSource _source;
    private bool _disposed = false;
    
    public Subscriber(EventSource source)
    {
        _source = source;
        _source.DataChanged += HandleDataChanged;
    }
    
    private void HandleDataChanged(object sender, EventArgs e)
    {
        // Handle the event
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing && _source != null)
            {
                // Unsubscribe from events
                _source.DataChanged -= HandleDataChanged;
            }
            
            _disposed = true;
        }
    }
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

## Further Reading

- [C# Events and Delegates Tutorial](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/events/)
- [Events (C# Programming Guide)](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/events/)
- [Delegates (C# Programming Guide)](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/)
- [Event-based Asynchronous Pattern (EAP)](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/event-based-asynchronous-pattern-eap)
- [C# in Depth by Jon Skeet](https://csharpindepth.com/)
- [CLR via C# by Jeffrey Richter](https://www.microsoftpressstore.com/store/clr-via-c-sharp-9780735667457)

## Related Topics

- [C# Lambda Expressions](../advanced/lambda-expressions.md)
- [C# Asynchronous Programming](../advanced/async-await.md)
- [C# Generics](../advanced/generics.md)
- [Design Patterns - Observer Pattern](../../architecture/patterns/observer.md)
- [Design Patterns - Command Pattern](../../architecture/patterns/command.md)
---
title: "C# Basic Syntax"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["c#", "syntax", "basics", "structure", "beginner"]
difficulty: "beginner"
---

# C# Basic Syntax

## Overview

C# is a statically-typed, object-oriented programming language developed by Microsoft. This document covers the fundamental syntax elements of C# that form the building blocks of any C# program. Understanding these basics is essential before diving into more complex language features.

## Core Concepts

### Program Structure

Every C# application starts with at least one file containing a class with a `Main` method, which serves as the entry point for the application:

```csharp
using System;

namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

The components of this basic program are:

- `using System;` - A directive that imports the System namespace
- `namespace HelloWorld` - Declares a namespace for your code
- `class Program` - Defines a class named Program
- `static void Main(string[] args)` - The entry point method
- `Console.WriteLine("Hello, World!");` - A statement that outputs text

### Statements and Expressions

C# code is composed of statements and expressions:

- **Statements** perform actions and are terminated with semicolons
- **Expressions** produce values

```csharp
// Statement
int x = 10;

// Expression within a statement
int y = x * 2;

// Statement calling a method
Console.WriteLine(y);
```

### Case Sensitivity

C# is case-sensitive, meaning `myVariable` and `MyVariable` are treated as distinct identifiers:

```csharp
int myVariable = 10;
int MyVariable = 20; // Different variable
```

### Whitespace and Line Breaks

C# generally ignores whitespace and line breaks, allowing you to format your code for readability:

```csharp
// These are equivalent:
int sum = a + b;

int sum =
    a +
    b;
```

### Code Blocks

Code blocks are enclosed in curly braces `{}` and define the scope of variables and control structures:

```csharp
if (condition)
{
    // This is a code block
    int localVariable = 10; // Scope limited to this block
}
// localVariable is not accessible here
```

### Comments

C# supports three types of comments:

```csharp
// Single-line comment

/* Multi-line
   comment */

/// <summary>
/// XML documentation comment for generating documentation
/// </summary>
public void DocumentedMethod()
{
    // Method implementation
}
```

## Identifiers

Identifiers are names given to variables, methods, classes, etc. C# has rules for valid identifiers:

- Must begin with a letter or underscore
- Can contain letters, digits, and underscores
- Cannot be a C# keyword (unless prefixed with `@`)
- Are case-sensitive

```csharp
// Valid identifiers
int age;
string userName;
double _value;
bool @class; // Using @ to use a keyword as an identifier

// Invalid identifiers
// int 1value;   // Cannot start with a digit
// double my-var; // Cannot contain hyphens
```

## C# Keywords

C# has reserved keywords that have special meaning and cannot be used as identifiers without the `@` prefix:

```csharp
abstract, as, base, bool, break, byte, case, catch, char, checked,
class, const, continue, decimal, default, delegate, do, double, else,
enum, event, explicit, extern, false, finally, fixed, float, for,
foreach, goto, if, implicit, in, int, interface, internal, is, lock,
long, namespace, new, null, object, operator, out, override, params,
private, protected, public, readonly, ref, return, sbyte, sealed,
short, sizeof, stackalloc, static, string, struct, switch, this, throw,
true, try, typeof, uint, ulong, unchecked, unsafe, ushort, using,
virtual, void, volatile, while
```

## Best Practices

- **Consistent Indentation**: Use 4 spaces (or tab configured as 4 spaces) for indentation
- **Meaningful Names**: Choose descriptive names for variables, methods, and classes
- **One Statement Per Line**: For improved readability, put each statement on its own line
- **Proper Spacing**: Use spaces around operators and after commas
- **Proper Bracing Style**: C# typically uses Allman style (opening brace on a new line)

```csharp
// Good formatting example
public int CalculateSum(int a, int b)
{
    int result = a + b;
    return result;
}
```

## Common Pitfalls

### Forgetting Semicolons

One of the most common syntax errors is forgetting to terminate statements with semicolons:

```csharp
int x = 10; // Correct
int y = 20  // Will cause a compile error
```

### Mismatched Braces

Ensure that each opening brace `{` has a corresponding closing brace `}`:

```csharp
if (condition)
{
    // Code block
} // Matching closing brace is essential
```

### Using Keywords as Identifiers

Attempting to use keywords as identifiers without the `@` prefix:

```csharp
int class = 10; // Error: 'class' is a keyword
int @class = 10; // Correct: using @ prefix
```

## Code Examples

### Complete Program Example

```csharp
using System;

namespace SyntaxExamples
{
    class Program
    {
        static void Main(string[] args)
        {
            // Variable declarations
            int number = 10;
            string greeting = "Hello";
            
            // Simple if statement
            if (number > 5)
            {
                Console.WriteLine($"{greeting}, the number {number} is greater than 5!");
            }
            else
            {
                Console.WriteLine($"{greeting}, the number {number} is 5 or less!");
            }
            
            // Simple loop
            for (int i = 0; i < 3; i++)
            {
                Console.WriteLine($"Loop iteration: {i}");
            }
            
            // Calling a method
            int result = AddNumbers(5, 7);
            Console.WriteLine($"The sum of 5 and 7 is {result}");
            
            // Wait for user input before closing
            Console.WriteLine("Press any key to exit...");
            Console.ReadKey();
        }
        
        // Method declaration
        static int AddNumbers(int a, int b)
        {
            return a + b;
        }
    }
}
```

## Further Reading

- [C# Programming Guide on Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/)
- [C# Coding Conventions](https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- [C# Language Specification](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/)

## Related Topics

- [Program Structure](program-structure.md)
- [Variables and Constants](variables-and-constants.md)
- [Namespaces](namespaces.md)
- [Comments and Documentation](comments-and-documentation.md)
---
title: "Algorithms for C# Developers"
date_created: 2025-04-02
date_updated: 2025-04-02
authors: ["Repository Maintainers"]
tags: ["algorithms", "sorting", "searching", "recursion", "dynamic programming", "performance"]
difficulty: "intermediate"
---

# Algorithms for C# Developers

## Overview

Algorithms are the fundamental building blocks of software engineering, providing systematic methods for solving problems efficiently. This guide covers essential algorithms that every C# developer should understand, including implementations, complexity analysis, and practical usage scenarios. Understanding these algorithms will enhance your problem-solving skills and enable you to write more efficient code.

## Complexity Analysis

Before diving into specific algorithms, it's important to understand how we analyze and compare algorithm efficiency.

### Big O Notation

Big O notation describes an algorithm's worst-case time or space requirements as input size grows:

- **O(1)** - Constant time (e.g., array access)
- **O(log n)** - Logarithmic time (e.g., binary search)
- **O(n)** - Linear time (e.g., linear search)
- **O(n log n)** - Linearithmic time (e.g., efficient sorting algorithms)
- **O(n²)** - Quadratic time (e.g., simple sorting algorithms)
- **O(2^n)** - Exponential time (e.g., recursive solutions without memoization)

```csharp
// O(1) - Constant time
public bool IsFirstElementNull<T>(T[] array)
{
    return array[0] == null;


### Longest Common Subsequence (LCS)

Finding the longest subsequence common to two sequences:

```csharp
public string LongestCommonSubsequence(string text1, string text2)
{
    int m = text1.Length;
    int n = text2.Length;
    
    // Create DP table
    int[,] dp = new int[m + 1, n + 1];
    
    // Fill the DP table
    for (int i = 1; i <= m; i++)
    {
        for (int j = 1; j <= n; j++)
        {
            if (text1[i - 1] == text2[j - 1])
            {
                dp[i, j] = dp[i - 1, j - 1] + 1;
            }
            else
            {
                dp[i, j] = Math.Max(dp[i - 1, j], dp[i, j - 1]);
            }
        }
    }
    
    // Reconstruct the LCS
    StringBuilder lcs = new StringBuilder();
    int x = m, y = n;
    
    while (x > 0 && y > 0)
    {
        if (text1[x - 1] == text2[y - 1])
        {
            lcs.Insert(0, text1[x - 1]);
            x--;
            y--;
        }
        else if (dp[x - 1, y] > dp[x, y - 1])
        {
            x--;
        }
        else
        {
            y--;
        }
    }
    
    return lcs.ToString();
}
```

### Knapsack Problem

Classic optimization problem:

```csharp
public int Knapsack(int[] weights, int[] values, int capacity)
{
    int n = weights.Length;
    int[,] dp = new int[n + 1, capacity + 1];
    
    for (int i = 1; i <= n; i++)
    {
        for (int w = 0; w <= capacity; w++)
        {
            if (weights[i - 1] <= w)
            {
                // Include this item
                int includeValue = values[i - 1] + dp[i - 1, w - weights[i - 1]];
                // Exclude this item
                int excludeValue = dp[i - 1, w];
                // Take maximum
                dp[i, w] = Math.Max(includeValue, excludeValue);
            }
            else
            {
                // Can't include this item
                dp[i, w] = dp[i - 1, w];
            }
        }
    }
    
    return dp[n, capacity];
}
```

### Coin Change Problem

Find the minimum number of coins to make a given amount:

```csharp
public int CoinChange(int[] coins, int amount)
{
    // dp[i] will store the minimum number of coins needed to make amount i
    int[] dp = new int[amount + 1];
    
    // Initialize with a value larger than any possible answer
    for (int i = 1; i <= amount; i++)
    {
        dp[i] = amount + 1;
    }
    
    dp[0] = 0; // 0 coins needed to make 0 amount
    
    for (int i = 1; i <= amount; i++)
    {
        foreach (int coin in coins)
        {
            if (coin <= i)
            {
                dp[i] = Math.Min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    
    return dp[amount] > amount ? -1 : dp[amount];
}
```

## Greedy Algorithms

Greedy algorithms make locally optimal choices at each step with the hope of finding a global optimum.

### Activity Selection

Select the maximum number of non-overlapping activities:

```csharp
public List<int> ActivitySelection(int[] start, int[] finish)
{
    int n = start.Length;
    
    // Create activity objects and sort by finish time
    var activities = new List<(int index, int start, int finish)>();
    for (int i = 0; i < n; i++)
    {
        activities.Add((i, start[i], finish[i]));
    }
    
    activities.Sort((a, b) => a.finish.CompareTo(b.finish));
    
    List<int> selected = new List<int>();
    
    // Select first activity
    selected.Add(activities[0].index);
    int lastFinishTime = activities[0].finish;
    
    // Consider rest of the activities
    for (int i = 1; i < n; i++)
    {
        // If this activity starts after the finish time of previously selected activity
        if (activities[i].start >= lastFinishTime)
        {
            selected.Add(activities[i].index);
            lastFinishTime = activities[i].finish;
        }
    }
    
    return selected;
}
```

### Fractional Knapsack

Unlike 0/1 Knapsack, items can be broken into fractions:

```csharp
public double FractionalKnapsack(int[] weights, int[] values, int capacity)
{
    int n = weights.Length;
    
    // Create value-to-weight ratios
    var items = new List<(int index, double ratio)>();
    for (int i = 0; i < n; i++)
    {
        items.Add((i, (double)values[i] / weights[i]));
    }
    
    // Sort by value-to-weight ratio in descending order
    items.Sort((a, b) => b.ratio.CompareTo(a.ratio));
    
    double totalValue = 0;
    int remainingCapacity = capacity;
    
    foreach (var item in items)
    {
        int index = item.index;
        
        if (weights[index] <= remainingCapacity)
        {
            // Take the whole item
            totalValue += values[index];
            remainingCapacity -= weights[index];
        }
        else
        {
            // Take a fraction of the item
            double fraction = (double)remainingCapacity / weights[index];
            totalValue += values[index] * fraction;
            break; // Knapsack is full
        }
    }
    
    return totalValue;
}
```

### Huffman Coding

Prefix-free encoding algorithm for data compression:

```csharp
public class HuffmanNode
{
    public char Character { get; set; }
    public int Frequency { get; set; }
    public HuffmanNode Left { get; set; }
    public HuffmanNode Right { get; set; }
    
    public bool IsLeaf => Left == null && Right == null;
}

public class HuffmanCoding
{
    private Dictionary<char, string> _encodingMap;
    
    public Dictionary<char, string> BuildEncodingMap(string text)
    {
        // Count frequencies
        Dictionary<char, int> frequencies = new Dictionary<char, int>();
        foreach (char c in text)
        {
            if (!frequencies.ContainsKey(c))
                frequencies[c] = 0;
            frequencies[c]++;
        }
        
        // Create priority queue with nodes
        var priorityQueue = new List<HuffmanNode>();
        foreach (var kvp in frequencies)
        {
            priorityQueue.Add(new HuffmanNode
            {
                Character = kvp.Key,
                Frequency = kvp.Value
            });
        }
        
        // Build Huffman tree
        while (priorityQueue.Count > 1)
        {
            // Sort by frequency (simulating a priority queue)
            priorityQueue.Sort((a, b) => a.Frequency.CompareTo(b.Frequency));
            
            // Take two nodes with lowest frequencies
            var left = priorityQueue[0];
            var right = priorityQueue[1];
            priorityQueue.RemoveRange(0, 2);
            
            // Create a new internal node
            var parent = new HuffmanNode
            {
                Character = '\0', // Internal node doesn't represent a character
                Frequency = left.Frequency + right.Frequency,
                Left = left,
                Right = right
            };
            
            priorityQueue.Add(parent);
        }
        
        // Get the root of Huffman tree
        HuffmanNode root = priorityQueue[0];
        
        // Generate codes
        _encodingMap = new Dictionary<char, string>();
        GenerateCode(root, "");
        
        return _encodingMap;
    }
    
    private void GenerateCode(HuffmanNode node, string code)
    {
        if (node == null)
            return;
            
        if (node.IsLeaf)
        {
            _encodingMap[node.Character] = code.Length > 0 ? code : "0"; // Special case for a single character
            return;
        }
        
        GenerateCode(node.Left, code + "0");
        GenerateCode(node.Right, code + "1");
    }
    
    public string Encode(string text)
    {
        if (_encodingMap == null)
            BuildEncodingMap(text);
            
        StringBuilder encoded = new StringBuilder();
        foreach (char c in text)
        {
            encoded.Append(_encodingMap[c]);
        }
        
        return encoded.ToString();
    }
}
```

## String Algorithms

### String Matching

#### Naive String Matching

```csharp
public List<int> NaiveStringMatch(string text, string pattern)
{
    List<int> occurrences = new List<int>();
    int n = text.Length;
    int m = pattern.Length;
    
    for (int i = 0; i <= n - m; i++)
    {
        int j;
        for (j = 0; j < m; j++)
        {
            if (text[i + j] != pattern[j])
                break;
        }
        
        if (j == m) // Pattern found
            occurrences.Add(i);
    }
    
    return occurrences;
}
```

#### Knuth-Morris-Pratt (KMP) Algorithm

Efficient string matching with O(n + m) time complexity:

```csharp
public List<int> KMPSearch(string text, string pattern)
{
    List<int> occurrences = new List<int>();
    int n = text.Length;
    int m = pattern.Length;
    
    // Preprocessing: Computing the LPS (Longest Proper Prefix which is also Suffix) array
    int[] lps = ComputeLPSArray(pattern);
    
    int i = 0; // Index for text
    int j = 0; // Index for pattern
    
    while (i < n)
    {
        if (pattern[j] == text[i])
        {
            i++;
            j++;
        }
        
        if (j == m)
        {
            // Pattern found at index i-j
            occurrences.Add(i - j);
            j = lps[j - 1];
        }
        else if (i < n && pattern[j] != text[i])
        {
            if (j != 0)
                j = lps[j - 1];
            else
                i++;
        }
    }
    
    return occurrences;
}

private int[] ComputeLPSArray(string pattern)
{
    int m = pattern.Length;
    int[] lps = new int[m];
    
    int len = 0;
    int i = 1;
    
    while (i < m)
    {
        if (pattern[i] == pattern[len])
        {
            len++;
            lps[i] = len;
            i++;
        }
        else
        {
            if (len != 0)
            {
                len = lps[len - 1];
            }
            else
            {
                lps[i] = 0;
                i++;
            }
        }
    }
    
    return lps;
}
```

### Edit Distance (Levenshtein Distance)

Calculate the minimum number of operations to transform one string into another:

```csharp
public int EditDistance(string word1, string word2)
{
    int m = word1.Length;
    int n = word2.Length;
    
    // Create a DP table
    int[,] dp = new int[m + 1, n + 1];
    
    // Initialize first row and column
    for (int i = 0; i <= m; i++)
        dp[i, 0] = i;
        
    for (int j = 0; j <= n; j++)
        dp[0, j] = j;
        
    // Fill the DP table
    for (int i = 1; i <= m; i++)
    {
        for (int j = 1; j <= n; j++)
        {
            if (word1[i - 1] == word2[j - 1])
            {
                // Characters match, no operation needed
                dp[i, j] = dp[i - 1, j - 1];
            }
            else
            {
                // Minimum of three operations:
                // 1. Replace: dp[i-1, j-1] + 1
                // 2. Insert: dp[i, j-1] + 1
                // 3. Delete: dp[i-1, j] + 1
                dp[i, j] = 1 + Math.Min(dp[i - 1, j - 1], Math.Min(dp[i, j - 1], dp[i - 1, j]));
            }
        }
    }
    
    return dp[m, n];
}
```

### Longest Palindromic Substring

Find the longest substring that is a palindrome:

```csharp
public string LongestPalindromicSubstring(string s)
{
    if (string.IsNullOrEmpty(s))
        return string.Empty;
        
    int start = 0, maxLength = 1;
    int n = s.Length;
    
    // dp[i, j] will be true if substring s[i..j] is palindrome
    bool[,] dp = new bool[n, n];
    
    // All substrings of length 1 are palindromes
    for (int i = 0; i < n; i++)
        dp[i, i] = true;
        
    // Check for substrings of length 2
    for (int i = 0; i < n - 1; i++)
    {
        if (s[i] == s[i + 1])
        {
            dp[i, i + 1] = true;
            start = i;
            maxLength = 2;
        }
    }
    
    // Check for lengths greater than 2
    for (int length = 3; length <= n; length++)
    {
        for (int i = 0; i < n - length + 1; i++)
        {
            // j is ending index of substring
            int j = i + length - 1;
            
            // Check if s[i+1..j-1] is palindrome and characters at i and j match
            if (dp[i + 1, j - 1] && s[i] == s[j])
            {
                dp[i, j] = true;
                
                if (length > maxLength)
                {
                    start = i;
                    maxLength = length;
                }
            }
        }
    }
    
    return s.Substring(start, maxLength);
}
```

## Backtracking Algorithms

Backtracking is an algorithmic technique for solving problems recursively by trying to build a solution incrementally.

### N-Queens Problem

Place N queens on an N×N chessboard so that no two queens threaten each other:

```csharp
public class NQueens
{
    private int _n;
    private List<List<string>> _solutions;
    
    public List<List<string>> SolveNQueens(int n)
    {
        _n = n;
        _solutions = new List<List<string>>();
        char[][] board = new char[n][];
        
        // Initialize empty board
        for (int i = 0; i < n; i++)
        {
            board[i] = new char[n];
            for (int j = 0; j < n; j++)
            {
                board[i][j] = '.';
            }
        }
        
        Backtrack(board, 0);
        return _solutions;
    }
    
    private void Backtrack(char[][] board, int row)
    {
        if (row == _n)
        {
            // Found a valid solution
            _solutions.Add(ConstructSolution(board));
            return;
        }
        
        for (int col = 0; col < _n; col++)
        {
            if (IsValid(board, row, col))
            {
                // Place queen
                board[row][col] = 'Q';
                
                // Move to next row
                Backtrack(board, row + 1);
                
                // Backtrack (remove queen)
                board[row][col] = '.';
            }
        }
    }
    
    private bool IsValid(char[][] board, int row, int col)
    {
        // Check column
        for (int i = 0; i < row; i++)
        {
            if (board[i][col] == 'Q')
                return false;
        }
        
        // Check upper-left diagonal
        for (int i = row - 1, j = col - 1; i >= 0 && j >= 0; i--, j--)
        {
            if (board[i][j] == 'Q')
                return false;
        }
        
        // Check upper-right diagonal
        for (int i = row - 1, j = col + 1; i >= 0 && j < _n; i--, j++)
        {
            if (board[i][j] == 'Q')
                return false;
        }
        
        return true;
    }
    
    private List<string> ConstructSolution(char[][] board)
    {
        List<string> solution = new List<string>();
        
        for (int i = 0; i < _n; i++)
        {
            solution.Add(new string(board[i]));
        }
        
        return solution;
    }
}
```

### Sudoku Solver

Solve a 9x9 Sudoku puzzle:

```csharp
public class SudokuSolver
{
    private const int GRID_SIZE = 9;
    
    public bool SolveSudoku(char[][] board)
    {
        for (int row = 0; row < GRID_SIZE; row++)
        {
            for (int col = 0; col < GRID_SIZE; col++)
            {
                // Skip filled cells
                if (board[row][col] != '.')
                    continue;
                    
                // Try digits 1-9
                for (char num = '1'; num <= '9'; num++)
                {
                    if (IsValid(board, row, col, num))
                    {
                        // Place this digit
                        board[row][col] = num;
                        
                        // Recursively try to solve rest of the puzzle
                        if (SolveSudoku(board))
                            return true;
                            
                        // If not successful, backtrack
                        board[row][col] = '.';
                    }
                }
                
                // No valid digit for this cell
                return false;
            }
        }
        
        // All cells filled
        return true;
    }
    
    private bool IsValid(char[][] board, int row, int col, char num)
    {
        // Check row
        for (int x = 0; x < GRID_SIZE; x++)
        {
            if (board[row][x] == num)
                return false;
        }
        
        // Check column
        for (int x = 0; x < GRID_SIZE; x++)
        {
            if (board[x][col] == num)
                return false;
        }
        
        // Check 3x3 box
        int boxRow = row - row % 3;
        int boxCol = col - col % 3;
        
        for (int i = 0; i < 3; i++)
        {
            for (int j = 0; j < 3; j++)
            {
                if (board[boxRow + i][boxCol + j] == num)
                    return false;
            }
        }
        
        return true;
    }
}
```

## Advanced Topics

### Bit Manipulation

Efficient operations using bit manipulation:

```csharp
public class BitManipulation
{
    // Check if n is a power of 2
    public bool IsPowerOfTwo(int n)
    {
        return n > 0 && (n & (n - 1)) == 0;
    }
    
    // Count set bits (1s) in an integer
    public int CountSetBits(int n)
    {
        int count = 0;
        while (n > 0)
        {
            count += n & 1;
            n >>= 1;
        }
        return count;
    }
    
    // Find the only element that appears once
    // (all other elements appear exactly twice)
    public int FindSingleNumber(int[] nums)
    {
        int result = 0;
        foreach (int num in nums)
        {
            result ^= num;
        }
        return result;
    }
    
    // Swap two numbers without using a temporary variable
    public void SwapWithoutTemp(ref int a, ref int b)
    {
        a = a ^ b;
        b = a ^ b;
        a = a ^ b;
    }
    
    // Get the rightmost set bit
    public int GetRightmostSetBit(int n)
    {
        return n & -n;
    }
    
    // Check if bit at position i is set
    public bool IsBitSet(int n, int i)
    {
        return (n & (1 << i)) != 0;
    }
    
    // Set bit at position i
    public int SetBit(int n, int i)
    {
        return n | (1 << i);
    }
    
    // Clear bit at position i
    public int ClearBit(int n, int i)
    {
        return n & ~(1 << i);
    }
    
    // Toggle bit at position i
    public int ToggleBit(int n, int i)
    {
        return n ^ (1 << i);
    }
}
```

### Divide and Conquer

#### Closest Pair of Points

Find the closest pair of points in a 2D plane:

```csharp
public class Point
{
    public double X { get; set; }
    public double Y { get; set; }
    
    public Point(double x, double y)
    {
        X = x;
        Y = y;
    }
    
    public double DistanceTo(Point other)
    {
        double dx = X - other.X;
        double dy = Y - other.Y;
        return Math.Sqrt(dx * dx + dy * dy);
    }
}

public class ClosestPair
{
    public (Point, Point, double) FindClosestPair(Point[] points)
    {
        // Sort points by x-coordinate
        Array.Sort(points, (a, b) => a.X.CompareTo(b.X));
        
        return ClosestPairRecursive(points, 0, points.Length - 1);
    }
    
    private (Point, Point, double) ClosestPairRecursive(Point[] points, int left, int right)
    {
        if (right - left <= 2)
        {
            return BruteForce(points, left, right);
        }
        
        int mid = (left + right) / 2;
        double midX = points[mid].X;
        
        // Recursively find closest pairs in left and right halves
        var leftPair = ClosestPairRecursive(points, left, mid);
        var rightPair = ClosestPairRecursive(points, mid + 1, right);
        
        // Get the minimum distance from left and right
        var minPair = leftPair.Item3 <= rightPair.Item3 ? leftPair : rightPair;
        double delta = minPair.Item3;
        
        // Find points close to the dividing line
        List<Point> strip = new List<Point>();
        for (int i = left; i <= right; i++)
        {
            if (Math.Abs(points[i].X - midX) < delta)
            {
                strip.Add(points[i]);
            }
        }
        
        // Sort strip by y-coordinate
        strip.Sort((a, b) => a.Y.CompareTo(b.Y));
        
        // Find closest pair in the strip
        for (int i = 0; i < strip.Count; i++)
        {
            for (int j = i + 1; j < strip.Count && strip[j].Y - strip[i].Y < delta; j++)
            {
                double distance = strip[i].DistanceTo(strip[j]);
                if (distance < delta)
                {
                    minPair = (strip[i], strip[j], distance);
                    delta = distance;
                }
            }
        }
        
        return minPair;
    }
    
    private (Point, Point, double) BruteForce(Point[] points, int left, int right)
    {
        double minDistance = double.MaxValue;
        Point minPoint1 = null;
        Point minPoint2 = null;
        
        for (int i = left; i <= right; i++)
        {
            for (int j = i + 1; j <= right; j++)
            {
                double distance = points[i].DistanceTo(points[j]);
                if (distance < minDistance)
                {
                    minDistance = distance;
                    minPoint1 = points[i];
                    minPoint2 = points[j];
                }
            }
        }
        
        return (minPoint1, minPoint2, minDistance);
    }
}
```

### Randomized Algorithms

#### Randomized Quick Sort

```csharp
public class RandomizedQuickSort
{
    private Random _random = new Random();
    
    public void Sort<T>(T[] array) where T : IComparable<T>
    {
        QuickSort(array, 0, array.Length - 1);
    }
    
    private void QuickSort<T>(T[] array, int low, int high) where T : IComparable<T>
    {
        if (low < high)
        {
            int pivotIndex = Partition(array, low, high);
            
            QuickSort(array, low, pivotIndex - 1);
            QuickSort(array, pivotIndex + 1, high);
        }
    }
    
    private int Partition<T>(T[] array, int low, int high) where T : IComparable<T>
    {
        // Choose a random pivot
        int randomPivotIndex = _random.Next(low, high + 1);
        Swap(array, randomPivotIndex, high);
        
        T pivot = array[high];
        int i = low - 1;
        
        for (int j = low; j < high; j++)
        {
            if (array[j].CompareTo(pivot) <= 0)
            {
                i++;
                Swap(array, i, j);
            }
        }
        
        Swap(array, i + 1, high);
        return i + 1;
    }
    
    private void Swap<T>(T[] array, int i, int j)
    {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }
}
```

## Best Practices

### Algorithm Selection

Choosing the right algorithm depends on several factors:

1. **Problem characteristics**: Understand the specific requirements and constraints of your problem.
2. **Input size**: Different algorithms perform better at different input sizes.
3. **Expected input patterns**: Some algorithms perform better on partially sorted data, etc.
4. **Time vs. space tradeoffs**: Sometimes you can trade memory for speed or vice versa.
5. **Implementation complexity**: Simpler algorithms might be preferred for maintainability.

```csharp
// Example of algorithm selection for sorting
public static void SortArray<T>(T[] array) where T : IComparable<T>
{
    // For very small arrays, insertion sort is often faster
    if (array.Length <= 20)
    {
        InsertionSort(array);
        return;
    }
    
    // For larger arrays, use more efficient algorithms
    if (IsNearlySorted(array))
    {
        // If data is nearly sorted, insertion sort works well
        InsertionSort(array);
    }
    else
    {
        // For general cases, use a more efficient algorithm
        Array.Sort(array); // Uses introspective sort in .NET
    }
}

private static bool IsNearlySorted<T>(T[] array) where T : IComparable<T>
{
    int inversions = 0;
    int threshold = array.Length / 10; // Arbitrary threshold
    
    for (int i = 0; i < array.Length - 1; i++)
    {
        if (array[i].CompareTo(array[i + 1]) > 0)
        {
            inversions++;
            if (inversions > threshold)
                return false;
        }
    }
    
    return true;
}
```

### Performance Optimization

1. **Use built-in methods**: .NET provides highly optimized implementations of common algorithms.
2. **Measure before optimizing**: Use benchmarking tools to identify actual bottlenecks.
3. **Consider algorithm complexity**: Choose algorithms with appropriate time and space complexity for your use case.
4. **Leverage data structure properties**: Use the right data structure for the specific operations your application performs most frequently.
5. **Cache results**: Store and reuse results of expensive computations.

```csharp
// Example: Using memoization for fibonacci
public class FibonacciCalculator
{
    private Dictionary<int, long> _cache = new Dictionary<int, long>();
    
    public long Calculate(int n)
    {
        // Check if result is in cache
        if (_cache.TryGetValue(n, out long result))
            return result;
            
        // Base cases
        if (n <= 1)
            return n;
            
        // Calculate and cache result
        result = Calculate(n - 1) + Calculate(n - 2);
        _cache[n] = result;
        
        return result;

// O(log n) - Logarithmic time
public int BinarySearch(int[] sortedArray, int target)
{
    int left = 0;
    int right = sortedArray.Length - 1;
    
    while (left <= right)
    {
        int mid = left + (right - left) / 2;
        
        if (sortedArray[mid] == target)
            return mid;
            
        if (sortedArray[mid] < target)
            left = mid + 1;
        else
            right = mid - 1;
    }
    
    return -1;
}

// O(n) - Linear time
public int FindMax(int[] array)
{
    int max = array[0];
    
    for (int i = 1; i < array.Length; i++)
    {
        if (array[i] > max)
            max = array[i];
    }
    
    return max;
}

// O(n²) - Quadratic time
public void BubbleSort(int[] array)
{
    for (int i = 0; i < array.Length; i++)
    {
        for (int j = 0; j < array.Length - i - 1; j++)
        {
            if (array[j] > array[j + 1])
            {
                (array[j], array[j + 1]) = (array[j + 1], array[j]);
            }
        }
    }
}
```

### Space Complexity

Space complexity measures the additional memory an algorithm requires:

```csharp
// O(1) space - Constant space
public int Sum(int[] array)
{
    int sum = 0;
    
    for (int i = 0; i < array.Length; i++)
    {
        sum += array[i];
    }
    
    return sum;
}

// O(n) space - Linear space
public int[] DoubleValues(int[] array)
{
    int[] result = new int[array.Length]; // O(n) space
    
    for (int i = 0; i < array.Length; i++)
    {
        result[i] = array[i] * 2;
    }
    
    return result;
}
```

## Sorting Algorithms

### Bubble Sort

Simple but inefficient, with O(n²) time complexity:

```csharp
public static void BubbleSort<T>(T[] array) where T : IComparable<T>
{
    bool swapped;
    for (int i = 0; i < array.Length - 1; i++)
    {
        swapped = false;
        for (int j = 0; j < array.Length - i - 1; j++)
        {
            if (array[j].CompareTo(array[j + 1]) > 0)
            {
                // Swap elements
                (array[j], array[j + 1]) = (array[j + 1], array[j]);
                swapped = true;
            }
        }
        
        // If no swapping occurred in this pass, array is sorted
        if (!swapped)
            break;
    }
}
```

### Selection Sort

Simple algorithm with O(n²) time complexity:

```csharp
public static void SelectionSort<T>(T[] array) where T : IComparable<T>
{
    for (int i = 0; i < array.Length - 1; i++)
    {
        // Find minimum element in unsorted portion
        int minIndex = i;
        for (int j = i + 1; j < array.Length; j++)
        {
            if (array[j].CompareTo(array[minIndex]) < 0)
            {
                minIndex = j;
            }
        }
        
        // Swap minimum element with first element of unsorted portion
        if (minIndex != i)
        {
            (array[i], array[minIndex]) = (array[minIndex], array[i]);
        }
    }
}
```

### Insertion Sort

Efficient for small or nearly sorted arrays, with O(n²) worst-case time complexity:

```csharp
public static void InsertionSort<T>(T[] array) where T : IComparable<T>
{
    for (int i = 1; i < array.Length; i++)
    {
        T key = array[i];
        int j = i - 1;
        
        // Move elements greater than key to one position ahead
        while (j >= 0 && array[j].CompareTo(key) > 0)
        {
            array[j + 1] = array[j];
            j--;
        }
        
        array[j + 1] = key;
    }
}
```

### Quick Sort

Efficient divide-and-conquer algorithm with O(n log n) average time complexity:

```csharp
public static void QuickSort<T>(T[] array) where T : IComparable<T>
{
    QuickSort(array, 0, array.Length - 1);
}

private static void QuickSort<T>(T[] array, int low, int high) where T : IComparable<T>
{
    if (low < high)
    {
        int partitionIndex = Partition(array, low, high);
        
        // Recursively sort elements before and after partition
        QuickSort(array, low, partitionIndex - 1);
        QuickSort(array, partitionIndex + 1, high);
    }
}

private static int Partition<T>(T[] array, int low, int high) where T : IComparable<T>
{
    // Choose the rightmost element as pivot
    T pivot = array[high];
    
    int i = low - 1; // Index of smaller element
    
    for (int j = low; j < high; j++)
    {
        // If current element is smaller than the pivot
        if (array[j].CompareTo(pivot) < 0)
        {
            i++;
            // Swap array[i] and array[j]
            (array[i], array[j]) = (array[j], array[i]);
        }
    }
    
    // Swap array[i+1] and array[high] (put the pivot in its correct position)
    (array[i + 1], array[high]) = (array[high], array[i + 1]);
    
    return i + 1;
}
```

### Merge Sort

Stable, divide-and-conquer algorithm with O(n log n) time complexity:

```csharp
public static void MergeSort<T>(T[] array) where T : IComparable<T>
{
    if (array.Length <= 1)
        return;
        
    T[] temp = new T[array.Length];
    MergeSort(array, temp, 0, array.Length - 1);
}

private static void MergeSort<T>(T[] array, T[] temp, int left, int right) where T : IComparable<T>
{
    if (left < right)
    {
        int middle = left + (right - left) / 2;
        
        // Sort first and second halves
        MergeSort(array, temp, left, middle);
        MergeSort(array, temp, middle + 1, right);
        
        // Merge the sorted halves
        Merge(array, temp, left, middle, right);
    }
}

private static void Merge<T>(T[] array, T[] temp, int left, int middle, int right) where T : IComparable<T>
{
    // Copy data to temp arrays
    for (int i = left; i <= right; i++)
    {
        temp[i] = array[i];
    }
    
    int i1 = left;
    int i2 = middle + 1;
    int current = left;
    
    // Merge temp arrays back into array[left..right]
    while (i1 <= middle && i2 <= right)
    {
        if (temp[i1].CompareTo(temp[i2]) <= 0)
        {
            array[current] = temp[i1];
            i1++;
        }
        else
        {
            array[current] = temp[i2];
            i2++;
        }
        current++;
    }
    
    // Copy remaining elements of left subarray if any
    while (i1 <= middle)
    {
        array[current] = temp[i1];
        i1++;
        current++;
    }
    
    // Note: Remaining elements of right subarray are already in correct position
}
```

### Heap Sort

In-place sorting algorithm with O(n log n) time complexity:

```csharp
public static void HeapSort<T>(T[] array) where T : IComparable<T>
{
    int n = array.Length;
    
    // Build heap (rearrange array)
    for (int i = n / 2 - 1; i >= 0; i--)
    {
        Heapify(array, n, i);
    }
    
    // Extract elements from heap one by one
    for (int i = n - 1; i > 0; i--)
    {
        // Move current root to end
        (array[0], array[i]) = (array[i], array[0]);
        
        // Call max heapify on the reduced heap
        Heapify(array, i, 0);
    }
}

private static void Heapify<T>(T[] array, int n, int i) where T : IComparable<T>
{
    int largest = i;       // Initialize largest as root
    int left = 2 * i + 1;  // left child
    int right = 2 * i + 2; // right child
    
    // If left child is larger than root
    if (left < n && array[left].CompareTo(array[largest]) > 0)
    {
        largest = left;
    }
    
    // If right child is larger than largest so far
    if (right < n && array[right].CompareTo(array[largest]) > 0)
    {
        largest = right;
    }
    
    // If largest is not root
    if (largest != i)
    {
        (array[i], array[largest]) = (array[largest], array[i]);
        
        // Recursively heapify the affected sub-tree
        Heapify(array, n, largest);
    }
}
```

### Comparison of Sorting Algorithms

| Algorithm      | Time Complexity (Average) | Time Complexity (Worst) | Space Complexity | Stable |
|----------------|---------------------------|-------------------------|------------------|--------|
| Bubble Sort    | O(n²)                     | O(n²)                   | O(1)             | Yes    |
| Selection Sort | O(n²)                     | O(n²)                   | O(1)             | No     |
| Insertion Sort | O(n²)                     | O(n²)                   | O(1)             | Yes    |
| Quick Sort     | O(n log n)                | O(n²)                   | O(log n)         | No     |
| Merge Sort     | O(n log n)                | O(n log n)              | O(n)             | Yes    |
| Heap Sort      | O(n log n)                | O(n log n)              | O(1)             | No     |

### Using .NET's Built-in Sorting

Instead of implementing sorting algorithms yourself, use .NET's built-in methods when possible:

```csharp
// Sort an array in-place
int[] numbers = { 5, 2, 8, 1, 9 };
Array.Sort(numbers);

// Sort a list in-place
List<string> names = new List<string> { "Charlie", "Alice", "Bob" };
names.Sort();

// Sort with custom comparer
names.Sort(StringComparer.OrdinalIgnoreCase);

// LINQ-based sorting (returns new sequence)
var sortedNames = names.OrderBy(n => n.Length).ThenBy(n => n);

// Custom sort using Comparison delegate
Array.Sort(people, (p1, p2) => p1.Age.CompareTo(p2.Age));
```

## Searching Algorithms

### Linear Search

Simple search with O(n) time complexity:

```csharp
public static int LinearSearch<T>(T[] array, T target) where T : IEquatable<T>
{
    for (int i = 0; i < array.Length; i++)
    {
        if (array[i].Equals(target))
            return i;
    }
    
    return -1; // Not found
}
```

### Binary Search

Efficient search for sorted collections with O(log n) time complexity:

```csharp
public static int BinarySearch<T>(T[] array, T target) where T : IComparable<T>
{
    int left = 0;
    int right = array.Length - 1;
    
    while (left <= right)
    {
        int mid = left + (right - left) / 2;
        int comparison = array[mid].CompareTo(target);
        
        if (comparison == 0)
            return mid;
        
        if (comparison < 0)
            left = mid + 1;
        else
            right = mid - 1;
    }
    
    return -1; // Not found
}

// Recursive version
public static int BinarySearchRecursive<T>(T[] array, T target) where T : IComparable<T>
{
    return BinarySearchRecursive(array, target, 0, array.Length - 1);
}

private static int BinarySearchRecursive<T>(T[] array, T target, int left, int right) where T : IComparable<T>
{
    if (left > right)
        return -1;
        
    int mid = left + (right - left) / 2;
    int comparison = array[mid].CompareTo(target);
    
    if (comparison == 0)
        return mid;
        
    if (comparison < 0)
        return BinarySearchRecursive(array, target, mid + 1, right);
    else
        return BinarySearchRecursive(array, target, left, mid - 1);
}
```

### Interpolation Search

Improved binary search for uniformly distributed data, with O(log log n) average time complexity:

```csharp
public static int InterpolationSearch(int[] array, int target)
{
    int low = 0;
    int high = array.Length - 1;
    
    while (low <= high && target >= array[low] && target <= array[high])
    {
        if (low == high)
        {
            if (array[low] == target)
                return low;
            return -1;
        }
        
        // Interpolation formula
        int pos = low + ((target - array[low]) * (high - low)) / (array[high] - array[low]);
        
        if (array[pos] == target)
            return pos;
            
        if (array[pos] < target)
            low = pos + 1;
        else
            high = pos - 1;
    }
    
    return -1; // Not found
}
```

### Using .NET's Built-in Search Methods

```csharp
// Array.BinarySearch
int[] sortedNumbers = { 1, 2, 3, 5, 8, 13, 21 };
int index = Array.BinarySearch(sortedNumbers, 8); // Returns 4

// List.BinarySearch
List<int> sortedList = new List<int> { 1, 2, 3, 5, 8, 13, 21 };
index = sortedList.BinarySearch(8); // Returns 4

// LINQ Contains, First, FirstOrDefault, etc.
bool contains = sortedList.Contains(8); // Returns true
int first = sortedList.First(n => n > 10); // Returns 13
int firstOrDefault = sortedList.FirstOrDefault(n => n > 100); // Returns 0 (default)

// Dictionary lookup
Dictionary<string, int> ages = new Dictionary<string, int> { { "Alice", 30 }, { "Bob", 25 } };
bool hasKey = ages.ContainsKey("Alice"); // Returns true
```

## Graph Algorithms

### Depth-First Search (DFS)

```csharp
public class Graph
{
    private int _vertices;
    private List<int>[] _adjacencyList;
    
    public Graph(int vertices)
    {
        _vertices = vertices;
        _adjacencyList = new List<int>[vertices];
        
        for (int i = 0; i < vertices; i++)
        {
            _adjacencyList[i] = new List<int>();
        }
    }
    
    public void AddEdge(int source, int destination)
    {
        _adjacencyList[source].Add(destination);
    }
    
    public void DFS(int startVertex)
    {
        bool[] visited = new bool[_vertices];
        DFSUtil(startVertex, visited);
    }
    
    private void DFSUtil(int vertex, bool[] visited)
    {
        // Mark the current node as visited and print it
        visited[vertex] = true;
        Console.Write($"{vertex} ");
        
        // Recurse for all adjacent vertices
        foreach (int adjacent in _adjacencyList[vertex])
        {
            if (!visited[adjacent])
            {
                DFSUtil(adjacent, visited);
            }
        }
    }
    
    // Iterative DFS using a stack
    public void DFSIterative(int startVertex)
    {
        bool[] visited = new bool[_vertices];
        Stack<int> stack = new Stack<int>();
        
        visited[startVertex] = true;
        stack.Push(startVertex);
        
        while (stack.Count > 0)
        {
            int vertex = stack.Pop();
            Console.Write($"{vertex} ");
            
            // Get all adjacent vertices of the popped vertex
            // If an adjacent is not visited, mark it visited and push to stack
            foreach (int adjacent in _adjacencyList[vertex])
            {
                if (!visited[adjacent])
                {
                    visited[adjacent] = true;
                    stack.Push(adjacent);
                }
            }
        }
    }
}

// Usage
var graph = new Graph(4);
graph.AddEdge(0, 1);
graph.AddEdge(0, 2);
graph.AddEdge(1, 2);
graph.AddEdge(2, 0);
graph.AddEdge(2, 3);
graph.AddEdge(3, 3);

Console.WriteLine("DFS starting from vertex 2:");
graph.DFS(2); // Output: 2 0 1 3
```

### Breadth-First Search (BFS)

```csharp
public void BFS(int startVertex)
{
    bool[] visited = new bool[_vertices];
    Queue<int> queue = new Queue<int>();
    
    visited[startVertex] = true;
    queue.Enqueue(startVertex);
    
    while (queue.Count > 0)
    {
        int vertex = queue.Dequeue();
        Console.Write($"{vertex} ");
        
        foreach (int adjacent in _adjacencyList[vertex])
        {
            if (!visited[adjacent])
            {
                visited[adjacent] = true;
                queue.Enqueue(adjacent);
            }
        }
    }
}
```

### Dijkstra's Algorithm for Shortest Path

```csharp
public class WeightedGraph
{
    private int _vertices;
    private List<Tuple<int, int>>[] _adjacencyList; // (vertex, weight)
    
    public WeightedGraph(int vertices)
    {
        _vertices = vertices;
        _adjacencyList = new List<Tuple<int, int>>[vertices];
        
        for (int i = 0; i < vertices; i++)
        {
            _adjacencyList[i] = new List<Tuple<int, int>>();
        }
    }
    
    public void AddEdge(int source, int destination, int weight)
    {
        _adjacencyList[source].Add(new Tuple<int, int>(destination, weight));
        // For undirected graph, add the reverse edge too
        // _adjacencyList[destination].Add(new Tuple<int, int>(source, weight));
    }
    
    public int[] Dijkstra(int startVertex)
    {
        int[] distance = new int[_vertices];
        bool[] processed = new bool[_vertices];
        
        // Initialize distances with maximum value
        for (int i = 0; i < _vertices; i++)
        {
            distance[i] = int.MaxValue;
        }
        
        // Priority queue to get vertex with minimum distance
        // (C# does not have a built-in min-priority queue, so we simulate it)
        // In production, you might use a proper PriorityQueue implementation
        
        // Distance to start vertex is always 0
        distance[startVertex] = 0;
        
        for (int count = 0; count < _vertices - 1; count++)
        {
            // Find the minimum distance vertex from unprocessed vertices
            int u = MinimumDistanceVertex(distance, processed);
            
            // Mark the selected vertex as processed
            processed[u] = true;
            
            // Update distances of adjacent vertices
            foreach (var edge in _adjacencyList[u])
            {
                int v = edge.Item1; // adjacent vertex
                int weight = edge.Item2; // edge weight
                
                if (!processed[v] && distance[u] != int.MaxValue && 
                    distance[u] + weight < distance[v])
                {
                    distance[v] = distance[u] + weight;
                }
            }
        }
        
        return distance;
    }
    
    private int MinimumDistanceVertex(int[] distance, bool[] processed)
    {
        int min = int.MaxValue;
        int minIndex = -1;
        
        for (int v = 0; v < _vertices; v++)
        {
            if (!processed[v] && distance[v] <= min)
            {
                min = distance[v];
                minIndex = v;
            }
        }
        
        return minIndex;
    }
}

// Usage
var graph = new WeightedGraph(5);
graph.AddEdge(0, 1, 9);
graph.AddEdge(0, 2, 6);
graph.AddEdge(0, 3, 5);
graph.AddEdge(0, 4, 3);
graph.AddEdge(2, 1, 2);
graph.AddEdge(2, 3, 4);

int[] distances = graph.Dijkstra(0);
for (int i = 0; i < distances.Length; i++)
{
    Console.WriteLine($"Distance from vertex 0 to {i} is {distances[i]}");
}
```

### A* Search Algorithm

Popular pathfinding algorithm for grid-based maps:

```csharp
public class Node
{
    public int X { get; }
    public int Y { get; }
    public int G { get; set; } // Cost from start to this node
    public int H { get; set; } // Heuristic (estimated cost to goal)
    public int F => G + H;     // Total cost
    public Node Parent { get; set; }
    
    public Node(int x, int y)
    {
        X = x;
        Y = y;
    }
}

public class AStar
{
    private readonly int[,] _grid;
    private readonly int _width;
    private readonly int _height;
    private readonly List<(int dx, int dy)> _directions = new List<(int, int)>
    {
        (0, 1), (1, 0), (0, -1), (-1, 0), // 4-directional
        // Optional: Include diagonals
        // (1, 1), (1, -1), (-1, -1), (-1, 1)
    };
    
    public AStar(int[,] grid)
    {
        _grid = grid;
        _width = grid.GetLength(0);
        _height = grid.GetLength(1);
    }
    
    public List<(int, int)> FindPath(int startX, int startY, int goalX, int goalY)
    {
        var openSet = new List<Node>();
        var closedSet = new HashSet<(int, int)>();
        
        var startNode = new Node(startX, startY);
        var goalNode = new Node(goalX, goalY);
        
        startNode.G = 0;
        startNode.H = ManhattanDistance(startNode, goalNode);
        
        openSet.Add(startNode);
        
        while (openSet.Count > 0)
        {
            // Find node with lowest F cost
            var current = openSet.OrderBy(n => n.F).First();
            
            if (current.X == goalX && current.Y == goalY)
            {
                // Goal reached, reconstruct path
                return ReconstructPath(current);
            }
            
            openSet.Remove(current);
            closedSet.Add((current.X, current.Y));
            
            foreach (var (dx, dy) in _directions)
            {
                int newX = current.X + dx;
                int newY = current.Y + dy;
                
                // Check if position is valid
                if (newX < 0 || newX >= _width || newY < 0 || newY >= _height)
                    continue;
                    
                // Check if position is walkable (not an obstacle)
                if (_grid[newX, newY] == 1) // 1 represents an obstacle
                    continue;
                    
                if (closedSet.Contains((newX, newY)))
                    continue;
                    
                int tentativeG = current.G + 1; // Cost is 1 per step
                
                var neighbor = new Node(newX, newY)
                {
                    Parent = current,
                    G = tentativeG,
                    H = ManhattanDistance(new Node(newX, newY), goalNode)
                };
                
                var existingNode = openSet.FirstOrDefault(n => n.X == newX && n.Y == newY);
                
                if (existingNode != null)
                {
                    if (tentativeG >= existingNode.G)
                        continue;
                        
                    // Path to existing node is better, update it
                    existingNode.G = tentativeG;
                    existingNode.Parent = current;
                }
                else
                {
                    openSet.Add(neighbor);
                }
            }
        }
        
        // No path found
        return new List<(int, int)>();
    }
    
    private int ManhattanDistance(Node a, Node b)
    {
        return Math.Abs(a.X - b.X) + Math.Abs(a.Y - b.Y);
    }
    
    private List<(int, int)> ReconstructPath(Node goalNode)
    {
        var path = new List<(int, int)>();
        var current = goalNode;
        
        while (current != null)
        {
            path.Add((current.X, current.Y));
            current = current.Parent;
        }
        
        path.Reverse();
        return path;
    }
}

// Usage
int[,] grid = new int[5, 5]
{
    { 0, 0, 0, 0, 0 },
    { 0, 1, 1, 0, 0 },
    { 0, 0, 0, 0, 0 },
    { 0, 1, 0, 1, 0 },
    { 0, 0, 0, 0, 0 }
};

var aStar = new AStar(grid);
var path = aStar.FindPath(0, 0, 4, 4);

foreach (var (x, y) in path)
{
    Console.WriteLine($"({x}, {y})");
}
```

## Dynamic Programming

Dynamic programming solves complex problems by breaking them into simpler subproblems and storing the results to avoid redundant calculations.

### Fibonacci Sequence

```csharp
// Recursive approach (inefficient)
public int FibonacciRecursive(int n)
{
    if (n <= 1)
        return n;
    
    return FibonacciRecursive(n - 1) + FibonacciRecursive(n - 2);
}

// Dynamic programming with memoization (top-down)
public int FibonacciMemoization(int n)
{
    if (n <= 1)
        return n;
    
    int[] memo = new int[n + 1];
    memo[0] = 0;
    memo[1] = 1;
    
    return FibonacciMemoizationHelper(n, memo);
}

private int FibonacciMemoizationHelper(int n, int[] memo)
{
    if (memo[n] != 0)
        return memo[n];
    
    memo[n] = FibonacciMemoizationHelper(n - 1, memo) + FibonacciMemoizationHelper(n - 2, memo);
    return memo[n];
}

// Dynamic programming with tabulation (bottom-up)
public int FibonacciTabulation(int n)
{
    if (n <= 1)
        return n;
    
    int[] table = new int[n + 1];
    table[0] = 0;
    table[1] = 1;
    
    for (int i = 2; i <= n; i++)
    {
        table[i] = table[i - 1] + table[i - 2];
    }
    
    return table[n];
}

// Space-optimized version
public int FibonacciOptimized(int n)
{
    if (n <= 1)
        return n;
    
    int prev = 0, current = 1, temp;
    
    for (int i = 2; i <= n; i++)
    {
        temp = current;
        current = prev + current;
        prev = temp;
    }
    
    return current;
}
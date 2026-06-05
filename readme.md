# 🐍 Python DSA Revision Sheet
### Your Weapons Arsenal Before Starting DSA

> **How to use this sheet:** Read through each section, run the code snippets mentally, then tackle the warmup problems at the end. If you can solve all warmups without hints, you're ready to start DSA.

---

## 1. Data Types & Variables — The Atoms

```python
# Core types you'll use constantly in DSA
n = 42              # int — indices, counters, values
f = 3.14            # float — rarely needed in DSA
b = True            # bool — flags, conditions
s = "hello"         # str — characters, parsing problems
none = None         # NoneType — null checks, sentinel values

# Type conversion (very common in DSA)
int("42")           # → 42
str(42)             # → "42"
ord('a')            # → 97  (char to ASCII — critical for string problems)
chr(97)             # → 'a' (ASCII to char)
```

**Key insight:** `ord('a') = 97`, `ord('A') = 65`, `ord('0') = 48` — memorise these.

---

## 2. Lists — Your Primary Workhorse

Lists are arrays in Python. 90% of DSA problems involve lists.

```python
arr = [3, 1, 4, 1, 5, 9, 2, 6]

# Access
arr[0]          # → 3  (first)
arr[-1]         # → 6  (last)
arr[1:4]        # → [1, 4, 1]  (slicing: start inclusive, end exclusive)
arr[::-1]       # → [6, 2, 9, 5, 1, 4, 1, 3]  (reverse)

# Mutation
arr.append(7)           # add to end          O(1)
arr.pop()               # remove from end      O(1)
arr.pop(0)              # remove from front    O(n) ← SLOW
arr.insert(0, 99)       # insert at index      O(n)
arr.remove(4)           # remove first 4       O(n)

# Info
len(arr)                # length
arr.index(9)            # first index of value
arr.count(1)            # count occurrences
sorted(arr)             # returns NEW sorted list  O(n log n)
arr.sort()              # sorts IN PLACE           O(n log n)
arr.sort(reverse=True)  # descending

# Initialisation patterns (you'll use these constantly)
zeros = [0] * 10                        # [0, 0, 0, ..., 0]
matrix = [[0] * 3 for _ in range(3)]   # 3×3 grid of zeros ← use list comp, NOT [[0]*3]*3
```

⚠️ **Gotcha:** `[[0]*3]*3` creates 3 references to the SAME row. Always use list comprehension for 2D arrays.

---

## 3. Strings — Know These Cold

Strings are **immutable** in Python — you cannot change them in-place.

```python
s = "Hello, World!"

# Access & slicing (same as lists)
s[0]            # 'H'
s[-1]           # '!'
s[7:12]         # 'World'
s[::-1]         # reverse string

# Methods
s.lower()               # "hello, world!"
s.upper()               # "HELLO, WORLD!"
s.strip()               # remove leading/trailing whitespace
s.split(",")            # ["Hello", " World!"]
s.split()               # splits on any whitespace
" ".join(["a","b","c"]) # "a b c"  ← critical for building strings
s.replace("l", "r")    # "Herro, Worrd!"
s.startswith("He")      # True
s.endswith("!")         # True
s.find("World")         # 7 (index), -1 if not found
s.count("l")            # 3
s.isdigit()             # False
s.isalpha()             # False
s.isalnum()             # False

# Building strings efficiently
# BAD:  result = ""  →  result += char  (creates new string each time, O(n²))
# GOOD: collect in list, then join
chars = []
for ch in s:
    chars.append(ch.upper())
result = "".join(chars)   # O(n)
```

---

## 4. Dictionaries — Hash Maps

The most powerful tool in DSA. Used for frequency counts, memoization, lookups.

```python
d = {"a": 1, "b": 2, "c": 3}

# Access
d["a"]              # 1
d.get("z")          # None  (safe, no KeyError)
d.get("z", 0)       # 0     (default value — use this for counters)

# Mutation
d["d"] = 4          # add/update
del d["a"]          # delete key
d.pop("b")          # delete and return value

# Iteration
for key in d:               # iterate keys
for val in d.values():      # iterate values
for k, v in d.items():      # iterate key-value pairs

# Existence check
"a" in d            # True   O(1) — THIS IS THE POWER
"z" not in d        # True

# Frequency counter pattern (extremely common)
from collections import Counter
freq = Counter("aabbccca")   # Counter({'a': 3, 'c': 3, 'b': 2})
freq.most_common(2)          # [('a', 3), ('c', 3)]

# defaultdict — avoids KeyError for missing keys
from collections import defaultdict
graph = defaultdict(list)    # graph[node] = [] automatically
count = defaultdict(int)     # count[key] += 1 without init
```

---

## 5. Sets — Uniqueness & Membership

O(1) average lookup. Use when you need "have I seen this before?"

```python
s = {1, 2, 3, 4, 5}

# Operations
s.add(6)            # add element
s.remove(3)         # remove (raises error if missing)
s.discard(99)       # remove (safe, no error)
3 in s              # False (after remove)  O(1)

# Set math
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}
a | b               # union:        {1,2,3,4,5,6}
a & b               # intersection: {3,4}
a - b               # difference:   {1,2}
a ^ b               # symmetric:    {1,2,5,6}

# Common pattern: deduplicate a list
unique = list(set([1, 2, 2, 3, 3, 3]))  # [1, 2, 3]
```

---

## 6. Tuples — Immutable Pairs

Used as hashable keys, coordinate pairs, and function return values.

```python
point = (3, 4)
point[0]            # 3
x, y = point        # unpacking

# Tuples as dict keys (lists can't be used as keys)
visited = set()
visited.add((0, 0))     # valid
visited.add((1, 2))     # valid
(0, 0) in visited       # True

# Common in graph problems
edges = [(0,1), (1,2), (2,3)]
```

---

## 7. Control Flow — Loops & Conditions

```python
# Conditions
if x > 0:
    pass
elif x == 0:
    pass
else:
    pass

# Ternary
result = "pos" if x > 0 else "neg"

# For loops
for i in range(5):          # 0, 1, 2, 3, 4
for i in range(2, 10, 2):   # 2, 4, 6, 8
for i in range(9, -1, -1):  # 9, 8, ..., 0  (reverse)

# Enumerate — index + value
for i, val in enumerate(arr):
    print(i, val)

# Zip — parallel iteration
for a, b in zip(list1, list2):
    print(a, b)

# While loops
while left < right:
    # two-pointer pattern
    left += 1
    right -= 1

# Loop control
break       # exit loop
continue    # skip to next iteration
```

---

## 8. Functions

```python
def solve(arr, target):
    """Docstring: what it does."""
    for i, val in enumerate(arr):
        if val == target:
            return i
    return -1           # always handle the failure case

# Default arguments
def binary_search(arr, target, lo=0, hi=None):
    if hi is None:
        hi = len(arr) - 1
    # ...

# Multiple return values
def min_max(arr):
    return min(arr), max(arr)

lo, hi = min_max([3,1,4,1,5])

# *args — variable positional args
def total(*nums):
    return sum(nums)

# Lambda — for sorting keys
pairs = [(1, 'b'), (2, 'a'), (3, 'c')]
pairs.sort(key=lambda x: x[1])   # sort by second element
```

---

## 9. Useful Built-ins — Know Every One

```python
# Math
abs(-5)             # 5
max(3, 7)           # 7
max([3, 1, 4])      # 4
min(3, 7)           # 3
sum([1,2,3])        # 6
pow(2, 10)          # 1024
2 ** 10             # 1024 (same)
round(3.7)          # 4

import math
math.sqrt(16)       # 4.0
math.floor(3.9)     # 3
math.ceil(3.1)      # 4
math.log(8, 2)      # 3.0
math.inf            # ∞  (use as initial min/max)
-math.inf           # -∞

# Sorting with keys
words = ["banana", "apple", "cherry"]
sorted(words, key=len)              # sort by length
sorted(words, key=lambda x: x[-1]) # sort by last char

# Any / All
any([False, True, False])   # True
all([True, True, True])     # True

# Enumerate, zip, map, filter
list(map(int, ["1","2","3"]))               # [1, 2, 3]
list(filter(lambda x: x > 2, [1,2,3,4]))   # [3, 4]

# Input parsing (competitive programming style)
n = int(input())
arr = list(map(int, input().split()))
```

---

## 10. Collections Module — DSA Essentials

```python
from collections import deque, Counter, defaultdict, OrderedDict
import heapq

# deque — double-ended queue, O(1) on both ends
# Use instead of list when you need popleft()
q = deque([1, 2, 3])
q.append(4)         # right add
q.appendleft(0)     # left add
q.pop()             # right remove
q.popleft()         # left remove  O(1) ← why deque > list here

# heapq — min-heap
heap = [3, 1, 4, 1, 5]
heapq.heapify(heap)             # in-place min-heap  O(n)
heapq.heappush(heap, 2)         # push              O(log n)
heapq.heappop(heap)             # pop min           O(log n)
heap[0]                         # peek min          O(1)

# Max-heap trick: negate values
heapq.heappush(heap, -val)
max_val = -heapq.heappop(heap)
```

---

## 11. Complexity — The Language of DSA

Always ask: *"What's the time and space complexity?"*

| Operation | List | Dict/Set | Sorted List |
|-----------|------|----------|-------------|
| Access by index | O(1) | — | O(1) |
| Search | O(n) | O(1) avg | O(log n) |
| Insert at end | O(1) amortized | O(1) avg | O(log n) |
| Insert at front | O(n) | — | O(log n) |
| Delete by value | O(n) | O(1) avg | O(log n) |

**Common complexities to recognise:**
- `O(1)` — hash lookup, stack push/pop, heap peek
- `O(log n)` — binary search, heap push/pop
- `O(n)` — single pass, linear scan
- `O(n log n)` — sorting
- `O(n²)` — nested loops (often a signal to optimise)
- `O(2ⁿ)` — recursive subsets (needs memoization)

---

## 12. Two Pointers & Sliding Window — Core Patterns

These two patterns solve a huge class of array/string problems.

```python
# Two Pointers — works on SORTED arrays
def two_sum_sorted(arr, target):
    left, right = 0, len(arr) - 1
    while left < right:
        s = arr[left] + arr[right]
        if s == target:
            return [left, right]
        elif s < target:
            left += 1
        else:
            right -= 1
    return []

# Sliding Window — fixed size
def max_sum_subarray(arr, k):
    window = sum(arr[:k])
    best = window
    for i in range(k, len(arr)):
        window += arr[i] - arr[i - k]   # slide: add right, drop left
        best = max(best, window)
    return best
```

---

## 13. Recursion — Think in Base Cases

```python
# Template every recursive function follows
def recurse(params):
    # 1. Base case — when to stop
    if base_condition:
        return base_value

    # 2. Recursive case — make the problem smaller
    return recurse(smaller_params)

# Factorial
def factorial(n):
    if n <= 1:          # base case
        return 1
    return n * factorial(n - 1)  # recursive case

# Fibonacci (naive — O(2ⁿ), use memoization for O(n))
def fib(n, memo={}):
    if n <= 1:
        return n
    if n in memo:
        return memo[n]
    memo[n] = fib(n-1, memo) + fib(n-2, memo)
    return memo[n]
```

---

## ⚔️ Warmup Problems

Solve these before starting DSA. They test the exact tools above.

### Level 1 — Syntax Fluency

**W1. Reverse a string** without using `[::-1]`
```python
def reverse_string(s): ...
# reverse_string("hello") → "olleh"
```

**W2. Count character frequency**
```python
def char_freq(s): ...
# char_freq("aabbbc") → {'a':2, 'b':3, 'c':1}
```

**W3. Remove duplicates from a list (preserve order)**
```python
def remove_dups(arr): ...
# remove_dups([1,2,2,3,1,4]) → [1,2,3,4]
```

---

### Level 2 — Logic & Patterns

**W4. Two Sum (unsorted)**
> Given a list and a target, return indices of two numbers that add to target.
```python
def two_sum(nums, target): ...
# two_sum([2,7,11,15], 9) → [0,1]
```

**W5. Valid Palindrome**
> Return True if a string reads the same forwards and backwards (ignore spaces, case).
```python
def is_palindrome(s): ...
# is_palindrome("A man a plan a canal Panama") → True
```

**W6. Max Subarray Sum (Kadane's Algorithm)**
> Find the contiguous subarray with the largest sum.
```python
def max_subarray(nums): ...
# max_subarray([-2,1,-3,4,-1,2,1,-5,4]) → 6
```

---

### Level 3 — Recursion & Thinking

**W7. Power function**
> Implement `x^n` using recursion.
```python
def my_pow(x, n): ...
# my_pow(2, 10) → 1024
```

**W8. Flatten a nested list**
> Given a list that may contain nested lists, return a flat list.
```python
def flatten(lst): ...
# flatten([1,[2,[3,4],5],6]) → [1,2,3,4,5,6]
```

**W9. Group anagrams**
> Given a list of words, group words that are anagrams of each other.
```python
def group_anagrams(words): ...
# group_anagrams(["eat","tea","tan","ate","nat","bat"])
# → [["eat","tea","ate"],["tan","nat"],["bat"]]
```

---

### ✅ Warmup Solutions

<details>
<summary>Click to reveal — try first!</summary>

```python
# W1
def reverse_string(s):
    result = []
    for ch in s:
        result.insert(0, ch)    # or use a stack: append then reverse
    return "".join(result)
# Better: collect forward, join reversed
def reverse_string(s):
    return "".join(reversed(s))

# W2
def char_freq(s):
    freq = {}
    for ch in s:
        freq[ch] = freq.get(ch, 0) + 1
    return freq

# W3
def remove_dups(arr):
    seen = set()
    result = []
    for x in arr:
        if x not in seen:
            seen.add(x)
            result.append(x)
    return result

# W4
def two_sum(nums, target):
    seen = {}
    for i, n in enumerate(nums):
        diff = target - n
        if diff in seen:
            return [seen[diff], i]
        seen[n] = i
    return []

# W5
def is_palindrome(s):
    cleaned = "".join(ch.lower() for ch in s if ch.isalnum())
    return cleaned == cleaned[::-1]

# W6
def max_subarray(nums):
    max_sum = curr = nums[0]
    for n in nums[1:]:
        curr = max(n, curr + n)
        max_sum = max(max_sum, curr)
    return max_sum

# W7
def my_pow(x, n):
    if n == 0: return 1
    if n < 0: return 1 / my_pow(x, -n)
    if n % 2 == 0:
        half = my_pow(x, n // 2)
        return half * half
    return x * my_pow(x, n - 1)

# W8
def flatten(lst):
    result = []
    for item in lst:
        if isinstance(item, list):
            result.extend(flatten(item))
        else:
            result.append(item)
    return result

# W9
from collections import defaultdict
def group_anagrams(words):
    groups = defaultdict(list)
    for word in words:
        key = tuple(sorted(word))   # sorted chars as hashable key
        groups[key].append(word)
    return list(groups.values())
```

</details>

---

## 🗺️ What Comes Next (DSA Roadmap)

Once warmups feel comfortable, work through topics in this order:

```
Arrays & Strings  →  Hash Maps  →  Two Pointers  →  Sliding Window
       ↓
    Recursion  →  Binary Search  →  Linked Lists  →  Stacks & Queues
       ↓
     Trees  →  Graphs (BFS/DFS)  →  Heaps  →  Dynamic Programming
```

**You're ready for DSA when:** solving W1–W6 takes under 10 minutes total, and you can explain the time complexity of each solution.

---

*Built for: Pre-DSA Python Review | Focus: Syntax + Patterns + Problem-solving instincts*

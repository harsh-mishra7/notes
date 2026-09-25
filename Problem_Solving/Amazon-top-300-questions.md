# Top 300 Amazon DSA Interview Questions

Ranked by how often each problem is reported in Amazon interviews — most frequent at the top.

**How the ranking works**
- Source: Amazon's company-wise LeetCode frequency data ([liquidslr/leetcode-company-wise-problems](https://github.com/liquidslr/leetcode-company-wise-problems), last updated Aug 2026).
- Score = 0.7 × all-time frequency + 0.3 × last-6-months frequency, so recent trends count too.
- **All-time** and **6 mo** are the dataset's frequency scores (0–100, relative to Amazon's most-asked problem). A `—` in 6 mo means it wasn't reported in the last 6 months.

**Mix:** 88 Easy · 175 Medium · 37 Hard

**Most common topics:** Array (162) · String (70) · Hash Table (66) · Two Pointers (47) · Dynamic Programming (47) · Sorting (39) · Binary Search (37) · Depth-First Search (36) · Math (34) · Breadth-First Search (33) · Tree (26) · Binary Tree (26)

**Amazon-specific tips**
- In this list, **Arrays, Strings, Hash Tables, Two Pointers and DP** dominate. Design-style problems like LRU Cache also rank near the top.
- Every round also includes **Leadership Principles** behavioural questions; prepare STAR stories alongside these problems.

---

## Trending now (last 3 months)

The problems reported most in the last 3 months — worth doing first if your interview is soon.

| # | Problem | Difficulty | Topics |
|---|---|---|---|
| 1 | [Two Sum](https://leetcode.com/problems/two-sum) | Easy | Array, Hash Table |
| 2 | [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water) | Hard | Array, Two Pointers, Dynamic Programming |
| 3 | [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) | Medium | Hash Table, String, Sliding Window |
| 4 | [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) | Easy | Array, Dynamic Programming |
| 5 | [Maximum Coins From K Consecutive Bags](https://leetcode.com/problems/maximum-coins-from-k-consecutive-bags) | Medium | Array, Binary Search, Greedy |
| 6 | [Add Two Numbers](https://leetcode.com/problems/add-two-numbers) | Medium | Linked List, Math, Recursion |
| 7 | [Group Anagrams](https://leetcode.com/problems/group-anagrams) | Medium | Array, Hash Table, String |
| 8 | [LRU Cache](https://leetcode.com/problems/lru-cache) | Medium | Hash Table, Linked List, Design |
| 9 | [Fruit Into Baskets](https://leetcode.com/problems/fruit-into-baskets) | Medium | Array, Hash Table, Sliding Window |
| 10 | [3Sum](https://leetcode.com/problems/3sum) | Medium | Array, Two Pointers, Sorting |
| 11 | [Number of Islands](https://leetcode.com/problems/number-of-islands) | Medium | Array, Depth-First Search, Breadth-First Search |
| 12 | [Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas) | Medium | Array, Binary Search |
| 13 | [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) | Medium | Two Pointers, String, Dynamic Programming |
| 14 | [Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays) | Hard | Array, Binary Search, Divide and Conquer |
| 15 | [Merge Intervals](https://leetcode.com/problems/merge-intervals) | Medium | Array, Sorting, Quicksort |
| 16 | [Palindrome Number](https://leetcode.com/problems/palindrome-number) | Easy | Math |
| 17 | [Majority Element](https://leetcode.com/problems/majority-element) | Easy | Array, Hash Table, Divide and Conquer |
| 18 | [Reorganize String](https://leetcode.com/problems/reorganize-string) | Medium | Hash Table, String, Greedy |
| 19 | [Container With Most Water](https://leetcode.com/problems/container-with-most-water) | Medium | Array, Two Pointers, Greedy |
| 20 | [Rotting Oranges](https://leetcode.com/problems/rotting-oranges) | Medium | Array, Breadth-First Search, Matrix |
| 21 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements) | Medium | Array, Hash Table, Divide and Conquer |
| 22 | [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array) | Medium | Array, Binary Search |
| 23 | [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence) | Medium | Array, Hash Table, Union-Find |
| 24 | [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring) | Hard | Hash Table, String, Sliding Window |
| 25 | [Course Schedule](https://leetcode.com/problems/course-schedule) | Medium | Depth-First Search, Breadth-First Search, Graph Theory |

---

## 1–50

| # | Problem | Difficulty | Topics | All-time | 6 mo |
|---|---|---|---|---|---|
| 1 | [Two Sum](https://leetcode.com/problems/two-sum) | Easy | Array, Hash Table | 100.0 | 100.0 |
| 2 | [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water) | Hard | Array, Two Pointers, Dynamic Programming | 88.0 | 85.7 |
| 3 | [LRU Cache](https://leetcode.com/problems/lru-cache) | Medium | Hash Table, Linked List, Design | 88.5 | 80.2 |
| 4 | [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) | Medium | Hash Table, String, Sliding Window | 84.5 | 81.6 |
| 5 | [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) | Easy | Array, Dynamic Programming | 83.8 | 81.9 |
| 6 | [Number of Islands](https://leetcode.com/problems/number-of-islands) | Medium | Array, Depth-First Search, Breadth-First Search | 85.6 | 76.0 |
| 7 | [Add Two Numbers](https://leetcode.com/problems/add-two-numbers) | Medium | Linked List, Math, Recursion | 79.7 | 79.0 |
| 8 | [Group Anagrams](https://leetcode.com/problems/group-anagrams) | Medium | Array, Hash Table, String | 80.6 | 74.9 |
| 9 | [3Sum](https://leetcode.com/problems/3sum) | Medium | Array, Two Pointers, Sorting | 78.3 | 74.9 |
| 10 | [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) | Medium | Two Pointers, String, Dynamic Programming | 78.5 | 72.9 |
| 11 | [Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays) | Hard | Array, Binary Search, Divide and Conquer | 76.8 | 74.5 |
| 12 | [Reorganize String](https://leetcode.com/problems/reorganize-string) | Medium | Hash Table, String, Greedy | 78.3 | 68.1 |
| 13 | [Merge Intervals](https://leetcode.com/problems/merge-intervals) | Medium | Array, Sorting, Quicksort | 76.6 | 72.0 |
| 14 | [Container With Most Water](https://leetcode.com/problems/container-with-most-water) | Medium | Array, Two Pointers, Greedy | 76.7 | 69.2 |
| 15 | [Course Schedule](https://leetcode.com/problems/course-schedule) | Medium | Depth-First Search, Breadth-First Search, Graph Theory | 75.0 | 72.0 |
| 16 | [Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas) | Medium | Array, Binary Search | 74.9 | 71.1 |
| 17 | [Rotting Oranges](https://leetcode.com/problems/rotting-oranges) | Medium | Array, Breadth-First Search, Matrix | 73.9 | 70.6 |
| 18 | [Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists) | Hard | Linked List, Divide and Conquer, Heap (Priority Queue) | 73.9 | 69.2 |
| 19 | [Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k) | Medium | Array, Hash Table, Prefix Sum | 72.9 | 69.2 |
| 20 | [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array) | Medium | Array, Binary Search | 71.6 | 70.2 |
| 21 | [Palindrome Number](https://leetcode.com/problems/palindrome-number) | Easy | Math | 70.8 | 70.6 |
| 22 | [Valid Parentheses](https://leetcode.com/problems/valid-parentheses) | Easy | String, Stack, Bracket Sequences | 74.4 | 61.0 |
| 23 | [Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix) | Easy | Array, String, Trie | 73.0 | 63.9 |
| 24 | [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence) | Medium | Array, Hash Table, Union-Find | 70.7 | 67.6 |
| 25 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements) | Medium | Array, Hash Table, Divide and Conquer | 71.7 | 64.5 |
| 26 | [Maximum Subarray](https://leetcode.com/problems/maximum-subarray) | Medium | Array, Divide and Conquer, Dynamic Programming | 72.0 | 63.2 |
| 27 | [Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree) | Medium | Tree, Depth-First Search, Binary Tree | 69.9 | 65.2 |
| 28 | [Majority Element](https://leetcode.com/problems/majority-element) | Easy | Array, Hash Table, Divide and Conquer | 69.6 | 65.8 |
| 29 | [Copy List with Random Pointer](https://leetcode.com/problems/copy-list-with-random-pointer) | Medium | Hash Table, Linked List | 69.8 | 62.5 |
| 30 | [Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) | Easy | Array, Two Pointers, Sorting | 70.9 | 56.7 |
| 31 | [Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number) | Medium | Hash Table, String, Backtracking | 66.9 | 64.5 |
| 32 | [House Robber](https://leetcode.com/problems/house-robber) | Medium | Array, Dynamic Programming | 67.3 | 63.2 |
| 33 | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs) | Easy | Math, Dynamic Programming, Memoization | 67.0 | 61.0 |
| 34 | [Fruit Into Baskets](https://leetcode.com/problems/fruit-into-baskets) | Medium | Array, Hash Table, Sliding Window | 63.6 | 68.6 |
| 35 | [Jump Game](https://leetcode.com/problems/jump-game) | Medium | Array, Dynamic Programming, Greedy | 67.9 | 57.6 |
| 36 | [Generate Parentheses](https://leetcode.com/problems/generate-parentheses) | Medium | String, Dynamic Programming, Backtracking | 69.1 | 54.6 |
| 37 | [Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists) | Easy | Linked List, Recursion | 66.7 | 59.4 |
| 38 | [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self) | Medium | Array, Prefix Sum | 67.5 | 56.7 |
| 39 | [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum) | Hard | Array, Queue, Sliding Window | 65.6 | 60.2 |
| 40 | [Subsets](https://leetcode.com/problems/subsets) | Medium | Array, Backtracking, Bit Manipulation | 64.0 | 63.2 |
| 41 | [Word Ladder](https://leetcode.com/problems/word-ladder) | Hard | Hash Table, String, Breadth-First Search | 68.3 | 52.3 |
| 42 | [Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array) | Easy | Array, Two Pointers | 63.6 | 61.8 |
| 43 | [Next Permutation](https://leetcode.com/problems/next-permutation) | Medium | Array, Two Pointers | 63.3 | 59.4 |
| 44 | [Rotate Image](https://leetcode.com/problems/rotate-image) | Medium | Array, Math, Matrix | 63.5 | 58.5 |
| 45 | [Course Schedule II](https://leetcode.com/problems/course-schedule-ii) | Medium | Depth-First Search, Breadth-First Search, Graph Theory | 65.8 | 52.3 |
| 46 | [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring) | Hard | Hash Table, String, Sliding Window | 61.7 | 61.8 |
| 47 | [Spiral Matrix](https://leetcode.com/problems/spiral-matrix) | Medium | Array, Matrix, Simulation | 65.3 | 51.1 |
| 48 | [Sort Colors](https://leetcode.com/problems/sort-colors) | Medium | Array, Two Pointers, Sorting | 61.4 | 60.2 |
| 49 | [Valid Anagram](https://leetcode.com/problems/valid-anagram) | Easy | Hash Table, String, Sorting | 64.3 | 52.3 |
| 50 | [Word Search](https://leetcode.com/problems/word-search) | Medium | Array, String, Backtracking | 65.3 | 49.7 |

## 51–100

| # | Problem | Difficulty | Topics | All-time | 6 mo |
|---|---|---|---|---|---|
| 51 | [Reverse Integer](https://leetcode.com/problems/reverse-integer) | Medium | Math | 62.3 | 56.7 |
| 52 | [Coin Change](https://leetcode.com/problems/coin-change) | Medium | Array, Dynamic Programming, Breadth-First Search | 63.1 | 54.6 |
| 53 | [Task Scheduler](https://leetcode.com/problems/task-scheduler) | Medium | Array, Hash Table, Greedy | 63.1 | 54.6 |
| 54 | [Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array) | Medium | Array, Binary Search | 60.8 | 58.5 |
| 55 | [Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream) | Hard | Two Pointers, Design, Sorting | 63.1 | 52.3 |
| 56 | [Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii) | Medium | Array, Two Pointers, Greedy | 61.6 | 55.7 |
| 57 | [Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement) | Medium | Hash Table, String, Sliding Window | 59.8 | 58.5 |
| 58 | [Unique Paths](https://leetcode.com/problems/unique-paths) | Medium | Math, Dynamic Programming, Combinatorics | 61.4 | 54.6 |
| 59 | [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram) | Hard | Array, Stack, Monotonic Stack | 60.4 | 56.7 |
| 60 | [Asteroid Collision](https://leetcode.com/problems/asteroid-collision) | Medium | Array, Stack, Simulation | 60.4 | 56.7 |
| 61 | [4Sum](https://leetcode.com/problems/4sum) | Medium | Array, Two Pointers, Sorting | 59.4 | 58.5 |
| 62 | [Capacity To Ship Packages Within D Days](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days) | Medium | Array, Binary Search | 59.2 | 58.5 |
| 63 | [First Missing Positive](https://leetcode.com/problems/first-missing-positive) | Hard | Array, Hash Table | 61.4 | 51.1 |
| 64 | [Move Zeroes](https://leetcode.com/problems/move-zeroes) | Easy | Array, Two Pointers | 60.0 | 53.5 |
| 65 | [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array) | Medium | Array, Divide and Conquer, Sorting | 62.1 | 46.8 |
| 66 | [Contains Duplicate](https://leetcode.com/problems/contains-duplicate) | Easy | Array, Hash Table, Sorting | 57.5 | 56.7 |
| 67 | [Roman to Integer](https://leetcode.com/problems/roman-to-integer) | Easy | Hash Table, Math, String | 61.2 | 46.8 |
| 68 | [Find Peak Element](https://leetcode.com/problems/find-peak-element) | Medium | Array, Binary Search | 60.4 | 48.3 |
| 69 | [Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum) | Hard | Dynamic Programming, Tree, Depth-First Search | 59.8 | 48.3 |
| 70 | [Single Number](https://leetcode.com/problems/single-number) | Easy | Array, Bit Manipulation | 58.6 | 51.1 |
| 71 | [Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list) | Easy | Linked List, Two Pointers, Stack | 57.2 | 53.5 |
| 72 | [Pascal's Triangle](https://leetcode.com/problems/pascals-triangle) | Easy | Array, Dynamic Programming | 60.6 | 45.1 |
| 73 | [Remove Element](https://leetcode.com/problems/remove-element) | Easy | Array, Two Pointers | 56.5 | 54.6 |
| 74 | [Permutations](https://leetcode.com/problems/permutations) | Medium | Array, Backtracking | 56.5 | 54.6 |
| 75 | [Find the Index of the First Occurrence in a String](https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string) | Easy | Two Pointers, String, String Matching | 55.0 | 57.6 |
| 76 | [N-Queens](https://leetcode.com/problems/n-queens) | Hard | Array, Backtracking, Algorithm X | 60.2 | 45.1 |
| 77 | [Word Break](https://leetcode.com/problems/word-break) | Medium | Array, Hash Table, String | 60.2 | 45.1 |
| 78 | [Search Insert Position](https://leetcode.com/problems/search-insert-position) | Easy | Array, Binary Search | 56.5 | 53.5 |
| 79 | [Single Element in a Sorted Array](https://leetcode.com/problems/single-element-in-a-sorted-array) | Medium | Array, Binary Search | 57.9 | 49.7 |
| 80 | [Zigzag Conversion](https://leetcode.com/problems/zigzag-conversion) | Medium | String | 57.0 | 51.1 |
| 81 | [Daily Temperatures](https://leetcode.com/problems/daily-temperatures) | Medium | Array, Stack, Monotonic Stack | 57.0 | 51.1 |
| 82 | [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list) | Easy | Linked List, Recursion | 58.1 | 48.3 |
| 83 | [Min Stack](https://leetcode.com/problems/min-stack) | Medium | Stack, Design | 58.6 | 46.8 |
| 84 | [Rotate Array](https://leetcode.com/problems/rotate-array) | Medium | Array, Math, Two Pointers | 59.0 | 45.1 |
| 85 | [Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1) | Medium | Array, Hash Table, Math | 60.6 | 41.2 |
| 86 | [Sqrt(x)](https://leetcode.com/problems/sqrtx) | Easy | Math, Binary Search, Newton's Method | 57.0 | 48.3 |
| 87 | [Binary Tree Right Side View](https://leetcode.com/problems/binary-tree-right-side-view) | Medium | Tree, Depth-First Search, Breadth-First Search | 57.0 | 48.3 |
| 88 | [Valid Palindrome](https://leetcode.com/problems/valid-palindrome) | Easy | Two Pointers, String | 53.6 | 55.7 |
| 89 | [Valid Sudoku](https://leetcode.com/problems/valid-sudoku) | Medium | Array, Hash Table, Matrix | 58.1 | 45.1 |
| 90 | [Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted) | Medium | Array, Two Pointers, Binary Search | 55.5 | 51.1 |
| 91 | [Max Consecutive Ones III](https://leetcode.com/problems/max-consecutive-ones-iii) | Medium | Array, Binary Search, Sliding Window | 56.0 | 49.7 |
| 92 | [Jump Game II](https://leetcode.com/problems/jump-game-ii) | Medium | Array, Dynamic Programming, Greedy | 59.4 | 41.2 |
| 93 | [Maximum Coins From K Consecutive Bags](https://leetcode.com/problems/maximum-coins-from-k-consecutive-bags) | Medium | Array, Binary Search, Greedy | 47.3 | 69.2 |
| 94 | [Next Greater Element I](https://leetcode.com/problems/next-greater-element-i) | Easy | Array, Hash Table, Stack | 54.7 | 51.1 |
| 95 | [All Nodes Distance K in Binary Tree](https://leetcode.com/problems/all-nodes-distance-k-in-binary-tree) | Medium | Hash Table, Tree, Depth-First Search | 56.5 | 46.8 |
| 96 | [Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii) | Medium | Array, Dynamic Programming, Greedy | 58.8 | 41.2 |
| 97 | [Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number) | Medium | Array, Two Pointers, Binary Search | 55.7 | 48.3 |
| 98 | [Merge Strings Alternately](https://leetcode.com/problems/merge-strings-alternately) | Easy | Two Pointers, String | 59.6 | 39.0 |
| 99 | [Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix) | Medium | Array, Binary Search, Matrix | 57.7 | 43.2 |
| 100 | [First Unique Character in a String](https://leetcode.com/problems/first-unique-character-in-a-string) | Easy | Hash Table, String, Queue | 55.2 | 48.3 |

## 101–150

| # | Problem | Difficulty | Topics | All-time | 6 mo |
|---|---|---|---|---|---|
| 101 | [Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list) | Medium | Linked List, Two Pointers | 52.7 | 53.5 |
| 102 | [Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes) | Medium | Array, Hash Table, Matrix | 55.5 | 46.8 |
| 103 | [Recyclable and Low Fat Products](https://leetcode.com/problems/recyclable-and-low-fat-products) | Easy | Database | 57.0 | 43.2 |
| 104 | [Split Array Largest Sum](https://leetcode.com/problems/split-array-largest-sum) | Hard | Array, Binary Search, Dynamic Programming | 52.4 | 53.5 |
| 105 | [String to Integer (atoi)](https://leetcode.com/problems/string-to-integer-atoi) | Medium | String | 53.8 | 49.7 |
| 106 | [String Compression](https://leetcode.com/problems/string-compression) | Medium | Two Pointers, String | 55.0 | 46.8 |
| 107 | [Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray) | Medium | Array, Dynamic Programming | 53.8 | 48.3 |
| 108 | [3Sum Closest](https://leetcode.com/problems/3sum-closest) | Medium | Array, Two Pointers, Sorting | 51.4 | 53.5 |
| 109 | [Next Greater Element II](https://leetcode.com/problems/next-greater-element-ii) | Medium | Array, Stack, Monotonic Stack | 54.1 | 46.8 |
| 110 | [Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group) | Hard | Linked List, Recursion | 56.0 | 41.2 |
| 111 | [Concatenation of Array](https://leetcode.com/problems/concatenation-of-array) | Easy | Array, Simulation | 48.5 | 58.5 |
| 112 | [Max Consecutive Ones](https://leetcode.com/problems/max-consecutive-ones) | Easy | Array | 51.4 | 51.1 |
| 113 | [Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree) | Hard | String, Tree, Depth-First Search | 56.5 | 39.0 |
| 114 | [Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree) | Easy | Tree, Depth-First Search, Binary Tree | 52.4 | 48.3 |
| 115 | [Combine Two Tables](https://leetcode.com/problems/combine-two-tables) | Easy | Database | 51.1 | 51.1 |
| 116 | [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum) | Medium | Array, Dynamic Programming, Knapsack Problem | 52.0 | 48.3 |
| 117 | [Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal) | Medium | Tree, Breadth-First Search, Binary Tree | 55.5 | 39.0 |
| 118 | [Check if Array Is Sorted and Rotated](https://leetcode.com/problems/check-if-array-is-sorted-and-rotated) | Easy | Array | 49.7 | 52.3 |
| 119 | [Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays) | Easy | Array, Hash Table, Two Pointers | 51.1 | 48.3 |
| 120 | [Reorder List](https://leetcode.com/problems/reorder-list) | Medium | Linked List, Two Pointers, Stack | 54.1 | 41.2 |
| 121 | [Analyze User Website Visit Pattern](https://leetcode.com/problems/analyze-user-website-visit-pattern) | Medium | Array, Hash Table, String | 62.6 | 20.9 |
| 122 | [Sum of Subarray Minimums](https://leetcode.com/problems/sum-of-subarray-minimums) | Medium | Array, Dynamic Programming, Stack | 50.7 | 48.3 |
| 123 | [Pow(x, n)](https://leetcode.com/problems/powx-n) | Medium | Math, Recursion | 53.6 | 41.2 |
| 124 | [Add Binary](https://leetcode.com/problems/add-binary) | Easy | Math, String, Bit Manipulation | 50.4 | 48.3 |
| 125 | [Odd Even Linked List](https://leetcode.com/problems/odd-even-linked-list) | Medium | Linked List | 53.3 | 41.2 |
| 126 | [Search a 2D Matrix II](https://leetcode.com/problems/search-a-2d-matrix-ii) | Medium | Array, Binary Search, Divide and Conquer | 50.7 | 46.8 |
| 127 | [Largest Number](https://leetcode.com/problems/largest-number) | Medium | Array, String, Greedy | 51.4 | 45.1 |
| 128 | [Edit Distance](https://leetcode.com/problems/edit-distance) | Medium | String, Dynamic Programming | 52.7 | 41.2 |
| 129 | [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence) | Medium | Array, Binary Search, Dynamic Programming | 52.7 | 41.2 |
| 130 | [Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching) | Hard | String, Dynamic Programming, Recursion | 53.6 | 39.0 |
| 131 | [Word Search II](https://leetcode.com/problems/word-search-ii) | Hard | Array, String, Backtracking | 53.6 | 39.0 |
| 132 | [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle) | Easy | Hash Table, Linked List, Two Pointers | 51.4 | 43.2 |
| 133 | [Concatenated Words](https://leetcode.com/problems/concatenated-words) | Hard | Array, String, Dynamic Programming | 53.0 | 39.0 |
| 134 | [Gas Station](https://leetcode.com/problems/gas-station) | Medium | Array, Greedy | 53.8 | 36.4 |
| 135 | [Rotate String](https://leetcode.com/problems/rotate-string) | Easy | String, String Matching | 48.1 | 49.7 |
| 136 | [Rotate List](https://leetcode.com/problems/rotate-list) | Medium | Linked List, Two Pointers | 50.0 | 45.1 |
| 137 | [Integer to Roman](https://leetcode.com/problems/integer-to-roman) | Medium | Hash Table, Math, String | 52.4 | 39.0 |
| 138 | [Decode String](https://leetcode.com/problems/decode-string) | Medium | String, Stack, Recursion | 49.7 | 45.1 |
| 139 | [Same Tree](https://leetcode.com/problems/same-tree) | Easy | Tree, Depth-First Search, Breadth-First Search | 48.1 | 48.3 |
| 140 | [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal) | Medium | Tree, Breadth-First Search, Binary Tree | 54.4 | 33.5 |
| 141 | [LFU Cache](https://leetcode.com/problems/lfu-cache) | Hard | Hash Table, Linked List, Design | 55.5 | 30.1 |
| 142 | [Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree) | Medium | Tree, Depth-First Search, Binary Search Tree | 52.4 | 36.4 |
| 143 | [Fibonacci Number](https://leetcode.com/problems/fibonacci-number) | Easy | Math, Dynamic Programming, Recursion | 51.1 | 39.0 |
| 144 | [Longest Valid Parentheses](https://leetcode.com/problems/longest-valid-parentheses) | Hard | String, Dynamic Programming, Stack | 50.0 | 41.2 |
| 145 | [Create Hello World Function](https://leetcode.com/problems/create-hello-world-function) | Easy |  | 58.6 | 20.9 |
| 146 | [Number of Provinces](https://leetcode.com/problems/number-of-provinces) | Medium | Depth-First Search, Breadth-First Search, Union-Find | 50.7 | 39.0 |
| 147 | [Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum) | Medium | Array, Binary Search, Sliding Window | 48.9 | 43.2 |
| 148 | [Missing Number](https://leetcode.com/problems/missing-number) | Easy | Array, Hash Table, Math | 54.4 | 30.1 |
| 149 | [Basic Calculator](https://leetcode.com/problems/basic-calculator) | Hard | Math, String, Stack | 51.7 | 36.4 |
| 150 | [Plus One](https://leetcode.com/problems/plus-one) | Easy | Array, Math | 50.4 | 39.0 |

## 151–200

| # | Problem | Difficulty | Topics | All-time | 6 mo |
|---|---|---|---|---|---|
| 151 | [Palindrome Partitioning](https://leetcode.com/problems/palindrome-partitioning) | Medium | String, Dynamic Programming, Backtracking | 50.4 | 39.0 |
| 152 | [Second Highest Salary](https://leetcode.com/problems/second-highest-salary) | Medium | Database | 47.7 | 45.1 |
| 153 | [Time Based Key-Value Store](https://leetcode.com/problems/time-based-key-value-store) | Medium | Hash Table, String, Binary Search | 47.7 | 45.1 |
| 154 | [Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list) | Easy | Linked List, Two Pointers | 51.4 | 36.4 |
| 155 | [Balanced Binary Tree](https://leetcode.com/problems/balanced-binary-tree) | Easy | Tree, Depth-First Search, Binary Tree | 50.0 | 39.0 |
| 156 | [Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array) | Medium | Array, Binary Search | 47.3 | 45.1 |
| 157 | [Isomorphic Strings](https://leetcode.com/problems/isomorphic-strings) | Easy | Hash Table, String | 47.3 | 45.1 |
| 158 | [Happy Number](https://leetcode.com/problems/happy-number) | Easy | Hash Table, Math, Two Pointers | 50.7 | 36.4 |
| 159 | [Squares of a Sorted Array](https://leetcode.com/problems/squares-of-a-sorted-array) | Easy | Array, Two Pointers, Sorting | 44.6 | 49.7 |
| 160 | [Permutation in String](https://leetcode.com/problems/permutation-in-string) | Medium | Hash Table, Two Pointers, String | 48.1 | 41.2 |
| 161 | [Reverse String](https://leetcode.com/problems/reverse-string) | Easy | Two Pointers, String | 45.6 | 46.8 |
| 162 | [Remove Duplicates from Sorted Array II](https://leetcode.com/problems/remove-duplicates-from-sorted-array-ii) | Medium | Array, Two Pointers | 48.9 | 39.0 |
| 163 | [Combination Sum](https://leetcode.com/problems/combination-sum) | Medium | Array, Backtracking | 52.7 | 30.1 |
| 164 | [Contains Duplicate II](https://leetcode.com/problems/contains-duplicate-ii) | Easy | Array, Hash Table, Sliding Window | 49.3 | 36.4 |
| 165 | [Candy](https://leetcode.com/problems/candy) | Hard | Array, Greedy | 53.3 | 26.0 |
| 166 | [Evaluate Division](https://leetcode.com/problems/evaluate-division) | Medium | Array, String, Depth-First Search | 50.0 | 33.5 |
| 167 | [Flood Fill](https://leetcode.com/problems/flood-fill) | Easy | Array, Depth-First Search, Breadth-First Search | 53.0 | 26.0 |
| 168 | [Vertical Order Traversal of a Binary Tree](https://leetcode.com/problems/vertical-order-traversal-of-a-binary-tree) | Hard | Hash Table, Tree, Depth-First Search | 47.3 | 39.0 |
| 169 | [Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii) | Medium | Hash Table, Linked List, Two Pointers | 43.6 | 46.8 |
| 170 | [Surrounded Regions](https://leetcode.com/problems/surrounded-regions) | Medium | Array, Depth-First Search, Breadth-First Search | 46.9 | 39.0 |
| 171 | [Count Primes](https://leetcode.com/problems/count-primes) | Medium | Array, Math, Enumeration | 46.9 | 39.0 |
| 172 | [Symmetric Tree](https://leetcode.com/problems/symmetric-tree) | Easy | Tree, Depth-First Search, Breadth-First Search | 47.3 | 36.4 |
| 173 | [Peak Index in a Mountain Array](https://leetcode.com/problems/peak-index-in-a-mountain-array) | Medium | Array, Binary Search, Ternary Search | 46.0 | 39.0 |
| 174 | [Maximum Profit in Job Scheduling](https://leetcode.com/problems/maximum-profit-in-job-scheduling) | Hard | Array, Binary Search, Dynamic Programming | 51.4 | 26.0 |
| 175 | [Random Pick with Weight](https://leetcode.com/problems/random-pick-with-weight) | Medium | Array, Math, Binary Search | 45.6 | 39.0 |
| 176 | [Binary Search](https://leetcode.com/problems/binary-search) | Easy | Array, Binary Search | 43.6 | 43.2 |
| 177 | [Word Break II](https://leetcode.com/problems/word-break-ii) | Hard | Array, Hash Table, String | 50.7 | 26.0 |
| 178 | [Maximum Average Subarray I](https://leetcode.com/problems/maximum-average-subarray-i) | Easy | Array, Sliding Window | 42.5 | 45.1 |
| 179 | [Running Sum of 1d Array](https://leetcode.com/problems/running-sum-of-1d-array) | Easy | Array, Prefix Sum | 42.5 | 45.1 |
| 180 | [Reverse Linked List II](https://leetcode.com/problems/reverse-linked-list-ii) | Medium | Linked List | 45.1 | 39.0 |
| 181 | [Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals) | Medium | Array, Dynamic Programming, Greedy | 43.1 | 43.2 |
| 182 | [Boats to Save People](https://leetcode.com/problems/boats-to-save-people) | Medium | Array, Two Pointers, Greedy | 42.0 | 45.1 |
| 183 | [House Robber III](https://leetcode.com/problems/house-robber-iii) | Medium | Dynamic Programming, Tree, Depth-First Search | 44.6 | 39.0 |
| 184 | [Reverse Pairs](https://leetcode.com/problems/reverse-pairs) | Hard | Array, Binary Search, Divide and Conquer | 44.6 | 39.0 |
| 185 | [Frequency of the Most Frequent Element](https://leetcode.com/problems/frequency-of-the-most-frequent-element) | Medium | Array, Binary Search, Greedy | 44.6 | 39.0 |
| 186 | [Minimum Operations to Make Array Non Decreasing](https://leetcode.com/problems/minimum-operations-to-make-array-non-decreasing) | Medium | Array, Greedy | 38.9 | 52.3 |
| 187 | [Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree) | Easy | Tree, Depth-First Search, Breadth-First Search | 46.9 | 33.5 |
| 188 | [Pacific Atlantic Water Flow](https://leetcode.com/problems/pacific-atlantic-water-flow) | Medium | Array, Depth-First Search, Breadth-First Search | 45.6 | 36.4 |
| 189 | [Minimum Number of Days to Make m Bouquets](https://leetcode.com/problems/minimum-number-of-days-to-make-m-bouquets) | Medium | Array, Binary Search | 42.5 | 43.2 |
| 190 | [Subsets II](https://leetcode.com/problems/subsets-ii) | Medium | Array, Backtracking, Bit Manipulation | 43.1 | 41.2 |
| 191 | [Sort List](https://leetcode.com/problems/sort-list) | Medium | Linked List, Two Pointers, Divide and Conquer | 46.0 | 33.5 |
| 192 | [Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string) | Medium | Hash Table, String, Sliding Window | 46.0 | 33.5 |
| 193 | [Subarrays with K Different Integers](https://leetcode.com/problems/subarrays-with-k-different-integers) | Hard | Array, Hash Table, Sliding Window | 46.0 | 33.5 |
| 194 | [Binary Tree Inorder Traversal](https://leetcode.com/problems/binary-tree-inorder-traversal) | Easy | Stack, Tree, Depth-First Search | 40.8 | 45.1 |
| 195 | [Maximum Width of Binary Tree](https://leetcode.com/problems/maximum-width-of-binary-tree) | Medium | Tree, Depth-First Search, Breadth-First Search | 45.6 | 33.5 |
| 196 | [Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list) | Easy | Linked List | 43.1 | 39.0 |
| 197 | [Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation) | Medium | Array, Math, Stack | 46.9 | 30.1 |
| 198 | [Sudoku Solver](https://leetcode.com/problems/sudoku-solver) | Hard | Array, Hash Table, Backtracking | 48.5 | 26.0 |
| 199 | [Integer to English Words](https://leetcode.com/problems/integer-to-english-words) | Hard | Math, String, Recursion | 48.5 | 26.0 |
| 200 | [Maximal Rectangle](https://leetcode.com/problems/maximal-rectangle) | Hard | Array, Dynamic Programming, Stack | 45.1 | 33.5 |

## 201–250

| # | Problem | Difficulty | Topics | All-time | 6 mo |
|---|---|---|---|---|---|
| 201 | [Find K Closest Elements](https://leetcode.com/problems/find-k-closest-elements) | Medium | Array, Two Pointers, Binary Search | 45.1 | 33.5 |
| 202 | [Search Suggestions System](https://leetcode.com/problems/search-suggestions-system) | Medium | Array, String, Binary Search | 46.5 | 30.1 |
| 203 | [Length of Last Word](https://leetcode.com/problems/length-of-last-word) | Easy | String | 43.6 | 36.4 |
| 204 | [Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists) | Easy | Hash Table, Linked List, Two Pointers | 46.0 | 30.1 |
| 205 | [Identify the Largest Outlier in an Array](https://leetcode.com/problems/identify-the-largest-outlier-in-an-array) | Medium | Array, Hash Table, Counting | 58.8 | — |
| 206 | [Sum of Total Strength of Wizards](https://leetcode.com/problems/sum-of-total-strength-of-wizards) | Hard | Array, Stack, Monotonic Stack | 40.2 | 43.2 |
| 207 | [Remove K Digits](https://leetcode.com/problems/remove-k-digits) | Medium | String, Stack, Greedy | 42.0 | 39.0 |
| 208 | [Divide Two Integers](https://leetcode.com/problems/divide-two-integers) | Medium | Math, Bit Manipulation | 44.1 | 33.5 |
| 209 | [Cheapest Flights Within K Stops](https://leetcode.com/problems/cheapest-flights-within-k-stops) | Medium | Dynamic Programming, Depth-First Search, Breadth-First Search | 44.1 | 33.5 |
| 210 | [Kth Missing Positive Number](https://leetcode.com/problems/kth-missing-positive-number) | Easy | Array, Binary Search | 40.8 | 41.2 |
| 211 | [Managers with at Least 5 Direct Reports](https://leetcode.com/problems/managers-with-at-least-5-direct-reports) | Medium | Database | 42.5 | 36.4 |
| 212 | [Valid Palindrome II](https://leetcode.com/problems/valid-palindrome-ii) | Easy | Two Pointers, String, Greedy | 42.5 | 36.4 |
| 213 | [Construct Binary Tree from Preorder and Inorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal) | Medium | Array, Hash Table, Divide and Conquer | 45.1 | 30.1 |
| 214 | [Lowest Common Ancestor of a Binary Search Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree) | Medium | Tree, Depth-First Search, Binary Search Tree | 45.1 | 30.1 |
| 215 | [Top K Frequent Words](https://leetcode.com/problems/top-k-frequent-words) | Medium | Array, Hash Table, String | 43.6 | 33.5 |
| 216 | [Ransom Note](https://leetcode.com/problems/ransom-note) | Easy | Hash Table, String, Counting | 40.2 | 41.2 |
| 217 | [Power of Two](https://leetcode.com/problems/power-of-two) | Easy | Math, Bit Manipulation, Recursion | 42.0 | 36.4 |
| 218 | [Accounts Merge](https://leetcode.com/problems/accounts-merge) | Medium | Array, Hash Table, String | 42.0 | 36.4 |
| 219 | [Unique Paths II](https://leetcode.com/problems/unique-paths-ii) | Medium | Array, Dynamic Programming, Matrix | 40.8 | 39.0 |
| 220 | [Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum) | Medium | Array, Dynamic Programming, Matrix | 44.6 | 30.1 |
| 221 | [Employees Earning More Than Their Managers](https://leetcode.com/problems/employees-earning-more-than-their-managers) | Easy | Database | 44.6 | 30.1 |
| 222 | [Rearrange Array Elements by Sign](https://leetcode.com/problems/rearrange-array-elements-by-sign) | Medium | Array, Two Pointers, Simulation | 44.6 | 30.1 |
| 223 | [Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string) | Medium | Two Pointers, String | 51.1 | 14.3 |
| 224 | [Open the Lock](https://leetcode.com/problems/open-the-lock) | Medium | Array, Hash Table, String | 41.4 | 36.4 |
| 225 | [Assign Cookies](https://leetcode.com/problems/assign-cookies) | Easy | Array, Two Pointers, Greedy | 40.2 | 39.0 |
| 226 | [Majority Element II](https://leetcode.com/problems/majority-element-ii) | Medium | Array, Hash Table, Sorting | 42.5 | 33.5 |
| 227 | [First Bad Version](https://leetcode.com/problems/first-bad-version) | Easy | Binary Search, Interactive | 38.9 | 41.2 |
| 228 | [Search in Rotated Sorted Array II](https://leetcode.com/problems/search-in-rotated-sorted-array-ii) | Medium | Array, Binary Search | 40.8 | 36.4 |
| 229 | [K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin) | Medium | Array, Math, Divide and Conquer | 47.3 | 20.9 |
| 230 | [Populating Next Right Pointers in Each Node](https://leetcode.com/problems/populating-next-right-pointers-in-each-node) | Medium | Linked List, Tree, Depth-First Search | 45.1 | 26.0 |
| 231 | [Basic Calculator II](https://leetcode.com/problems/basic-calculator-ii) | Medium | Math, String, Stack | 50.0 | 14.3 |
| 232 | [Rising Temperature](https://leetcode.com/problems/rising-temperature) | Easy | Database | 41.4 | 33.5 |
| 233 | [Is Subsequence](https://leetcode.com/problems/is-subsequence) | Easy | Two Pointers, String, Dynamic Programming | 41.4 | 33.5 |
| 234 | [Online Stock Span](https://leetcode.com/problems/online-stock-span) | Medium | Stack, Design, Monotonic Stack | 44.6 | 26.0 |
| 235 | [Wildcard Matching](https://leetcode.com/problems/wildcard-matching) | Hard | String, Dynamic Programming, Greedy | 38.9 | 39.0 |
| 236 | [Article Views I](https://leetcode.com/problems/article-views-i) | Easy | Database | 46.5 | 20.9 |
| 237 | [Target Sum](https://leetcode.com/problems/target-sum) | Medium | Array, Dynamic Programming, Backtracking | 44.1 | 26.0 |
| 238 | [Contiguous Array](https://leetcode.com/problems/contiguous-array) | Medium | Array, Hash Table, Prefix Sum | 44.1 | 26.0 |
| 239 | [Interleaving String](https://leetcode.com/problems/interleaving-string) | Medium | String, Dynamic Programming | 36.7 | 43.2 |
| 240 | [Maximum Frequency After Subarray Operation](https://leetcode.com/problems/maximum-frequency-after-subarray-operation) | Medium | Array, Hash Table, Dynamic Programming | 48.9 | 14.3 |
| 241 | [01 Matrix](https://leetcode.com/problems/01-matrix) | Medium | Array, Dynamic Programming, Breadth-First Search | 40.2 | 33.5 |
| 242 | [Recover Binary Search Tree](https://leetcode.com/problems/recover-binary-search-tree) | Medium | Tree, Depth-First Search, Binary Search Tree | 38.9 | 36.4 |
| 243 | [Reverse Vowels of a String](https://leetcode.com/problems/reverse-vowels-of-a-string) | Easy | Two Pointers, String | 38.9 | 36.4 |
| 244 | [Set Mismatch](https://leetcode.com/problems/set-mismatch) | Easy | Array, Hash Table, Bit Manipulation | 41.4 | 30.1 |
| 245 | [Add Two Integers](https://leetcode.com/problems/add-two-integers) | Easy | Math | 41.4 | 30.1 |
| 246 | [Min Cost Climbing Stairs](https://leetcode.com/problems/min-cost-climbing-stairs) | Easy | Array, Dynamic Programming | 43.1 | 26.0 |
| 247 | [Construct Binary Search Tree from Preorder Traversal](https://leetcode.com/problems/construct-binary-search-tree-from-preorder-traversal) | Medium | Array, Stack, Tree | 37.4 | 39.0 |
| 248 | [Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst) | Medium | Tree, Depth-First Search, Binary Search Tree | 45.1 | 20.9 |
| 249 | [Plates Between Candles](https://leetcode.com/problems/plates-between-candles) | Medium | Array, String, Binary Search | 47.7 | 14.3 |
| 250 | [Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words) | Hard | Hash Table, String, Sliding Window | 38.2 | 36.4 |

## 251–300

| # | Problem | Difficulty | Topics | All-time | 6 mo |
|---|---|---|---|---|---|
| 251 | [Partition List](https://leetcode.com/problems/partition-list) | Medium | Linked List, Two Pointers | 38.2 | 36.4 |
| 252 | [Valid Parenthesis String](https://leetcode.com/problems/valid-parenthesis-string) | Medium | String, Dynamic Programming, Stack | 38.2 | 36.4 |
| 253 | [Amount of Time for Binary Tree to Be Infected](https://leetcode.com/problems/amount-of-time-for-binary-tree-to-be-infected) | Medium | Hash Table, Tree, Depth-First Search | 38.2 | 36.4 |
| 254 | [Count and Say](https://leetcode.com/problems/count-and-say) | Medium | String | 40.8 | 30.1 |
| 255 | [House Robber II](https://leetcode.com/problems/house-robber-ii) | Medium | Array, Dynamic Programming | 40.8 | 30.1 |
| 256 | [Maximum Points You Can Obtain from Cards](https://leetcode.com/problems/maximum-points-you-can-obtain-from-cards) | Medium | Array, Sliding Window, Prefix Sum | 40.8 | 30.1 |
| 257 | [Average Time of Process per Machine](https://leetcode.com/problems/average-time-of-process-per-machine) | Easy | Database | 42.5 | 26.0 |
| 258 | [Lowest Common Ancestor of a Binary Tree III](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree-iii) | Medium | Hash Table, Two Pointers, Tree | 44.6 | 20.9 |
| 259 | [Add Digits](https://leetcode.com/problems/add-digits) | Easy | Math, Simulation, Number Theory | 36.7 | 39.0 |
| 260 | [Word Ladder II](https://leetcode.com/problems/word-ladder-ii) | Hard | Hash Table, String, Backtracking | 44.1 | 20.9 |
| 261 | [Insert Interval](https://leetcode.com/problems/insert-interval) | Medium | Array | 37.4 | 36.4 |
| 262 | [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence) | Medium | String, Dynamic Programming, Longest Common Subsequence | 37.4 | 36.4 |
| 263 | [Find Missing and Repeated Values](https://leetcode.com/problems/find-missing-and-repeated-values) | Easy | Array, Hash Table, Math | 37.4 | 36.4 |
| 264 | [Make Array Zero by Subtracting Equal Amounts](https://leetcode.com/problems/make-array-zero-by-subtracting-equal-amounts) | Easy | Array, Hash Table, Greedy | 46.5 | 14.3 |
| 265 | [Design Circular Queue](https://leetcode.com/problems/design-circular-queue) | Medium | Array, Linked List, Design | 38.2 | 33.5 |
| 266 | [Big Countries](https://leetcode.com/problems/big-countries) | Easy | Database | 39.5 | 30.1 |
| 267 | [Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs) | Medium | Linked List, Recursion | 40.8 | 26.0 |
| 268 | [Longest Continuous Subarray With Absolute Diff Less Than or Equal to Limit](https://leetcode.com/problems/longest-continuous-subarray-with-absolute-diff-less-than-or-equal-to-limit) | Medium | Array, Queue, Sliding Window | 37.4 | 33.5 |
| 269 | [Burst Balloons](https://leetcode.com/problems/burst-balloons) | Hard | Array, Dynamic Programming | 35.9 | 36.4 |
| 270 | [First Unique Number](https://leetcode.com/problems/first-unique-number) | Medium | Array, Hash Table, Design | 35.9 | 36.4 |
| 271 | [Max Area of Island](https://leetcode.com/problems/max-area-of-island) | Medium | Array, Depth-First Search, Breadth-First Search | 40.2 | 26.0 |
| 272 | [Critical Connections in a Network](https://leetcode.com/problems/critical-connections-in-a-network) | Hard | Depth-First Search, Graph Theory, Biconnected Component | 40.2 | 26.0 |
| 273 | [Coin Change II](https://leetcode.com/problems/coin-change-ii) | Medium | Array, Dynamic Programming, Knapsack Problem | 38.2 | 30.1 |
| 274 | [Predict the Winner](https://leetcode.com/problems/predict-the-winner) | Medium | Array, Math, Dynamic Programming | 36.7 | 33.5 |
| 275 | [Fizz Buzz](https://leetcode.com/problems/fizz-buzz) | Easy | Math, String, Simulation | 42.0 | 20.9 |
| 276 | [Clone Graph](https://leetcode.com/problems/clone-graph) | Medium | Hash Table, Depth-First Search, Breadth-First Search | 39.5 | 26.0 |
| 277 | [Monthly Transactions I](https://leetcode.com/problems/monthly-transactions-i) | Medium | Database | 39.5 | 26.0 |
| 278 | [Angle Between Hands of a Clock](https://leetcode.com/problems/angle-between-hands-of-a-clock) | Medium | Math | 31.3 | 45.1 |
| 279 | [Sort Characters By Frequency](https://leetcode.com/problems/sort-characters-by-frequency) | Medium | Hash Table, String, Sorting | 41.4 | 20.9 |
| 280 | [Can Place Flowers](https://leetcode.com/problems/can-place-flowers) | Easy | Array, Greedy | 37.4 | 30.1 |
| 281 | [Simplify Path](https://leetcode.com/problems/simplify-path) | Medium | String, Stack | 44.1 | 14.3 |
| 282 | [Convert Sorted Array to Binary Search Tree](https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree) | Easy | Array, Divide and Conquer, Tree | 38.9 | 26.0 |
| 283 | [Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks) | Easy | Stack, Design, Queue | 38.9 | 26.0 |
| 284 | [Design HashMap](https://leetcode.com/problems/design-hashmap) | Easy | Array, Hash Table, Linked List | 38.9 | 26.0 |
| 285 | [Multiply Strings](https://leetcode.com/problems/multiply-strings) | Medium | Math, String, Simulation | 33.3 | 39.0 |
| 286 | [Students and Examinations](https://leetcode.com/problems/students-and-examinations) | Easy | Database | 34.2 | 36.4 |
| 287 | [Furthest Building You Can Reach](https://leetcode.com/problems/furthest-building-you-can-reach) | Medium | Array, Greedy, Heap (Priority Queue) | 40.8 | 20.9 |
| 288 | [The kth Factor of n](https://leetcode.com/problems/the-kth-factor-of-n) | Medium | Math, Number Theory, Prime Factorization | 43.6 | 14.3 |
| 289 | [Range Sum Query - Immutable](https://leetcode.com/problems/range-sum-query-immutable) | Easy | Array, Design, Prefix Sum | 36.7 | 30.1 |
| 290 | [Bus Routes](https://leetcode.com/problems/bus-routes) | Hard | Array, Hash Table, Breadth-First Search | 36.7 | 30.1 |
| 291 | [Alien Dictionary](https://leetcode.com/problems/alien-dictionary) | Hard | Array, String, Depth-First Search | 38.2 | 26.0 |
| 292 | [Subarray Product Less Than K](https://leetcode.com/problems/subarray-product-less-than-k) | Medium | Array, Binary Search, Sliding Window | 38.2 | 26.0 |
| 293 | [Product Sales Analysis III](https://leetcode.com/problems/product-sales-analysis-iii) | Medium | Database | 38.2 | 26.0 |
| 294 | [Shortest Path in Binary Matrix](https://leetcode.com/problems/shortest-path-in-binary-matrix) | Medium | Array, Breadth-First Search, Matrix | 38.2 | 26.0 |
| 295 | [Longest Arithmetic Sequence After Changing At Most One Element](https://leetcode.com/problems/longest-arithmetic-sequence-after-changing-at-most-one-element) | Medium | Array, Enumeration | 32.3 | 39.0 |
| 296 | [Duplicate Emails](https://leetcode.com/problems/duplicate-emails) | Easy | Database | 33.3 | 36.4 |
| 297 | [Count Binary Substrings](https://leetcode.com/problems/count-binary-substrings) | Easy | Two Pointers, String | 35.9 | 30.1 |
| 298 | [Find Pivot Index](https://leetcode.com/problems/find-pivot-index) | Easy | Array, Prefix Sum | 35.9 | 30.1 |
| 299 | [Find the Smallest Divisor Given a Threshold](https://leetcode.com/problems/find-the-smallest-divisor-given-a-threshold) | Medium | Array, Binary Search | 35.1 | 30.1 |
| 300 | [Customer Who Visited but Did Not Make Any Transactions](https://leetcode.com/problems/customer-who-visited-but-did-not-make-any-transactions) | Easy | Database | 35.1 | 30.1 |

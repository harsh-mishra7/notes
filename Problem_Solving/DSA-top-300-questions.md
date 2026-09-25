# Top 300 DSA Interview Questions (Big Tech / Product Companies)

Ranked by how often each problem is actually reported in interviews at 30 big product companies — most frequent at the top.

**How the ranking works**
- Source: company-wise LeetCode frequency data ([liquidslr/leetcode-company-wise-problems](https://github.com/liquidslr/leetcode-company-wise-problems), last updated Aug 2026).
- Companies: **Google, Amazon, Microsoft, Meta, Apple** (weighted 1.5×) + Bloomberg, Adobe, Uber, Oracle, LinkedIn, Salesforce, Goldman Sachs, Walmart Labs, Atlassian, Flipkart, Netflix, Nvidia, Intuit, PayPal, TikTok, ByteDance, Airbnb, ServiceNow, Cisco, Visa, Qualcomm, Samsung, DE Shaw, J.P. Morgan, Morgan Stanley.
- Score = Σ over companies of (0.7 × all-time frequency + 0.3 × last-6-months frequency), so recent trends count too.
- **Cos** = how many of the 30 companies ask it. **Big 5**: G = Google, A = Amazon, MS = Microsoft, M = Meta, Ap = Apple.

**Mix:** 81 Easy · 172 Medium · 47 Hard

**Most common topics:** Array (161) · String (78) · Hash Table (68) · Dynamic Programming (48) · Two Pointers (43) · Sorting (40) · Depth-First Search (38) · Breadth-First Search (38) · Binary Search (34) · Math (33) · Stack (28) · Matrix (27)

**Suggested approach:** do 1–100 first (that's the core almost everyone asks), then 101–200, then 201–300.

---

## 1–50

| # | Problem | Difficulty | Topics | Cos | Big 5 |
|---|---|---|---|---|---|
| 1 | [LRU Cache](https://leetcode.com/problems/lru-cache) | Medium | Hash Table, Linked List, Design | 28 | G A MS M Ap |
| 2 | [Two Sum](https://leetcode.com/problems/two-sum) | Easy | Array, Hash Table | 28 | G A MS M Ap |
| 3 | [Number of Islands](https://leetcode.com/problems/number-of-islands) | Medium | Array, Depth-First Search, Breadth-First Search | 25 | G A MS M Ap |
| 4 | [Merge Intervals](https://leetcode.com/problems/merge-intervals) | Medium | Array, Sorting, Quicksort | 26 | G A MS M Ap |
| 5 | [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters) | Medium | Hash Table, String, Sliding Window | 25 | G A MS M Ap |
| 6 | [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) | Easy | Array, Dynamic Programming | 23 | G A MS M Ap |
| 7 | [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water) | Hard | Array, Two Pointers, Dynamic Programming | 23 | G A MS M Ap |
| 8 | [Valid Parentheses](https://leetcode.com/problems/valid-parentheses) | Easy | String, Stack, Bracket Sequences | 26 | G A MS M Ap |
| 9 | [Group Anagrams](https://leetcode.com/problems/group-anagrams) | Medium | Array, Hash Table, String | 20 | G A MS M Ap |
| 10 | [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring) | Medium | Two Pointers, String, Dynamic Programming | 23 | G A MS M Ap |
| 11 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements) | Medium | Array, Hash Table, Divide and Conquer | 22 | G A MS M Ap |
| 12 | [3Sum](https://leetcode.com/problems/3sum) | Medium | Array, Two Pointers, Sorting | 20 | G A MS M Ap |
| 13 | [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array) | Medium | Array, Binary Search | 21 | G A MS M Ap |
| 14 | [Maximum Subarray](https://leetcode.com/problems/maximum-subarray) | Medium | Array, Divide and Conquer, Dynamic Programming | 22 | G A MS M Ap |
| 15 | [Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k) | Medium | Array, Hash Table, Prefix Sum | 21 | G A MS M Ap |
| 16 | [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array) | Medium | Array, Divide and Conquer, Sorting | 23 | G A MS M Ap |
| 17 | [Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays) | Hard | Array, Binary Search, Divide and Conquer | 20 | G A MS M Ap |
| 18 | [Container With Most Water](https://leetcode.com/problems/container-with-most-water) | Medium | Array, Two Pointers, Greedy | 21 | G A MS M Ap |
| 19 | [Rotting Oranges](https://leetcode.com/problems/rotting-oranges) | Medium | Array, Breadth-First Search, Matrix | 23 | G A MS M Ap |
| 20 | [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence) | Medium | Array, Hash Table, Union-Find | 21 | G A MS M Ap |
| 21 | [Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists) | Hard | Linked List, Divide and Conquer, Heap (Priority Queue) | 19 | G A MS M Ap |
| 22 | [Spiral Matrix](https://leetcode.com/problems/spiral-matrix) | Medium | Array, Matrix, Simulation | 20 | G A MS M Ap |
| 23 | [Course Schedule](https://leetcode.com/problems/course-schedule) | Medium | Depth-First Search, Breadth-First Search, Graph Theory | 20 | G A MS M Ap |
| 24 | [Generate Parentheses](https://leetcode.com/problems/generate-parentheses) | Medium | String, Dynamic Programming, Backtracking | 19 | G A MS M Ap |
| 25 | [Course Schedule II](https://leetcode.com/problems/course-schedule-ii) | Medium | Depth-First Search, Breadth-First Search, Graph Theory | 18 | G A MS M Ap |
| 26 | [Move Zeroes](https://leetcode.com/problems/move-zeroes) | Easy | Array, Two Pointers | 19 | G A MS M Ap |
| 27 | [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list) | Easy | Linked List, Recursion | 18 | G A MS M Ap |
| 28 | [Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii) | Medium | Array, Two Pointers, Greedy | 18 | G A MS M Ap |
| 29 | [Coin Change](https://leetcode.com/problems/coin-change) | Medium | Array, Dynamic Programming, Breadth-First Search | 19 | G A MS M Ap |
| 30 | [House Robber](https://leetcode.com/problems/house-robber) | Medium | Array, Dynamic Programming | 20 | G A MS M Ap |
| 31 | [Word Search](https://leetcode.com/problems/word-search) | Medium | Array, String, Backtracking | 19 | G A MS M Ap |
| 32 | [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum) | Hard | Array, Queue, Sliding Window | 19 | G A MS M Ap |
| 33 | [Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas) | Medium | Array, Binary Search | 19 | G A MS M Ap |
| 34 | [Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream) | Hard | Two Pointers, Design, Sorting | 18 | G A MS M Ap |
| 35 | [Sort Colors](https://leetcode.com/problems/sort-colors) | Medium | Array, Two Pointers, Sorting | 20 | G A MS M Ap |
| 36 | [Rotate Image](https://leetcode.com/problems/rotate-image) | Medium | Array, Math, Matrix | 18 | G A MS M Ap |
| 37 | [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self) | Medium | Array, Prefix Sum | 18 | G A MS M Ap |
| 38 | [Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix) | Easy | Array, String, Trie | 14 | G A MS M Ap |
| 39 | [Add Two Numbers](https://leetcode.com/problems/add-two-numbers) | Medium | Linked List, Math, Recursion | 15 | G A MS M Ap |
| 40 | [Word Ladder](https://leetcode.com/problems/word-ladder) | Hard | Hash Table, String, Breadth-First Search | 19 | G A MS M Ap |
| 41 | [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence) | Medium | Array, Binary Search, Dynamic Programming | 18 | G A MS M |
| 42 | [Find Peak Element](https://leetcode.com/problems/find-peak-element) | Medium | Array, Binary Search | 18 | G A MS M Ap |
| 43 | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs) | Easy | Math, Dynamic Programming, Memoization | 16 | G A MS M Ap |
| 44 | [Jump Game](https://leetcode.com/problems/jump-game) | Medium | Array, Dynamic Programming, Greedy | 18 | G A MS M Ap |
| 45 | [Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number) | Medium | Hash Table, String, Backtracking | 16 | G A MS M Ap |
| 46 | [Next Permutation](https://leetcode.com/problems/next-permutation) | Medium | Array, Two Pointers | 16 | G A MS M |
| 47 | [Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array) | Easy | Array, Two Pointers, Sorting | 16 | G A MS M Ap |
| 48 | [Valid Palindrome](https://leetcode.com/problems/valid-palindrome) | Easy | Two Pointers, String | 16 | G A MS M Ap |
| 49 | [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring) | Hard | Hash Table, String, Sliding Window | 16 | G A MS M Ap |
| 50 | [Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists) | Easy | Linked List, Recursion | 16 | G A MS M Ap |

## 51–100

| # | Problem | Difficulty | Topics | Cos | Big 5 |
|---|---|---|---|---|---|
| 51 | [Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1) | Medium | Array, Hash Table, Math | 14 | G A MS M Ap |
| 52 | [Majority Element](https://leetcode.com/problems/majority-element) | Easy | Array, Hash Table, Divide and Conquer | 13 | G A MS M |
| 53 | [Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array) | Medium | Array, Binary Search | 14 | G A MS M Ap |
| 54 | [Asteroid Collision](https://leetcode.com/problems/asteroid-collision) | Medium | Array, Stack, Simulation | 16 | G A MS M |
| 55 | [Valid Anagram](https://leetcode.com/problems/valid-anagram) | Easy | Hash Table, String, Sorting | 15 | G A MS M Ap |
| 56 | [Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree) | Medium | Tree, Depth-First Search, Binary Tree | 14 | G A MS M Ap |
| 57 | [Text Justification](https://leetcode.com/problems/text-justification) | Hard | Array, String, Simulation | 16 | G A MS M Ap |
| 58 | [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram) | Hard | Array, Stack, Monotonic Stack | 17 | G A MS M Ap |
| 59 | [Decode String](https://leetcode.com/problems/decode-string) | Medium | String, Stack, Recursion | 15 | G A MS M Ap |
| 60 | [Word Break](https://leetcode.com/problems/word-break) | Medium | Array, Hash Table, String | 14 | G A MS M Ap |
| 61 | [Combination Sum](https://leetcode.com/problems/combination-sum) | Medium | Array, Backtracking | 16 | G A MS M Ap |
| 62 | [Pow(x, n)](https://leetcode.com/problems/powx-n) | Medium | Math, Recursion | 15 | G A MS M |
| 63 | [Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii) | Medium | Array, Dynamic Programming, Greedy | 14 | G A MS M Ap |
| 64 | [Roman to Integer](https://leetcode.com/problems/roman-to-integer) | Easy | Hash Table, Math, String | 13 | G A MS M Ap |
| 65 | [Copy List with Random Pointer](https://leetcode.com/problems/copy-list-with-random-pointer) | Medium | Hash Table, Linked List | 13 | G A MS M |
| 66 | [Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group) | Hard | Linked List, Recursion | 15 | G A MS M Ap |
| 67 | [Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array) | Easy | Array, Two Pointers | 12 | G A MS M Ap |
| 68 | [Min Stack](https://leetcode.com/problems/min-stack) | Medium | Stack, Design | 14 | G A MS M Ap |
| 69 | [Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray) | Medium | Array, Dynamic Programming | 14 | G A MS M |
| 70 | [LFU Cache](https://leetcode.com/problems/lfu-cache) | Hard | Hash Table, Linked List, Design | 17 | G A MS M Ap |
| 71 | [Gas Station](https://leetcode.com/problems/gas-station) | Medium | Array, Greedy | 14 | G A MS M Ap |
| 72 | [String Compression](https://leetcode.com/problems/string-compression) | Medium | Two Pointers, String | 12 | G A MS M Ap |
| 73 | [Daily Temperatures](https://leetcode.com/problems/daily-temperatures) | Medium | Array, Stack, Monotonic Stack | 14 | G A MS M |
| 74 | [Rotate Array](https://leetcode.com/problems/rotate-array) | Medium | Array, Math, Two Pointers | 11 | G A MS M Ap |
| 75 | [Palindrome Number](https://leetcode.com/problems/palindrome-number) | Easy | Math | 9 | G A MS M |
| 76 | [Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum) | Hard | Dynamic Programming, Tree, Depth-First Search | 14 | G A MS M Ap |
| 77 | [Edit Distance](https://leetcode.com/problems/edit-distance) | Medium | String, Dynamic Programming | 14 | G A MS M Ap |
| 78 | [Permutations](https://leetcode.com/problems/permutations) | Medium | Array, Backtracking | 14 | G A MS M Ap |
| 79 | [Reverse Integer](https://leetcode.com/problems/reverse-integer) | Medium | Math | 11 | G A MS M Ap |
| 80 | [First Missing Positive](https://leetcode.com/problems/first-missing-positive) | Hard | Array, Hash Table | 15 | G A MS M Ap |
| 81 | [Subsets](https://leetcode.com/problems/subsets) | Medium | Array, Backtracking, Bit Manipulation | 13 | G A MS M |
| 82 | [Basic Calculator II](https://leetcode.com/problems/basic-calculator-ii) | Medium | Math, String, Stack | 13 | G A MS M Ap |
| 83 | [String to Integer (atoi)](https://leetcode.com/problems/string-to-integer-atoi) | Medium | String | 12 | G A MS M Ap |
| 84 | [Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree) | Hard | String, Tree, Depth-First Search | 14 | G A MS M Ap |
| 85 | [Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement) | Medium | Hash Table, String, Sliding Window | 12 | G A MS M Ap |
| 86 | [First Unique Character in a String](https://leetcode.com/problems/first-unique-character-in-a-string) | Easy | Hash Table, String, Queue | 13 | G A MS M Ap |
| 87 | [Time Based Key-Value Store](https://leetcode.com/problems/time-based-key-value-store) | Medium | Hash Table, String, Binary Search | 13 | G A MS M Ap |
| 88 | [Zigzag Conversion](https://leetcode.com/problems/zigzag-conversion) | Medium | String | 10 | G A MS M Ap |
| 89 | [Unique Paths](https://leetcode.com/problems/unique-paths) | Medium | Math, Dynamic Programming, Combinatorics | 13 | G A MS M |
| 90 | [Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix) | Medium | Array, Binary Search, Matrix | 14 | G A MS M Ap |
| 91 | [Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree) | Easy | Tree, Depth-First Search, Binary Tree | 13 | G A MS M Ap |
| 92 | [Binary Tree Right Side View](https://leetcode.com/problems/binary-tree-right-side-view) | Medium | Tree, Depth-First Search, Breadth-First Search | 12 | G A MS M Ap |
| 93 | [Jump Game II](https://leetcode.com/problems/jump-game-ii) | Medium | Array, Dynamic Programming, Greedy | 14 | G A MS M Ap |
| 94 | [Maximal Square](https://leetcode.com/problems/maximal-square) | Medium | Array, Dynamic Programming, Matrix | 16 | G A MS M Ap |
| 95 | [Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list) | Medium | Linked List, Two Pointers | 12 | G A MS M Ap |
| 96 | [Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string) | Medium | Two Pointers, String | 13 | G A MS M Ap |
| 97 | [Happy Number](https://leetcode.com/problems/happy-number) | Easy | Hash Table, Math, Two Pointers | 13 | G A MS M Ap |
| 98 | [Max Consecutive Ones III](https://leetcode.com/problems/max-consecutive-ones-iii) | Medium | Array, Binary Search, Sliding Window | 11 | G A MS M |
| 99 | [Valid Sudoku](https://leetcode.com/problems/valid-sudoku) | Medium | Array, Hash Table, Matrix | 12 | G A MS M Ap |
| 100 | [Sqrt(x)](https://leetcode.com/problems/sqrtx) | Easy | Math, Binary Search, Newton's Method | 12 | G A MS M Ap |

## 101–150

| # | Problem | Difficulty | Topics | Cos | Big 5 |
|---|---|---|---|---|---|
| 101 | [Longest Valid Parentheses](https://leetcode.com/problems/longest-valid-parentheses) | Hard | String, Dynamic Programming, Stack | 13 | G A MS M |
| 102 | [Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes) | Medium | Array, Hash Table, Matrix | 11 | G A MS M Ap |
| 103 | [Palindromic Substrings](https://leetcode.com/problems/palindromic-substrings) | Medium | Two Pointers, String, Dynamic Programming | 14 | G A MS M Ap |
| 104 | [Contains Duplicate](https://leetcode.com/problems/contains-duplicate) | Easy | Array, Hash Table, Sorting | 11 | G A MS M Ap |
| 105 | [Basic Calculator](https://leetcode.com/problems/basic-calculator) | Hard | Math, String, Stack | 13 | G A MS M Ap |
| 106 | [Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching) | Hard | String, Dynamic Programming, Recursion | 11 | G A MS M Ap |
| 107 | [Next Greater Element II](https://leetcode.com/problems/next-greater-element-ii) | Medium | Array, Stack, Monotonic Stack | 14 | G A MS M Ap |
| 108 | [Capacity To Ship Packages Within D Days](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days) | Medium | Array, Binary Search | 13 | G A MS M Ap |
| 109 | [Simplify Path](https://leetcode.com/problems/simplify-path) | Medium | String, Stack | 12 | G A MS M Ap |
| 110 | [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence) | Medium | String, Dynamic Programming, Longest Common Subsequence | 14 | G A MS M Ap |
| 111 | [Candy](https://leetcode.com/problems/candy) | Hard | Array, Greedy | 15 | G A MS M |
| 112 | [Single Element in a Sorted Array](https://leetcode.com/problems/single-element-in-a-sorted-array) | Medium | Array, Binary Search | 11 | G A MS M |
| 113 | [Task Scheduler](https://leetcode.com/problems/task-scheduler) | Medium | Array, Hash Table, Greedy | 13 | G A MS M Ap |
| 114 | [Cheapest Flights Within K Stops](https://leetcode.com/problems/cheapest-flights-within-k-stops) | Medium | Dynamic Programming, Depth-First Search, Breadth-First Search | 13 | G A MS M Ap |
| 115 | [Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number) | Medium | Array, Two Pointers, Binary Search | 13 | G A MS M |
| 116 | [Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal) | Medium | Tree, Breadth-First Search, Binary Tree | 13 | G A MS M Ap |
| 117 | [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle) | Easy | Hash Table, Linked List, Two Pointers | 11 | G A MS M |
| 118 | [All Nodes Distance K in Binary Tree](https://leetcode.com/problems/all-nodes-distance-k-in-binary-tree) | Medium | Hash Table, Tree, Depth-First Search | 13 | G A MS M Ap |
| 119 | [Single Number](https://leetcode.com/problems/single-number) | Easy | Array, Bit Manipulation | 10 | G A MS M |
| 120 | [Random Pick with Weight](https://leetcode.com/problems/random-pick-with-weight) | Medium | Array, Math, Binary Search | 11 | G A MS M Ap |
| 121 | [N-Queens](https://leetcode.com/problems/n-queens) | Hard | Array, Backtracking, Algorithm X | 9 | G A MS M |
| 122 | [Contains Duplicate II](https://leetcode.com/problems/contains-duplicate-ii) | Easy | Array, Hash Table, Sliding Window | 10 | G A MS M Ap |
| 123 | [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal) | Medium | Tree, Breadth-First Search, Binary Tree | 11 | G A MS M Ap |
| 124 | [4Sum](https://leetcode.com/problems/4sum) | Medium | Array, Two Pointers, Sorting | 9 | G A MS M Ap |
| 125 | [Integer to Roman](https://leetcode.com/problems/integer-to-roman) | Medium | Hash Table, Math, String | 12 | G A MS M |
| 126 | [Split Array Largest Sum](https://leetcode.com/problems/split-array-largest-sum) | Hard | Array, Binary Search, Dynamic Programming | 11 | G A MS M |
| 127 | [Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted) | Medium | Array, Two Pointers, Binary Search | 10 | G A MS M Ap |
| 128 | [Permutation in String](https://leetcode.com/problems/permutation-in-string) | Medium | Hash Table, Two Pointers, String | 12 | G A MS M Ap |
| 129 | [Word Search II](https://leetcode.com/problems/word-search-ii) | Hard | Array, String, Backtracking | 12 | G A MS M Ap |
| 130 | [Vertical Order Traversal of a Binary Tree](https://leetcode.com/problems/vertical-order-traversal-of-a-binary-tree) | Hard | Hash Table, Tree, Depth-First Search | 10 | G A MS M |
| 131 | [Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree) | Easy | Tree, Depth-First Search, Breadth-First Search | 10 | G A MS M Ap |
| 132 | [Find the Index of the First Occurrence in a String](https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string) | Easy | Two Pointers, String, String Matching | 8 | G A MS M Ap |
| 133 | [Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array) | Medium | Array, Binary Search | 13 | G A MS M Ap |
| 134 | [Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum) | Medium | Array, Binary Search, Sliding Window | 11 | G A MS M Ap |
| 135 | [Fruit Into Baskets](https://leetcode.com/problems/fruit-into-baskets) | Medium | Array, Hash Table, Sliding Window | 10 | G A MS M |
| 136 | [Subarray Product Less Than K](https://leetcode.com/problems/subarray-product-less-than-k) | Medium | Array, Binary Search, Sliding Window | 12 | G A MS M Ap |
| 137 | [Largest Number](https://leetcode.com/problems/largest-number) | Medium | Array, String, Greedy | 11 | G A MS M |
| 138 | [Search a 2D Matrix II](https://leetcode.com/problems/search-a-2d-matrix-ii) | Medium | Array, Binary Search, Divide and Conquer | 11 | G A MS M Ap |
| 139 | [Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list) | Easy | Linked List, Two Pointers, Stack | 9 | G A MS M |
| 140 | [Reverse String](https://leetcode.com/problems/reverse-string) | Easy | Two Pointers, String | 9 | G A MS M Ap |
| 141 | [Reorder List](https://leetcode.com/problems/reorder-list) | Medium | Linked List, Two Pointers, Stack | 10 | G A MS M Ap |
| 142 | [Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list) | Easy | Linked List, Two Pointers | 9 | G A MS M Ap |
| 143 | [Decode Ways](https://leetcode.com/problems/decode-ways) | Medium | String, Dynamic Programming | 13 | G A MS M |
| 144 | [Find K Closest Elements](https://leetcode.com/problems/find-k-closest-elements) | Medium | Array, Two Pointers, Binary Search | 14 | G A MS M Ap |
| 145 | [Pascal's Triangle](https://leetcode.com/problems/pascals-triangle) | Easy | Array, Dynamic Programming | 7 | G A MS M Ap |
| 146 | [Next Greater Element I](https://leetcode.com/problems/next-greater-element-i) | Easy | Array, Hash Table, Stack | 11 | G A MS M Ap |
| 147 | [Accounts Merge](https://leetcode.com/problems/accounts-merge) | Medium | Array, Hash Table, String | 11 | G A MS M |
| 148 | [Restore IP Addresses](https://leetcode.com/problems/restore-ip-addresses) | Medium | String, Backtracking | 12 | G A MS M |
| 149 | [Fibonacci Number](https://leetcode.com/problems/fibonacci-number) | Easy | Math, Dynamic Programming, Recursion | 8 | G A MS M Ap |
| 150 | [Alien Dictionary](https://leetcode.com/problems/alien-dictionary) | Hard | Array, String, Depth-First Search | 10 | G A MS M Ap |

## 151–200

| # | Problem | Difficulty | Topics | Cos | Big 5 |
|---|---|---|---|---|---|
| 151 | [Plus One](https://leetcode.com/problems/plus-one) | Easy | Array, Math | 8 | G A MS M |
| 152 | [Remove Element](https://leetcode.com/problems/remove-element) | Easy | Array, Two Pointers | 8 | G A MS M Ap |
| 153 | [Can Place Flowers](https://leetcode.com/problems/can-place-flowers) | Easy | Array, Greedy | 11 | G A MS M Ap |
| 154 | [Contiguous Array](https://leetcode.com/problems/contiguous-array) | Medium | Array, Hash Table, Prefix Sum | 10 | G A MS M |
| 155 | [Merge Strings Alternately](https://leetcode.com/problems/merge-strings-alternately) | Easy | Two Pointers, String | 7 | G A MS M |
| 156 | [Reorganize String](https://leetcode.com/problems/reorganize-string) | Medium | Hash Table, String, Greedy | 10 | G A MS M |
| 157 | [Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists) | Easy | Hash Table, Linked List, Two Pointers | 11 | G A MS M Ap |
| 158 | [Insert Interval](https://leetcode.com/problems/insert-interval) | Medium | Array | 12 | G A MS M Ap |
| 159 | [Fizz Buzz](https://leetcode.com/problems/fizz-buzz) | Easy | Math, String, Simulation | 11 | G A MS M Ap |
| 160 | [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum) | Medium | Array, Dynamic Programming, Knapsack Problem | 10 | G A MS M Ap |
| 161 | [Palindrome Partitioning](https://leetcode.com/problems/palindrome-partitioning) | Medium | String, Dynamic Programming, Backtracking | 8 | G A MS M |
| 162 | [Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree) | Medium | Hash Table, String, Design | 10 | G A MS M Ap |
| 163 | [Word Break II](https://leetcode.com/problems/word-break-ii) | Hard | Array, Hash Table, String | 10 | G A MS M |
| 164 | [Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree) | Medium | Tree, Depth-First Search, Binary Search Tree | 10 | G A MS M Ap |
| 165 | [Remove All Adjacent Duplicates in String II](https://leetcode.com/problems/remove-all-adjacent-duplicates-in-string-ii) | Medium | String, Stack | 12 | G A MS M |
| 166 | [Max Area of Island](https://leetcode.com/problems/max-area-of-island) | Medium | Array, Depth-First Search, Breadth-First Search | 12 | G A MS M Ap |
| 167 | [Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst) | Medium | Tree, Depth-First Search, Binary Search Tree | 10 | G A MS M |
| 168 | [Reverse Linked List II](https://leetcode.com/problems/reverse-linked-list-ii) | Medium | Linked List | 11 | G A MS M Ap |
| 169 | [Valid Palindrome II](https://leetcode.com/problems/valid-palindrome-ii) | Easy | Two Pointers, String, Greedy | 9 | G A MS M Ap |
| 170 | [Missing Number](https://leetcode.com/problems/missing-number) | Easy | Array, Hash Table, Math | 9 | G A MS M Ap |
| 171 | [Maximum Number of Events That Can Be Attended](https://leetcode.com/problems/maximum-number-of-events-that-can-be-attended) | Medium | Array, Greedy, Sorting | 12 | G A MS M |
| 172 | [Design Hit Counter](https://leetcode.com/problems/design-hit-counter) | Medium | Array, Binary Search, Design | 10 | G A MS M Ap |
| 173 | [Top K Frequent Words](https://leetcode.com/problems/top-k-frequent-words) | Medium | Array, Hash Table, String | 13 | G A MS M Ap |
| 174 | [Maximal Rectangle](https://leetcode.com/problems/maximal-rectangle) | Hard | Array, Dynamic Programming, Stack | 10 | G A MS M |
| 175 | [Reconstruct Itinerary](https://leetcode.com/problems/reconstruct-itinerary) | Hard | Array, String, Depth-First Search | 13 | G A MS M Ap |
| 176 | [Integer to English Words](https://leetcode.com/problems/integer-to-english-words) | Hard | Math, String, Recursion | 12 | G A MS M Ap |
| 177 | [3Sum Closest](https://leetcode.com/problems/3sum-closest) | Medium | Array, Two Pointers, Sorting | 7 | G A MS M |
| 178 | [Making A Large Island](https://leetcode.com/problems/making-a-large-island) | Hard | Array, Depth-First Search, Breadth-First Search | 9 | G A MS M |
| 179 | [Squares of a Sorted Array](https://leetcode.com/problems/squares-of-a-sorted-array) | Easy | Array, Two Pointers, Sorting | 7 | G A MS M |
| 180 | [Minimum Absolute Difference](https://leetcode.com/problems/minimum-absolute-difference) | Easy | Array, Sorting | 10 | G A MS M |
| 181 | [All O`one Data Structure](https://leetcode.com/problems/all-oone-data-structure) | Hard | Hash Table, Linked List, Design | 11 | G A MS M Ap |
| 182 | [Valid Parenthesis String](https://leetcode.com/problems/valid-parenthesis-string) | Medium | String, Dynamic Programming, Stack | 10 | G A MS M Ap |
| 183 | [Add Binary](https://leetcode.com/problems/add-binary) | Easy | Math, String, Bit Manipulation | 7 | G A MS M |
| 184 | [Rotate List](https://leetcode.com/problems/rotate-list) | Medium | Linked List, Two Pointers | 8 | G A MS M |
| 185 | [Evaluate Division](https://leetcode.com/problems/evaluate-division) | Medium | Array, String, Depth-First Search | 10 | G A MS M Ap |
| 186 | [Maximum Product of Three Numbers](https://leetcode.com/problems/maximum-product-of-three-numbers) | Easy | Array, Math, Sorting | 10 | G A MS M Ap |
| 187 | [Remove K Digits](https://leetcode.com/problems/remove-k-digits) | Medium | String, Stack, Greedy | 10 | G A MS M |
| 188 | [Diagonal Traverse](https://leetcode.com/problems/diagonal-traverse) | Medium | Array, Matrix, Simulation | 9 | G A MS M |
| 189 | [Exclusive Time of Functions](https://leetcode.com/problems/exclusive-time-of-functions) | Medium | Array, Stack | 10 | G A MS M Ap |
| 190 | [Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs) | Medium | Linked List, Recursion | 10 | G A MS M |
| 191 | [Symmetric Tree](https://leetcode.com/problems/symmetric-tree) | Easy | Tree, Depth-First Search, Breadth-First Search | 8 | G A MS M Ap |
| 192 | [House Robber II](https://leetcode.com/problems/house-robber-ii) | Medium | Array, Dynamic Programming | 11 | G A MS M |
| 193 | [Combination Sum II](https://leetcode.com/problems/combination-sum-ii) | Medium | Array, Backtracking | 12 | G A MS M |
| 194 | [Shortest Path in Binary Matrix](https://leetcode.com/problems/shortest-path-in-binary-matrix) | Medium | Array, Breadth-First Search, Matrix | 10 | G A MS M Ap |
| 195 | [Design Circular Queue](https://leetcode.com/problems/design-circular-queue) | Medium | Array, Linked List, Design | 10 | G A MS M Ap |
| 196 | [Power of Two](https://leetcode.com/problems/power-of-two) | Easy | Math, Bit Manipulation, Recursion | 8 | G A MS M |
| 197 | [Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum) | Medium | Array, Dynamic Programming, Matrix | 9 | G A MS M |
| 198 | [Sum of Subarray Minimums](https://leetcode.com/problems/sum-of-subarray-minimums) | Medium | Array, Dynamic Programming, Stack | 9 | G A MS M Ap |
| 199 | [Kth Missing Positive Number](https://leetcode.com/problems/kth-missing-positive-number) | Easy | Array, Binary Search | 8 | G A MS M |
| 200 | [Add Strings](https://leetcode.com/problems/add-strings) | Easy | Math, String, Simulation | 10 | G A MS M |

## 201–250

| # | Problem | Difficulty | Topics | Cos | Big 5 |
|---|---|---|---|---|---|
| 201 | [Recyclable and Low Fat Products](https://leetcode.com/problems/recyclable-and-low-fat-products) | Easy | Database | 5 | G A MS M |
| 202 | [Sudoku Solver](https://leetcode.com/problems/sudoku-solver) | Hard | Array, Hash Table, Backtracking | 10 | G A MS M Ap |
| 203 | [Maximum Profit in Job Scheduling](https://leetcode.com/problems/maximum-profit-in-job-scheduling) | Hard | Array, Binary Search, Dynamic Programming | 10 | G A MS M |
| 204 | [Clone Graph](https://leetcode.com/problems/clone-graph) | Medium | Hash Table, Depth-First Search, Breadth-First Search | 10 | G A MS M Ap |
| 205 | [Create Hello World Function](https://leetcode.com/problems/create-hello-world-function) | Easy |  | 5 | G A MS M |
| 206 | [Binary Search](https://leetcode.com/problems/binary-search) | Easy | Array, Binary Search | 7 | G A MS M Ap |
| 207 | [Search Insert Position](https://leetcode.com/problems/search-insert-position) | Easy | Array, Binary Search | 5 | G A MS M |
| 208 | [Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals) | Medium | Array, Dynamic Programming, Greedy | 12 | G A MS M Ap |
| 209 | [Burst Balloons](https://leetcode.com/problems/burst-balloons) | Hard | Array, Dynamic Programming | 10 | G A MS M |
| 210 | [Minimum Remove to Make Valid Parentheses](https://leetcode.com/problems/minimum-remove-to-make-valid-parentheses) | Medium | String, Stack | 9 | G A MS M Ap |
| 211 | [Reverse Bits](https://leetcode.com/problems/reverse-bits) | Easy | Divide and Conquer, Bit Manipulation | 9 | G A MS M Ap |
| 212 | [Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays) | Easy | Array, Hash Table, Two Pointers | 6 | G A MS M Ap |
| 213 | [Wildcard Matching](https://leetcode.com/problems/wildcard-matching) | Hard | String, Dynamic Programming, Greedy | 9 | G A MS M |
| 214 | [Isomorphic Strings](https://leetcode.com/problems/isomorphic-strings) | Easy | Hash Table, String | 7 | G A MS M |
| 215 | [Flatten Nested List Iterator](https://leetcode.com/problems/flatten-nested-list-iterator) | Medium | Stack, Tree, Depth-First Search | 11 | G A MS M Ap |
| 216 | [Delete Node in a Linked List](https://leetcode.com/problems/delete-node-in-a-linked-list) | Medium | Linked List | 8 | G A MS M Ap |
| 217 | [Flood Fill](https://leetcode.com/problems/flood-fill) | Easy | Array, Depth-First Search, Breadth-First Search | 9 | G A MS M Ap |
| 218 | [Surrounded Regions](https://leetcode.com/problems/surrounded-regions) | Medium | Array, Depth-First Search, Breadth-First Search | 9 | G A MS M |
| 219 | [Design HashMap](https://leetcode.com/problems/design-hashmap) | Easy | Array, Hash Table, Linked List | 9 | G A MS M Ap |
| 220 | [Same Tree](https://leetcode.com/problems/same-tree) | Easy | Tree, Depth-First Search, Breadth-First Search | 7 | G A MS M Ap |
| 221 | [Pacific Atlantic Water Flow](https://leetcode.com/problems/pacific-atlantic-water-flow) | Medium | Array, Depth-First Search, Breadth-First Search | 10 | G A MS M Ap |
| 222 | [Construct Binary Tree from Preorder and Inorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal) | Medium | Array, Hash Table, Divide and Conquer | 8 | G A MS M |
| 223 | [Bus Routes](https://leetcode.com/problems/bus-routes) | Hard | Array, Hash Table, Breadth-First Search | 10 | G A MS M |
| 224 | [Count Primes](https://leetcode.com/problems/count-primes) | Medium | Array, Math, Enumeration | 9 | G A MS M Ap |
| 225 | [Concatenation of Array](https://leetcode.com/problems/concatenation-of-array) | Easy | Array, Simulation | 5 | G A MS M |
| 226 | [Best Time to Buy and Sell Stock III](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii) | Hard | Array, Dynamic Programming | 10 | G A MS M Ap |
| 227 | [Reverse Vowels of a String](https://leetcode.com/problems/reverse-vowels-of-a-string) | Easy | Two Pointers, String | 9 | G A MS M Ap |
| 228 | [Divide Two Integers](https://leetcode.com/problems/divide-two-integers) | Medium | Math, Bit Manipulation | 7 | G A MS M |
| 229 | [Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks) | Easy | Stack, Design, Queue | 8 | G A MS M Ap |
| 230 | [Longest Increasing Path in a Matrix](https://leetcode.com/problems/longest-increasing-path-in-a-matrix) | Hard | Array, Dynamic Programming, Depth-First Search | 10 | G A MS M Ap |
| 231 | [Number of Provinces](https://leetcode.com/problems/number-of-provinces) | Medium | Depth-First Search, Breadth-First Search, Union-Find | 7 | G A MS M |
| 232 | [Subarrays with K Different Integers](https://leetcode.com/problems/subarrays-with-k-different-integers) | Hard | Array, Hash Table, Sliding Window | 10 | G A MS M |
| 233 | [Combine Two Tables](https://leetcode.com/problems/combine-two-tables) | Easy | Database | 5 | G A MS M |
| 234 | [Lowest Common Ancestor of a Binary Tree III](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree-iii) | Medium | Hash Table, Two Pointers, Tree | 8 | G A MS M Ap |
| 235 | [Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string) | Medium | Hash Table, String, Sliding Window | 8 | G A MS M Ap |
| 236 | [Balanced Binary Tree](https://leetcode.com/problems/balanced-binary-tree) | Easy | Tree, Depth-First Search, Binary Tree | 7 | G A MS M Ap |
| 237 | [Peak Index in a Mountain Array](https://leetcode.com/problems/peak-index-in-a-mountain-array) | Medium | Array, Binary Search, Ternary Search | 7 | G A MS M |
| 238 | [Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation) | Medium | Array, Math, Stack | 9 | G A MS M Ap |
| 239 | [Is Subsequence](https://leetcode.com/problems/is-subsequence) | Easy | Two Pointers, String, Dynamic Programming | 8 | G A MS M |
| 240 | [Open the Lock](https://leetcode.com/problems/open-the-lock) | Medium | Array, Hash Table, String | 11 | G A MS M |
| 241 | [Second Highest Salary](https://leetcode.com/problems/second-highest-salary) | Medium | Database | 7 | G A MS M Ap |
| 242 | [Max Consecutive Ones](https://leetcode.com/problems/max-consecutive-ones) | Easy | Array | 5 | G A MS M |
| 243 | [K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin) | Medium | Array, Math, Divide and Conquer | 10 | G A MS M Ap |
| 244 | [Sort List](https://leetcode.com/problems/sort-list) | Medium | Linked List, Two Pointers, Divide and Conquer | 8 | G A MS M |
| 245 | [Binary Tree Vertical Order Traversal](https://leetcode.com/problems/binary-tree-vertical-order-traversal) | Medium | Hash Table, Tree, Depth-First Search | 7 | G A MS M Ap |
| 246 | [Maximum Average Subarray I](https://leetcode.com/problems/maximum-average-subarray-i) | Easy | Array, Sliding Window | 7 | G A MS M |
| 247 | [Subsets II](https://leetcode.com/problems/subsets-ii) | Medium | Array, Backtracking, Bit Manipulation | 6 | G A MS M |
| 248 | [Rising Temperature](https://leetcode.com/problems/rising-temperature) | Easy | Database | 7 | G A MS M |
| 249 | [Logger Rate Limiter](https://leetcode.com/problems/logger-rate-limiter) | Easy | Hash Table, Design, Data Stream | 9 | G A MS M Ap |
| 250 | [Is Graph Bipartite?](https://leetcode.com/problems/is-graph-bipartite) | Medium | Depth-First Search, Breadth-First Search, Union-Find | 10 | G A MS M Ap |

## 251–300

| # | Problem | Difficulty | Topics | Cos | Big 5 |
|---|---|---|---|---|---|
| 251 | [Check if Array Is Sorted and Rotated](https://leetcode.com/problems/check-if-array-is-sorted-and-rotated) | Easy | Array | 6 | G A MS M |
| 252 | [Sort Characters By Frequency](https://leetcode.com/problems/sort-characters-by-frequency) | Medium | Hash Table, String, Sorting | 8 | G A MS M |
| 253 | [Number of Visible People in a Queue](https://leetcode.com/problems/number-of-visible-people-in-a-queue) | Hard | Array, Stack, Monotonic Stack | 11 | G A MS M |
| 254 | [Delete Node in a BST](https://leetcode.com/problems/delete-node-in-a-bst) | Medium | Tree, Binary Search Tree, Binary Tree | 10 | G A MS M |
| 255 | [Count Binary Substrings](https://leetcode.com/problems/count-binary-substrings) | Easy | Two Pointers, String | 10 | G A MS M |
| 256 | [Length of Last Word](https://leetcode.com/problems/length-of-last-word) | Easy | String | 6 | G A MS M |
| 257 | [Reverse Pairs](https://leetcode.com/problems/reverse-pairs) | Hard | Array, Binary Search, Divide and Conquer | 7 | G A MS M Ap |
| 258 | [Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list) | Easy | Linked List | 7 | G A MS M |
| 259 | [Find K Pairs with Smallest Sums](https://leetcode.com/problems/find-k-pairs-with-smallest-sums) | Medium | Array, Heap (Priority Queue) | 10 | G A MS M |
| 260 | [Ransom Note](https://leetcode.com/problems/ransom-note) | Easy | Hash Table, String, Counting | 6 | G A MS M Ap |
| 261 | [Add Digits](https://leetcode.com/problems/add-digits) | Easy | Math, Simulation, Number Theory | 8 | G A MS M |
| 262 | [Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence) | Medium | String, Dynamic Programming | 8 | G A MS M |
| 263 | [Subarray Sums Divisible by K](https://leetcode.com/problems/subarray-sums-divisible-by-k) | Medium | Array, Hash Table, Prefix Sum | 8 | G A MS M |
| 264 | [Remove All Adjacent Duplicates In String](https://leetcode.com/problems/remove-all-adjacent-duplicates-in-string) | Easy | String, Stack | 8 | G A MS M Ap |
| 265 | [Multiply Strings](https://leetcode.com/problems/multiply-strings) | Medium | Math, String, Simulation | 8 | G A MS M |
| 266 | [Odd Even Linked List](https://leetcode.com/problems/odd-even-linked-list) | Medium | Linked List | 6 | G A MS M Ap |
| 267 | [Shortest Bridge](https://leetcode.com/problems/shortest-bridge) | Medium | Array, Depth-First Search, Breadth-First Search | 11 | G A MS M |
| 268 | [Russian Doll Envelopes](https://leetcode.com/problems/russian-doll-envelopes) | Hard | Array, Binary Search, Dynamic Programming | 9 | G A MS M |
| 269 | [Delete and Earn](https://leetcode.com/problems/delete-and-earn) | Medium | Array, Hash Table, Dynamic Programming | 9 | G A MS M |
| 270 | [Convert Sorted Array to Binary Search Tree](https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree) | Easy | Array, Divide and Conquer, Tree | 8 | G A MS M Ap |
| 271 | [Frequency of the Most Frequent Element](https://leetcode.com/problems/frequency-of-the-most-frequent-element) | Medium | Array, Binary Search, Greedy | 7 | G A MS M |
| 272 | [Article Views I](https://leetcode.com/problems/article-views-i) | Easy | Database | 6 | G A MS M |
| 273 | [Special Binary String](https://leetcode.com/problems/special-binary-string) | Hard | String, Divide and Conquer, Sorting | 7 | G A MS |
| 274 | [Rotate String](https://leetcode.com/problems/rotate-string) | Easy | String, String Matching | 6 | G A MS M |
| 275 | [Find the Smallest Divisor Given a Threshold](https://leetcode.com/problems/find-the-smallest-divisor-given-a-threshold) | Medium | Array, Binary Search | 9 | G A MS M |
| 276 | [Find the Length of the Longest Common Prefix](https://leetcode.com/problems/find-the-length-of-the-longest-common-prefix) | Medium | Array, Hash Table, String | 9 | G A MS M |
| 277 | [Search in Rotated Sorted Array II](https://leetcode.com/problems/search-in-rotated-sorted-array-ii) | Medium | Array, Binary Search | 8 | G A MS M |
| 278 | [Add Two Integers](https://leetcode.com/problems/add-two-integers) | Easy | Math | 6 | G A MS M |
| 279 | [Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words) | Hard | Hash Table, String, Sliding Window | 7 | G A MS M Ap |
| 280 | [Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii) | Medium | Hash Table, Linked List, Two Pointers | 7 | G A MS M |
| 281 | [Maximum Points You Can Obtain from Cards](https://leetcode.com/problems/maximum-points-you-can-obtain-from-cards) | Medium | Array, Sliding Window, Prefix Sum | 8 | G A MS M |
| 282 | [Shortest Path in a Grid with Obstacles Elimination](https://leetcode.com/problems/shortest-path-in-a-grid-with-obstacles-elimination) | Hard | Array, Breadth-First Search, Matrix | 9 | G A M |
| 283 | [Lowest Common Ancestor of a Binary Search Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree) | Medium | Tree, Depth-First Search, Binary Search Tree | 6 | G A MS M |
| 284 | [Meeting Rooms III](https://leetcode.com/problems/meeting-rooms-iii) | Hard | Array, Hash Table, Sorting | 10 | G A MS M |
| 285 | [Triangle](https://leetcode.com/problems/triangle) | Medium | Array, Dynamic Programming | 8 | G A MS M |
| 286 | [Word Ladder II](https://leetcode.com/problems/word-ladder-ii) | Hard | Hash Table, String, Backtracking | 9 | G A MS M Ap |
| 287 | [Remove Duplicate Letters](https://leetcode.com/problems/remove-duplicate-letters) | Medium | String, Stack, Greedy | 8 | G A MS M |
| 288 | [Design Memory Allocator](https://leetcode.com/problems/design-memory-allocator) | Medium | Array, Hash Table, Design | 10 | G A MS M Ap |
| 289 | [Trapping Rain Water II](https://leetcode.com/problems/trapping-rain-water-ii) | Hard | Array, Breadth-First Search, Heap (Priority Queue) | 9 | G A MS M |
| 290 | [Minimum Operations to Reduce an Integer to 0](https://leetcode.com/problems/minimum-operations-to-reduce-an-integer-to-0) | Medium | Dynamic Programming, Greedy, Bit Manipulation | 7 | G A MS |
| 291 | [Unique Paths II](https://leetcode.com/problems/unique-paths-ii) | Medium | Array, Dynamic Programming, Matrix | 8 | G A MS M |
| 292 | [Sum Root to Leaf Numbers](https://leetcode.com/problems/sum-root-to-leaf-numbers) | Medium | Tree, Depth-First Search, Binary Tree | 7 | G A MS M |
| 293 | [Last Stone Weight](https://leetcode.com/problems/last-stone-weight) | Easy | Array, Heap (Priority Queue) | 10 | G A MS M |
| 294 | [Sort an Array](https://leetcode.com/problems/sort-an-array) | Medium | Array, Divide and Conquer, Sorting | 7 | G A MS M |
| 295 | [Max Points on a Line](https://leetcode.com/problems/max-points-on-a-line) | Hard | Array, Hash Table, Math | 9 | G A MS M Ap |
| 296 | [Meeting Rooms](https://leetcode.com/problems/meeting-rooms) | Easy | Array, Sorting, Quicksort | 9 | G A MS M Ap |
| 297 | [Nested List Weight Sum](https://leetcode.com/problems/nested-list-weight-sum) | Medium | Depth-First Search, Breadth-First Search | 6 | G A M |
| 298 | [Maximum Width of Binary Tree](https://leetcode.com/problems/maximum-width-of-binary-tree) | Medium | Tree, Depth-First Search, Breadth-First Search | 8 | G A MS M |
| 299 | [Design Tic-Tac-Toe](https://leetcode.com/problems/design-tic-tac-toe) | Medium | Array, Hash Table, Design | 8 | G A MS M Ap |
| 300 | [Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits) | Easy | Divide and Conquer, Bit Manipulation | 8 | G A MS M Ap |

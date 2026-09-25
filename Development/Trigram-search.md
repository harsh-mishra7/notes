# Database Indexes: Inverted Index & Trigram Index

## 1. Inverted Index

### What is an Inverted Index?

An **inverted index** is a data structure that maps content (like words or terms) to their locations in a document or document set. Instead of storing "document → words", it stores "word → documents".

### How it Works

**Traditional Index (Forward Index):**
```
Document 1: "the quick brown fox"
Document 2: "the lazy dog"
Document 3: "brown dog runs"
```

**Inverted Index:**
```
"the"    → [Document 1, Document 2]
"quick"  → [Document 1]
"brown"  → [Document 1, Document 3]
"fox"    → [Document 1]
"lazy"   → [Document 2]
"dog"    → [Document 2, Document 3]
"runs"   → [Document 3]
```

### Use Cases
- Full-text search engines (Elasticsearch, Solr)
- Search functionality in databases
- Information retrieval systems

### Advantages
- Fast keyword lookups
- Efficient for Boolean queries (AND, OR, NOT)
- Scalable for large document collections

### Disadvantages
- Requires more storage space
- Slower for range queries
- Updates can be expensive (need to reindex)

---

## 2. PostgreSQL Trigram Index

### What is a Trigram?

A **trigram** is a group of three consecutive characters from a string. It's used for approximate string matching and pattern searching.

### Example Trigrams

For the word `"cat"`:
```
" c", "ca", "at", "t "  (with padding spaces)
```

For the word `"hello"`:
```
" h", "he", "el", "ll", "lo", "o "
```

### PostgreSQL Trigram Index (GIN Index)

PostgreSQL provides the `pg_trgm` extension which creates a **GIN (Generalized Inverted Index)** on trigrams.

**Installation:**
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

**Creating a Trigram Index:**
```sql
CREATE INDEX idx_users_name_trigram ON users USING gin (name gin_trgm_ops);
```

### How Trigram Index Works Internally

**Example:** Table with users:
```
id | name
---|-------
1  | Alice
2  | Alexander
3  | Bob
4  | Albert
```

**Trigrams extracted:**
```
"Alice"     → [" a", "al", "li", "ic", "ce", "e "]
"Alexander" → [" a", "al", "le", "ex", "xa", "an", "nd", "de", "er", "r "]
"Bob"       → [" b", "bo", "ob", "b "]
"Albert"    → [" a", "al", "lb", "be", "er", "rt", "t "]
```

**Inverted Trigram Index:**
```
" a"  → [Alice(1), Alexander(2), Albert(4)]
"al"  → [Alice(1), Alexander(2), Albert(4)]
"li"  → [Alice(1)]
"ic"  → [Alice(1)]
"ce"  → [Alice(1)]
" b"  → [Bob(3), Albert(4)]
...
```

### Advantages of Trigram Index
- Fast **LIKE** and **ILIKE** queries: `WHERE name LIKE '%alic%'`
- Fast similarity searches: `WHERE name % 'alice'` (with `pg_trgm`)
- Useful for typo tolerance and fuzzy matching
- Works well with partial string matching

### When to Use
```sql
-- Pattern matching queries that benefit from trigram index
SELECT * FROM users WHERE name LIKE '%alex%';        -- Fast
SELECT * FROM users WHERE name ILIKE '%JOHN%';       -- Fast
SELECT * FROM users WHERE name % 'jon';              -- Fast (similarity)
```

---

## 3. Query Execution Using Trigram Index

### Query Example: LIKE Search

**Query:**
```sql
SELECT * FROM users WHERE name LIKE '%alic%';
```

### Step-by-Step Execution

#### Step 1: Extract Trigrams from Pattern
PostgreSQL extracts trigrams from the search pattern `'%alic%'`:
```
Trigrams from "alic": ["al", "li", "ic"]
```

#### Step 2: Look Up in Index
PostgreSQL searches the GIN index for documents containing ALL these trigrams:
```
Index lookup:
  "al" → [Alice(1), Alexander(2), Albert(4)]
  "li" → [Alice(1)]
  "ic" → [Alice(1)]

Intersection: [Alice(1)] ✓
```

#### Step 3: Candidate Filtering
Results from index (candidates):
```
Row 1: Alice
```

#### Step 4: Refinement/Verification
Since index-only scan may not be exact, PostgreSQL verifies candidates with the actual LIKE condition:
```
Alice LIKE '%alic%' ? → TRUE ✓
```

#### Step 5: Return Results
```
id | name
---|------
1  | Alice
```

---

### Full Query Execution Example with Multiple Results

**Query:**
```sql
SELECT * FROM users WHERE name LIKE '%al%';
```

**Execution:**

1. **Extract Trigrams:** `["al"]` from pattern `'%al%'`

2. **Index Lookup:**
   ```
   "al" → [Alice(1), Alexander(2), Albert(4)]
   ```

3. **Candidate Set:** `[Alice(1), Alexander(2), Albert(4)]`

4. **Refinement (verify each candidate):**
   ```
   Alice     LIKE '%al%' ? → TRUE  ✓
   Alexander LIKE '%al%' ? → TRUE  ✓
   Albert    LIKE '%al%' ? → TRUE  ✓
   ```

5. **Results:**
   ```
   id | name
   ---|----------
   1  | Alice
   2  | Alexander
   4  | Albert
   ```

---

### Similarity Search Example

**Query with similarity operator (requires `pg_trgm`):**
```sql
SELECT * FROM users WHERE name % 'alice' ORDER BY similarity(name, 'alice') DESC;
```

**Execution:**

1. **Extract Trigrams:** Trigrams from `'alice'` → `[" a", "al", "li", "ic", "ce", "e "]`

2. **Index Lookup:** Find all rows with at least one matching trigram

3. **Similarity Calculation:** Calculate similarity score for each candidate
   ```
   similarity("Alice", "alice") = 0.75 (high match)
   similarity("Alexander", "alice") = 0.42 (medium)
   similarity("Albert", "alice") = 0.33 (lower)
   similarity("Bob", "alice") = 0 (no match)
   ```

4. **Results (ordered by similarity):**
   ```
   id | name      | similarity
   ---|-----------|----------
   1  | Alice     | 0.75
   2  | Alexander | 0.42
   4  | Albert    | 0.33
   ```

---

## 4. Performance Considerations

### When Trigram Index is Beneficial
- Searching for text patterns within strings
- Large text columns (name, description, etc.)
- Frequent LIKE/ILIKE queries with leading wildcards (`%pattern%`)
- Similarity/typo tolerance searches

### When to Avoid
- Exact match searches (use B-tree index instead)
- Prefix searches without leading wildcard (`pattern%` - B-tree is better)
- Queries on small tables (sequential scan might be faster)
- Frequent updates (index maintenance overhead)

### Index Size Trade-off
```
Table: 1 million rows with average 20 char names

B-tree index size:  ~20 MB
Trigram GIN index:  ~80-120 MB (more storage needed)
```

---

## 5. Summary

| Aspect | Inverted Index | Trigram Index |
|--------|----------------|---------------|
| **What** | Maps terms to documents | Maps 3-char sequences to rows |
| **Use** | Full-text search | Pattern/similarity matching |
| **Query Type** | Keyword search | LIKE, ILIKE, similarity (%) |
| **Speed** | Very fast for keywords | Fast for substring patterns |
| **Storage** | High overhead | Moderate overhead |
| **Precision** | High (exact terms) | Approximate (requires verification) |

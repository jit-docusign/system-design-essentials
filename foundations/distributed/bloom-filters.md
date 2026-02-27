# Bloom Filters

## What it is

A **Bloom filter** is a space-efficient probabilistic data structure that answers the question: **"Is element X in this set?"** with two possible answers:
- **"Definitely not in the set"** — 100% correct, no false negatives.
- **"Probably in the set"** — may be wrong (false positive), never wrong in the negative direction.

A Bloom filter can represent a set of millions of elements using kilobytes of memory, at the cost of a configurable false positive rate. It **cannot remove elements** (without an extension) and **cannot return the elements** themselves — it only answers membership queries.

**Used in**: Cassandra (SSTable membership), HBase, LevelDB/RocksDB (key existence), PostgreSQL/Redshift (join optimization), web caches (crawled URL detection), network packet filters, malicious URL detection.

---

## How it works

### Data Structure

A Bloom filter consists of:
- A **bit array** of M bits (all initialized to 0).
- **K hash functions** that map any element to K positions in the bit array.

```
Bit array (M=20 bits):
  Index: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19
  Value: 0  0  0  0  0  0  0  0  0  0   0  0  0  0  0  0  0  0  0  0
```

### Insert Operation

To add an element `x`:
1. Compute K hash values: `h1(x), h2(x), ..., hK(x)`.
2. Set the bits at each of those positions to 1.

```
Add "apple" with K=3 hash functions:
  h1("apple") = 3  → set bit 3 = 1
  h2("apple") = 9  → set bit 9 = 1
  h3("apple") = 15 → set bit 15 = 1

Bit array:
  Index: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19
  Value: 0  0  0  1  0  0  0  0  0  1   0  0  0  0  0  1  0  0  0  0

Add "banana":
  h1("banana") = 1  → set bit 1 = 1
  h2("banana") = 9  → set bit 9 = 1 (already 1)
  h3("banana") = 17 → set bit 17 = 1

Bit array:
  Index: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19
  Value: 0  1  0  1  0  0  0  0  0  1   0  0  0  0  0  1  0  1  0  0
```

### Query Operation

To check if element `y` is in the set:
1. Compute K hash values: `h1(y), h2(y), ..., hK(y)`.
2. Check all K bits.
3. If **all K bits are 1**: element is "probably in the set."
4. If **any bit is 0**: element is **definitely not** in the set.

```
Query "apple":
  h1("apple") = 3  → bit[3] = 1 ✓
  h2("apple") = 9  → bit[9] = 1 ✓
  h3("apple") = 15 → bit[15] = 1 ✓
  All 1s → "Probably in set" → CORRECT (was inserted)

Query "mango":
  h1("mango") = 5  → bit[5] = 0 ✗
  A 0 found → "Definitely NOT in set" → CORRECT

Query "cherry" (false positive):
  h1("cherry") = 1  → bit[1] = 1 ✓ (set by "banana")
  h2("cherry") = 9  → bit[9] = 1 ✓ (set by "apple"/"banana")
  h3("cherry") = 17 → bit[17] = 1 ✓ (set by "banana")
  All 1s → "Probably in set" → FALSE POSITIVE ("cherry" was never inserted!)
```

### Implementation

```python
import hashlib
import math

class BloomFilter:
    def __init__(self, capacity: int, false_positive_rate: float = 0.01):
        # Optimal bit array size
        self.m = -int(capacity * math.log(false_positive_rate) / (math.log(2) ** 2))
        # Optimal number of hash functions
        self.k = int(self.m / capacity * math.log(2))
        self.bit_array = bytearray(self.m // 8 + 1)
    
    def _hashes(self, item: str):
        """Generate k hash positions for an item."""
        for i in range(self.k):
            h = hashlib.sha256(f"{item}:{i}".encode()).hexdigest()
            yield int(h, 16) % self.m
    
    def add(self, item: str):
        for pos in self._hashes(item):
            self.bit_array[pos // 8] |= (1 << (pos % 8))
    
    def might_contain(self, item: str) -> bool:
        """Returns False = definitely not present. True = probably present."""
        return all(
            self.bit_array[pos // 8] & (1 << (pos % 8))
            for pos in self._hashes(item)
        )

# Example: 
bf = BloomFilter(capacity=10000, false_positive_rate=0.01)
# bit_array size: ~12KB for 10,000 elements with 1% FP rate

bf.add("user:123")
bf.add("user:456")

print(bf.might_contain("user:123"))   # True (definitely added)
print(bf.might_contain("user:789"))   # False (definitely not)
print(bf.might_contain("user:000"))   # False or True (1% chance of True = false positive)
```

### False Positive Rate Formula

$$
P_{fp} = \left(1 - e^{-kn/m}\right)^k
$$

Where:
- $m$ = number of bits
- $n$ = number of inserted elements
- $k$ = number of hash functions

Optimal $k$ minimizes false positives: $k = \frac{m}{n} \ln 2$

At 1% false positive rate with 10,000 elements: bit array ≈ 12 KB.  
At 1% FP rate with 1,000,000 elements: bit array ≈ 1.2 MB.
Compare to storing 1M 16-byte UUIDs: 16 MB — 13× larger.

### Use Case: Cassandra SSTables

Cassandra stores data in immutable SSTable files. To check if a key exists in an SSTable:
- Without Bloom filter: would need to scan or binary-search every SSTable on disk (expensive).
- With Bloom filter: query the per-SSTable Bloom filter (in memory) first.
  - If "definitely not" → skip the SSTable entirely.
  - If "probably yes" → read from SSTable on disk.

Result: dramatically fewer disk reads for non-existent keys (a common pattern in read-heavy workloads).

```
Query: GET key="user:999"

SSTable 1: Bloom filter says "definitely not" → SKIP (no disk I/O)
SSTable 2: Bloom filter says "definitely not" → SKIP
SSTable 3: Bloom filter says "probably yes" → read from disk → found!
SSTable 4: Bloom filter says "definitely not" → SKIP

Result: only 1 disk read instead of 4.
```

### Counting Bloom Filter (Supports Deletion)

Standard Bloom filters don't support deletion — setting bits back to 0 can remove bits shared by other elements. A **Counting Bloom Filter** replaces each bit with a small counter (4 bits):
- Add: increment the K counters.
- Delete: decrement the K counters.

Trade-off: 4× more memory per slot.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Extremely space-efficient (bits per element) | False positives — cannot be eliminated entirely |
| O(K) constant time insert and query | Cannot delete (standard version) |
| No disk I/O for negative queries (great for caches) | Cannot list elements or return their values |
| Configurable false positive rate | FP rate increases as more elements are added beyond capacity |

---

## When to use

- **Cache miss optimization**: before querying a slow backend, check a Bloom filter. If "definitely not," return early.
- **Database key existence checks** (Cassandra, HBase, LevelDB): avoid reading SSTable files for missing keys.
- **Duplicate URL detection**: web crawlers check if a URL was already visited.
- **Email spam filtering**: set of known spam domains.
- **Distributed join optimization**: broadcast a Bloom filter of one join side to filter the other side. (Bloom filter joins in Spark/Flink.)

Not appropriate when:
- Any false positive is unacceptable (use a hash set).
- You need to retrieve elements or verify their content.
- The set is small enough to fit in memory as a hash set.

---

## Common Pitfalls

- **Using beyond capacity**: adding more elements than the configured capacity increases the false positive rate beyond design targets. Monitor fill rate and rebuild the filter when approaching capacity.
- **Sharing a Bloom filter across different keyspaces**: two different types of keys inserted in the same filter increase false positives for each. Use separate filters per logical set.
- **Not sizing correctly**: common rule of thumb — each element requires approximately `-log(fp_rate) / ln(2)^2` bits. For 1% FP rate: ~9.6 bits per element; for 0.1% FP rate: ~14.4 bits per element.

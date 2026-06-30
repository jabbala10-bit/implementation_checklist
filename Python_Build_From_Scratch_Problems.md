# Python "Build From Scratch" — Worked System Design Problems for Google Interview

**Companion to Python_Best_Practices_Quick_Reference.md · Gunasekar Jabbala**

> These are the specific "implement X from scratch" prompts that recur across Google Principal-level Python rounds — testing whether you understand the mechanism underneath a standard library convenience, not just whether you can call functools.lru_cache. Each problem: the ask, the naive-but-wrong approach, the correct implementation, the follow-ups that actually get asked, complexity analysis.

---

## Table of Contents

1. [LRU Cache From Scratch](#1-lru-cache-from-scratch)
2. [Rate Limiter (Token Bucket)](#2-rate-limiter-token-bucket)
3. [Thread-Safe Singleton](#3-thread-safe-singleton)
4. [A Simple Thread Pool](#4-a-simple-thread-pool)
5. [A Trie (Prefix Tree)](#5-a-trie-prefix-tree)
6. [A Bounded Blocking Queue (Producer-Consumer)](#6-a-bounded-blocking-queue-producer-consumer)

---

## 1. LRU Cache From Scratch

**The ask:** "Implement an LRU (Least Recently Used) cache with O(1) get and put, without using functools.lru_cache or OrderedDict."

**Why this is asked:** it tests whether you actually understand why O(1) is achievable, combining a hash map for O(1) lookup with a doubly linked list for O(1) reordering and eviction, rather than just knowing the cache exists as a library feature.

**The naive-but-wrong approach:** a plain dict plus tracking recency with a separate list — but removing an item from the middle of a Python list is O(n), which breaks the O(1) requirement the moment you need to move a recently-accessed item.

**The correct implementation:**
```python
class Node:
    __slots__ = ("key", "value", "prev", "next")
    def __init__(self, key=None, value=None):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}  # key -> Node, gives O(1) lookup
        # sentinel head/tail nodes simplify edge cases (empty list, single node)
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _add_to_front(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_front(node)  # mark as most recently used
        return node.value

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, value)
        self.cache[key] = node
        self._add_to_front(node)
        if len(self.cache) > self.capacity:
            lru = self.tail.prev  # least recently used = right before sentinel tail
            self._remove(lru)
            del self.cache[lru.key]
```

**Why sentinel head/tail nodes matter, worth saying unprompted:** without them, you need special-case branching for "list is empty" or "removing the only node" — sentinels make every insert/remove operation uniform, no edge-case branches, which is a specific, nameable design choice interviewers notice.

**Complexity:** get and put are both O(1) — dict gives O(1) lookup, doubly linked list gives O(1) removal and insertion since you have a direct node reference, not needing to search.

**Common follow-ups:**
- "Make it thread-safe." Wrap get/put bodies in a threading.Lock; note that this serializes all access, which is correct but limits concurrency — a follow-up-to-the-follow-up is discussing sharded or segmented locking for higher throughput.
- "What if capacity is 0?" Handle explicitly: put should not insert anything if capacity is 0, an edge case worth testing per the testing-edge-cases discipline.
- "Extend to LFU, Least Frequently Used." Requires tracking frequency counts and a frequency-to-nodes mapping — a meaningfully harder follow-up that tests whether LRU was actually understood or just memorized.

---

## 2. Rate Limiter (Token Bucket)

**The ask:** "Implement a rate limiter allowing N requests per time window, supporting bursts."

**Why this is asked:** distinguishes candidates who understand the actual algorithm tradeoffs, token bucket versus fixed window versus sliding window, from those who only know "rate limiting" as a buzzword — directly relevant to the Model Serving document's token-bucket schema, now asked as a from-scratch implementation rather than a JSON config.

**The correct implementation, token bucket, which smooths bursts better than fixed window:**
```python
import time
import threading

class TokenBucketRateLimiter:
    def __init__(self, capacity: int, refill_rate: float):
        """
        capacity: max tokens the bucket can hold (controls burst size)
        refill_rate: tokens added per second
        """
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate
        self.last_refill = time.monotonic()  # monotonic, not time.time() - immune to system clock changes
        self.lock = threading.Lock()

    def _refill(self):
        now = time.monotonic()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now

    def allow_request(self, cost: int = 1) -> bool:
        with self.lock:
            self._refill()
            if self.tokens >= cost:
                self.tokens -= cost
                return True
            return False
```

**Why time.monotonic() instead of time.time(), worth saying unprompted:** time.time() reflects wall-clock time and can jump backward, for instance during NTP sync or a manual clock change, which would corrupt the elapsed-time calculation; time.monotonic() is guaranteed never to go backward, making it the correct choice for measuring intervals.

**Complexity:** O(1) per allow_request call — no loops, no data structure traversal, just arithmetic and a lock acquisition.

**Common follow-ups:**
- "Why token bucket over a fixed window counter?" Fixed window allows a burst of twice the limit right at the window boundary, N requests at the end of one window and N more immediately at the start of the next — token bucket smooths this since tokens refill continuously, not in discrete resets.
- "Make this work across multiple processes or machines." Single-process in-memory state doesn't work distributed; needs a shared store, such as Redis with INCR and EXPIRE, or a sliding-window log in a shared datastore — connects directly to the distributed coordination themes in the architecture series.
- "What's the cost of acquiring the lock on every single request at very high QPS?" A real, valid concern — lock contention becomes the bottleneck at extreme scale; sharding the rate limiter, for example per-user buckets each with its own lock, reduces contention, the same principle as per-tenant resource isolation.

---

## 3. Thread-Safe Singleton

**The ask:** "Implement a singleton pattern in Python that's safe under concurrent initialization."

**Why this is asked:** tests whether you understand the actual race condition in naive singleton implementations, and whether you know Python-specific idioms, metaclasses and __new__, rather than just porting a Java-style pattern verbatim.

**The naive-but-wrong approach, with a race condition on first creation:**
```python
class NaiveSingleton:
    _instance = None
    def __new__(cls):
        if cls._instance is None:           # <- two threads can both pass this check
            cls._instance = super().__new__(cls)  #    before either assigns, creating two instances
        return cls._instance
```

**The correct, thread-safe implementation, double-checked locking:**
```python
import threading

class Singleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:        # first check without lock - fast path once initialized
            with cls._lock:
                if cls._instance is None:  # second check WITH the lock - prevents the race
                    cls._instance = super().__new__(cls)
        return cls._instance
```

**Why the check happens twice, worth explaining precisely:** the first check, without a lock, is a fast path — once the singleton exists, every subsequent call skips the lock entirely, avoiding unnecessary lock contention. The second check, inside the lock, is the actual race-condition fix — if two threads both pass the first check simultaneously, only one acquires the lock first, creates the instance, and the second thread's locked check now correctly sees the instance already exists and skips re-creating it.

**Alternative idiom, module-level singleton, often the more Pythonic answer:** Python modules are only imported and executed once and cached in sys.modules, so a plain module-level instance is naturally a singleton with no explicit pattern needed:
```python
# config_singleton.py
class _Config:
    def __init__(self):
        self.settings = load_settings()

config = _Config()  # module import is inherently thread-safe and happens once
# elsewhere: from config_singleton import config
```
**Worth stating this alternative unprompted** — it signals you know Python's import system provides this property natively, rather than always reaching for the more complex __new__-based pattern out of habit.

**Common follow-ups:**
- "What about __init__ running multiple times on the same singleton instance?" __new__ controls instance creation, but __init__ still runs every time Singleton() is called, even returning the same instance — guard with an initialization flag if __init__ has side effects you don't want repeated.
- "Is a singleton actually good design here?" A fair pushback to be ready for — singletons introduce global mutable state and make testing harder, since it's hard to isolate test instances; dependency injection is often the more testable alternative, and naming this tradeoff unprompted shows design judgment beyond just implementing the pattern correctly.
---

## 4. A Simple Thread Pool

**The ask:** "Implement a basic thread pool that executes submitted tasks using a fixed number of worker threads, without using `concurrent.futures.ThreadPoolExecutor`."

**Why this is asked:** tests understanding of producer-consumer coordination, queue-based work distribution, and graceful shutdown — the actual mechanics underneath the standard library convenience.

**The correct implementation:**
```python
import queue
import threading

class SimpleThreadPool:
    def __init__(self, num_workers: int):
        self.tasks = queue.Queue()
        self.workers = []
        self._shutdown = False
        for _ in range(num_workers):
            t = threading.Thread(target=self._worker_loop, daemon=True)
            t.start()
            self.workers.append(t)

    def _worker_loop(self):
        while True:
            task = self.tasks.get()  # blocks until a task is available
            if task is None:         # sentinel value signals shutdown
                break
            func, args, kwargs = task
            try:
                func(*args, **kwargs)
            except Exception as e:
                print(f"Task raised an exception: {e}")  # don't let one bad task kill the worker thread
            finally:
                self.tasks.task_done()

    def submit(self, func, *args, **kwargs):
        self.tasks.put((func, args, kwargs))

    def shutdown(self, wait: bool = True):
        if wait:
            self.tasks.join()  # block until all submitted tasks are processed
        for _ in self.workers:
            self.tasks.put(None)  # one sentinel per worker, to unblock and terminate each
        for t in self.workers:
            t.join()
```

**Why the `try/except` inside `_worker_loop` matters, worth saying unprompted:** without it, an unhandled exception in one submitted task would crash that worker thread permanently, silently reducing your pool's effective capacity over time as tasks raise errors — a subtle, production-relevant bug that's easy to miss in a quick implementation.

**Why `queue.Queue` specifically, not a plain list with manual locking:** `queue.Queue` is already thread-safe internally (uses a lock and condition variables under the hood) and provides blocking `get()`/`put()` plus `task_done()`/`join()` for tracking completion — reimplementing this with a list and manual `Lock` would be redoing work the standard library already does correctly, and a Principal-level answer should recognize that reuse here.

**Complexity:** `submit` is O(1) (queue append). Task execution throughput is bounded by `num_workers` — at most `num_workers` tasks run truly concurrently (subject to the GIL for CPU-bound tasks, per the GIL discussion in the Quick Reference document's Section 7).

**Common follow-ups:**
- *"What happens if `submit` is called after `shutdown`?"* → As written, this is a real bug — the queue would accept the task, but workers have already received their sentinel and exited; a production version needs a `self._shutdown` check (already declared but unused above) that raises or rejects new submissions after shutdown begins. Naming this gap yourself, unprompted, is stronger than having it found.
- *"How would you add a maximum queue size to apply backpressure?"* → `queue.Queue(maxsize=N)` makes `put()` block (or raise, with `put_nowait`) once full — directly connects to the backpressure concept from the architecture series' Kafka/streaming discussion.

---

## 5. A Trie (Prefix Tree)

**The ask:** "Implement a Trie supporting `insert`, `search` (exact match), and `starts_with` (prefix match)."

**Why this is asked:** tests whether you can build a tree-based structure correctly and reason about its complexity relative to alternatives (a naive list of strings with linear scan, or a hash set that can't efficiently support prefix queries at all).

**The correct implementation:**
```python
class TrieNode:
    __slots__ = ("children", "is_end_of_word")
    def __init__(self):
        self.children = {}       # char -> TrieNode
        self.is_end_of_word = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end_of_word = True

    def _find_node(self, prefix: str):
        node = self.root
        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]
        return node

    def search(self, word: str) -> bool:
        node = self._find_node(word)
        return node is not None and node.is_end_of_word

    def starts_with(self, prefix: str) -> bool:
        return self._find_node(prefix) is not None
```

**Complexity:** `insert`, `search`, and `starts_with` are all O(L) where L is the length of the word/prefix — independent of how many words are stored in the trie, which is the key advantage over a naive list scan (O(N*L) for N stored words).

**Why this matters beyond the abstract exercise, worth connecting explicitly:** this is the exact underlying data structure for autocomplete and prefix-based search features — directly relevant to the RAG document's discussion of search/retrieval systems, even though vector-based semantic search (the RAG document's main focus) solves a different problem (semantic similarity) than a trie's exact-prefix matching.

**Common follow-ups:**
- *"Add a method to return all words with a given prefix."* → Requires a DFS/BFS traversal from the prefix's node, collecting all paths to `is_end_of_word` nodes — tests whether you can compose the existing `_find_node` helper with a new traversal, rather than rewriting from scratch.
- *"What's the memory cost compared to a hash set of the same words?"* → A trie shares common prefixes across words (e.g., "cat" and "car" share the "ca" path), which can be more memory-efficient for datasets with many shared prefixes, but has per-node overhead (a dict per node) that can make it less efficient for datasets with little prefix overlap — a genuine, case-dependent tradeoff worth stating rather than claiming one is universally better.
- *"How would you handle Unicode/case-insensitivity?"* → Normalize input (lowercase, Unicode normalization) before insert/search — a real-world detail that's easy to skip in algorithm exercises but matters in production.

---

## 6. A Bounded Blocking Queue (Producer-Consumer)

**The ask:** "Implement a bounded blocking queue from scratch — `put` blocks if full, `get` blocks if empty — without using `queue.Queue`."

**Why this is asked:** this is the single best test of whether you understand `threading.Condition` correctly — a frequently-misunderstood primitive, and the actual mechanism underneath `queue.Queue` itself.

**The correct implementation:**
```python
import threading
from collections import deque

class BoundedBlockingQueue:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.items = deque()
        self.lock = threading.Lock()
        self.not_full = threading.Condition(self.lock)   # signaled when there's room to put
        self.not_empty = threading.Condition(self.lock)   # signaled when there's an item to get

    def put(self, item):
        with self.not_full:
            while len(self.items) >= self.capacity:
                self.not_full.wait()   # releases the lock while waiting, re-acquires on wake
            self.items.append(item)
            self.not_empty.notify()    # wake one thread waiting in get()

    def get(self):
        with self.not_empty:
            while len(self.items) == 0:
                self.not_empty.wait()
            item = self.items.popleft()
            self.not_full.notify()     # wake one thread waiting in put()
            return item
```

**Why `while` and not `if` around the wait, the single most important detail in this entire problem:** when a thread wakes from `wait()`, it must *re-check* the condition before proceeding — between being notified and actually acquiring the lock to resume, another thread might have already consumed the available slot/item. Using `if` instead of `while` is a classic, subtle concurrency bug that passes casual testing (works fine with 2 threads) but fails intermittently under real contention (3+ threads) — stating this explicitly, unprompted, is one of the highest-signal moments available in this entire document.

**Why two separate `Condition` objects sharing one lock, rather than one condition for both:** this lets `notify()` wake specifically the right kind of waiter (a producer waiting for space vs. a consumer waiting for data) rather than waking all waiters indiscriminately and having most of them re-check and go back to sleep — a real efficiency improvement at scale, though a single shared condition with a generic `notify_all()` is also a *correct*, if less efficient, alternative worth mentioning if asked about simplification.

**Complexity:** `put` and `get` are O(1) for the actual queue operation (deque append/popleft); the real "cost" here is in contention and context-switching under heavy concurrent use, not algorithmic complexity.

**Common follow-ups:**
- *"What if a thread is waiting and you want to shut down cleanly?"* → Needs a sentinel value or an explicit shutdown flag checked inside the `while` loop's condition, plus a `notify_all()` (not just `notify()`) on shutdown to wake every waiting thread so they can observe the shutdown state — the same sentinel pattern as the thread pool problem in Section 4.
- *"How does this compare to using `asyncio.Queue` in an async context?"* → `asyncio.Queue` provides the same bounded producer-consumer semantics but for single-threaded cooperative concurrency (`await queue.put()` / `await queue.get()`) rather than OS threads — choosing between them follows the exact threading-vs-asyncio decision table from the Quick Reference document's Section 7.

---

## Closing Notes

**The pattern across all six problems, worth internalizing rather than memorizing each individually:** every "build X from scratch" prompt is really testing whether you understand *why* the standard library's convenience version works — the data structure combination that makes O(1) possible (LRU cache), the precise semantics of a synchronization primitive (Condition's `while` vs `if`), the tradeoff between two valid algorithms (token bucket vs. fixed window). Memorizing these six implementations verbatim helps less than being able to derive the reasoning live for a seventh problem you haven't seen, which is exactly the skill the Estimation Mastery document's Section 6 generalization principle describes for a completely different category of problem — the underlying meta-skill (reason from mechanism, not memorized answer) is the same one this entire document series has been building toward all along.

---
title: 'Building a Generic Hashmap in C'
description: 'A walkthrough of my implementation, the bugs I ran into, and what they taught me.'
pubDate: 2026-09-09
---

## A quick note before we start

Day job: C#. C: a hobby, and a fairly new one. So a quick disclaimer before you read any further — I don't have years of C experience to draw on here. What follows is my best understanding, built by actually implementing it, getting it wrong a few times, and talking it through with an AI assistant along the way to get pointer mechanics and design questions explained until they clicked. I'm not presenting this as expert C, just as an honest account of how I built it and what I learned doing it.

The full code lives at [github.com/soerenlemke/kvstore_c](https://github.com/soerenlemke/kvstore_c), if you want to see it in context or follow along.

## Why

I've been working through the ["Build Your Own Database" tutorial](https://build-your-own.org/database/01_files) in Go, and after getting a feel for B+trees, atomic file writes, and node byte layouts, I wanted to actually build something myself instead of just following along. So I started a small key-value store project — first sketched out in Go to nail down the architecture (interfaces, a logging decorator, that kind of thing), and then, because I wanted something lower-level with no garbage collector, rewritten from scratch in C.

Before I could get to the store itself, I needed a hashmap. C doesn't ship with one, so this is my own implementation, built step by step, with a lot of help clarifying the pointer mechanics along the way.

## Design decisions

A few choices shaped the whole implementation:

**Bytes in, bytes out.** The hashmap doesn't know anything about my key-value store's `Entry` type. Keys and values are just `uint8_t*` plus a `size_t` length — no assumption that they're null-terminated strings, no assumption about what they represent. This is what makes it reusable outside this one project.

**Copy, don't borrow.** When you `put` a key/value pair, the hashmap makes its own copies. The caller's original buffers can be freed or reused right after the call returns without corrupting the store. This costs a `malloc` + `memcpy` per insert, but it makes ownership unambiguous.

**Separate chaining for collisions.** Each bucket is the head of a singly linked list. Simple to reason about, simple to implement, even if open addressing would probably be more cache-friendly.

**Opaque type in the header.** The public header only exposes `typedef struct HashMap HashMap;` — no fields. The actual struct definition, with `buckets`, `capacity`, and `count`, lives in the `.c` file. Callers can only interact with the hashmap through the functions I expose, never by reaching into its internals directly.

**C23.** `nullptr`, `auto`, and `bool`/`true`/`false` as real keywords instead of `<stdbool.h>` macros. Nothing exotic, just the modern defaults.

## The structs

```c
typedef struct HashMapNode {
    uint8_t* key;
    size_t key_len;
    uint8_t* value;
    size_t value_len;
    struct HashMapNode* next;
} HashMapNode;

typedef struct HashMap {
    HashMapNode** buckets;
    size_t capacity;
    size_t count;
} HashMap;
```

`HashMapNode` is a linked-list node — one per stored entry. `buckets` is an array of list heads: `capacity` many slots, each either `nullptr` or a pointer to the first node in that bucket's chain. That's why it's a double pointer — an array of pointers is itself a pointer, and each element in that array is *also* a pointer (to a node or to nothing).

The self-reference (`struct HashMapNode* next`) inside the struct is also why the tag name (`struct HashMapNode`) had to appear both after `struct` and again at the end of the `typedef` — at the point where `next` is declared, the `typedef` alias doesn't exist yet, only the tag does.

## Hashing

FNV-1a, adapted to work over a byte range instead of a null-terminated string:

```c
static size_t hash_bytes(const uint8_t* key, const size_t key_len) {
    size_t hash = 1469598103934665603ULL;
    for (size_t i = 0; i < key_len; i++) {
        hash ^= key[i];
        hash *= 1099511628211ULL;
    }
    return hash;
}
```

The two constants aren't arbitrary — they're the standard FNV-1a offset basis and prime for 64-bit hashes. You don't derive them yourself, you just use them, same as you'd use a fixed value for π rather than recomputing it.

The bucket index is just `hash_bytes(key, key_len) % map->capacity` — squeezing an arbitrarily large hash into a valid array index.

## Where I actually messed up

This is the part I wanted to write down, because these were the mistakes that taught me the most.

### Comparing pointers instead of contents

My first version of the "does this key already exist" check looked like:

```c
if (node->key == key) {
```

This compares two *addresses*, not two byte sequences. Two different buffers holding identical bytes will basically never have the same address. I needed a real comparison, first by length and then by content:

```c
static bool keys_equal(const uint8_t* a, const size_t a_len, const uint8_t* b, const size_t b_len) {
    if (a_len != b_len) {
        return false;
    }
    return memcmp(a, b, a_len) == 0;
}
```

The length check first matters — `memcmp` alone, without checking lengths, would happily compare `"foo"` against the first three bytes of `"foobar"` and call them equal.

### Writing to the pointer instead of through it

`hashmap_get` needs to hand back both the found value *and* whether it was found at all. Since a C function can only directly return one value, the value comes back through an out-parameter: `uint8_t** out_value`.

My first attempt:

```c
out_value = node->value;
```

This overwrites the local copy of the pointer inside the function — it has zero effect on the caller. What I actually needed was to write *through* the pointer, to the location it points at:

```c
*out_value = node->value;
```

The distinction between "change what this pointer points to" and "change what's stored at the address this pointer points to" is the whole reason double pointers as out-parameters exist, and it's an easy one to get backwards.

### Mutating the wrong node

While inserting a new node, after a full pass through the bucket's chain found no existing key, I wrote:

```c
memcpy(key_copy, key, key_len);
free(node->key);
node->key = key_copy;
```

`node` at this point is `nullptr` — the search loop only reaches the "create new node" branch once it has walked off the end of the list. I meant `new_node`, the struct I'd just allocated a few lines earlier. `free(node->key)` on a null pointer is an immediate crash, and even set aside the crash, a freshly `malloc`'d node has nothing to free yet — there was no reason for a `free()` call there at all.

### Forgetting to advance the loop

More than once, a search loop was missing its "move to the next node" step:

```c
while (node != nullptr) {
    if (keys_equal(...)) { ... }
    // missing: node = node->next;
}
```

Without it, the loop either never terminates or keeps re-checking the same node — and in `hashmap_remove`, the bug was worse: after freeing a node, the loop tried to read `node->next` on memory that no longer belonged to me. Use-after-free, not just an infinite loop.

### Double free in `hashmap_remove`

The relinking logic (bucket head vs. a node with a predecessor) originally called `hashmap_node_free(node)` inside both branches of an `if`/`else`, *and* once more after the `if`/`else` block — freeing the same node twice. This is the kind of bug that doesn't always crash immediately, which makes it worse: it can corrupt the allocator's internal bookkeeping and cause an unrelated `malloc` to fail much later, far from where the actual bug lives.

## Testing

Rather than a scratch `main.c`, I set up a separate CMake target for tests, linked against the hashmap as its own library:

```cmake
add_library(hashmap
    src/hashmap/hashmap.c
)
target_include_directories(hashmap PUBLIC src/hashmap)

enable_testing()
add_executable(hashmap_tests
    tests/hashmap_tests.c
)
target_link_libraries(hashmap_tests hashmap)
add_test(NAME hashmap_tests COMMAND hashmap_tests)
```

The tests themselves are plain `assert()` calls, no framework — put/get round-trip, get on a missing key, update an existing key, remove the first node in a chain vs. a middle node, remove a missing key, and, deliberately, a test with `capacity = 1` to force every key into the same bucket so the chaining logic actually gets exercised instead of every key landing in its own empty slot.

I also compile the debug configuration with AddressSanitizer:

```cmake
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -fsanitize=address -fno-omit-frame-pointer -g")
set(CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG} -fsanitize=address")
```

One thing worth knowing if you're on Apple Silicon: Valgrind doesn't work reliably on macOS/ARM, so ASan is really the only practical option. And ASan's use-after-free, double-free, and buffer-overflow detection all work fine on macOS — but its leak detector doesn't run reliably there. A clean ASan run tells you your memory *safety* is fine; it doesn't tell you your memory is fully *freed*. For that, Xcode's Instruments (Leaks template) is the more reliable macOS-native option.

## What's still missing

No resizing. `capacity` is fixed at creation time, so as the map fills up, chains get longer and lookups degrade from O(1) toward O(n) — exactly the problem separate chaining is supposed to avoid once you have enough entries. That's the next thing to fix before this becomes the backing store for anything real.

## Conclusion

None of the individual bugs here were exotic — pointer vs. pointee, wrong variable name, missing loop increment, one extra `free()` call. What made them worth writing down is that they're exactly the class of mistake C makes easy to introduce and easy to miss: nothing in the type system stops you from writing `out_value = x` instead of `*out_value = x`, and nothing stops a stray `free()` from compiling just fine while quietly corrupting your heap.

The full project — this hashmap plus the key-value store being built on top of it — is on GitHub at [github.com/soerenlemke/kvstore_c](https://github.com/soerenlemke/kvstore_c). If you're working through something similar, I'd say the single most useful habit was writing tests that deliberately force collisions and chain traversal — most of these bugs were invisible until a bucket actually held more than one node.
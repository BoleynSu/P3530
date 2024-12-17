---
title: Intrinsic for reading uninitialized memory
document: P3530
date: 2024-12-17
audience: TODO
author:
  - name: Boleyn Su
    email: <boleyn.su@gmail.com>
  - name: Gašper Ažman
    email: <gasper.azman@gmail.com>
toc: true
references:
  - id: SS
    citation-label: Sparse
    title: "An Efficient Representation for Sparse Sets"
    author:
      - family: Briggs
        given: Preston
      - family: Torczon
        given: Linda
    URL: https://dl.acm.org/doi/pdf/10.1145/176454.176484
  - id: FRZ
    citation-label: Freeze
    title: "Taming Undefined Behavior in LLVM"
    author:
      - given: Juneyoung
        family: Lee
      - given: Yoonseung
        family: Kim
      - given: YoungJu
        family: Song
      - given: Chung-Kil
        family: Hur
      - given: Sanjoy
        family: Das
      - given: David
        family: Majnemer
      - given: John
        family: Regehr
      - given: Nuno P.
        family: Lopes
    URL: https://sf.snu.ac.kr/publications/undefllvm.pdf
---

# Abstract

With current standard, it is impossible to safely read uninitialized memory. Thus,
data structures or algorithms relying on reading uninitialized memory cannot be
implemented in standard C++.

We therefore propose to add a new intrinsic to allow reading uninitialized memory.

# Motivation

With the adoption of erroneous behavior[@P2795R5], reading uninitialized memory has a more defined behavior
than current C++23 behavior. It makes the code relying on a specific C++ compiler's implementation
for handling the undefined behavior caused by reading uninitialized memory no longer possible. However, there are
indeed (though rare) use cases where reading uninitialized memory can be useful. Therefore, instead
of making it always an error, e.g. undefined behavior or erroneous behavior, we should propose
a way to have defined behavior for reading uninitialized memory.

# Background

There is a well-known data structure to present sparse sets [@SS] as shown below.

```c++
template <int n>
class SparseSet {
  // The invariants are index_of[elements[i]] == i for all 0<=i<size
  // and elements[0..size-1] contains all elements in the set.
  // These invariants guarantee the correctness.
  int elements[n];
  int index_of[n];
  int size;
public:
  SparseSet() : size(0) {}  // we do not initialize elements and index_of
  void clear() { size = 0; }
  bool find(int x) {
    // assume x in [0, n)
    int i = index_of[x];
    // There is a chance we read index_of[x] before writing to it
    // which is totally fine (if we assume reading uninitialized
    // variable not UB).
    // Because the invariants we maintain guaranteed that if and only if
    // the below condition holds, x in in the set.
    return 0 <= i && i < size && elements[i] == x;
  }
  void insert(int x) {
    // assume x in [0, n)
    if (find(x)) {
      return;
    }
    // The invariants are maintained.
    index_of[x] = size;
    elements[size] = x;
    size++;
  }
  void remove(int x) {
    // assume x in [0, n)
    if (!find(x)) {
      return;
    }
    // The invariants are maintained.
    size--;
    int i = index_of[x];
    elements[i] = elements[size];
    index_of[elements[size]] = i;
  }
};
```

However, in the above implementation, we may read uninitialized memory in the `find` member function,
which is undefined behavior or erroneous behavior according to the current standard.

To author's knowledge, it is impossible to implement a data structure supporting the same set of operations
with a worse time complexity of O(1) for each operation without relying on reading uninitialized memory.

# Analysis

The LLVM project already supports `freeze` instruction which is the lower level equivalent of our proposal [@FRZ].

# Proposal

We propose to add `std::read_maybe_uninitialized(const T& v)`. `read_maybe_uninitialized(v)`
will return the value of `v` if `v` is already written. Otherwise, it returns an arbitrary
value. Before `v` is written, each invoke of `read_maybe_uninitialized(v)` can return different
values.

# Wording

TODO


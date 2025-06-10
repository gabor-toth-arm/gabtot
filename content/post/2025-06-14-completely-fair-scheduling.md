---
layout:     post 
title:      "Completely Fair Scheduling"
subtitle:   "The red-black magic of Linux"
date:       2025-06-14
author:     "Gabor Toth"
URL:        "/2025/06/14/completely-fair-scheduling"
image:      "img/post-bg-coffee.jpeg"
draft:      false
---

Linux scheduling lies at the heart of how the kernel manages time and resources. It ensures that every process—from real-time tasks to background jobs—gets a fair share of CPU time. The scheduler must constantly balance performance, responsiveness, and fairness across diverse workloads.

While Linux supports multiple schedulers, its default and most widely used is the Completely Fair Scheduler (CFS). This post dives into how CFS works and the red-black tree magic behind its design.

## <a href="https://docs.kernel.org/scheduler/sched-design-CFS.html#cfs-scheduler">Completely Fair Scheduler (CFS)</a>

The Completely Fair Scheduler (CFS) manages runnable processes using a self-balancing binary tree known as a Red-Black Tree. The central idea is fairness: each process receives CPU time proportional to its virtual runtime (vruntime), a metric that increases as the task runs. Tasks with lower vruntime are under-served and scheduled sooner.

The vruntime is weighted by task priority (niceness): lower-priority tasks accumulate it faster, making them yield CPU time sooner; high-priority tasks accumulate more slowly, getting longer time slices.

To reduce overhead, CFS doesn’t switch tasks constantly. It defines a minimum time slice (a few milliseconds), allowing tasks to run for meaningful periods before preemption.

### Key CFS Mechanisms

- Each runnable task is stored as a node in a Red-Black Tree.
- The sorting key in the tree is the task's `vruntime`.
- The task with the smallest `vruntime` (leftmost node) is selected to run next.
- When a task runs, its `vruntime` increases, causing it to move right in the tree.
- When a task becomes runnable again, it's reinserted at the appropriate position.

Currently, CFS relies on the <a href="https://www.kernel.org/doc/html/v6.16-rc1/core-api/rbtree.html">rbtree</a> API, defined in `<linux/rbtree.h>`. Although CFS strongly depends on the rbtree data structure, it remains versatile and can be applied to a variety of custom use cases independently of CFS. A comprehensive grasp of the red-black tree algorithm isn't necessary to understand CFS, however, a basic familiarity can provide useful context. The upcoming chapters will offer a concise overview and illustrative examples to aid comprehension.

## Red-Black Trees

A standard binary tree can quickly become skewed, turning into a linked list and degrading performance. For example:
```bash
      A(10)
         \
       B(20)
          \
        C(30)
           \
         D(40)
```

To prevent this, Linux uses a Red-Black Tree, a self-balancing binary search tree. It guarantees that the longest path from root to leaf is no more than twice the shortest path. This keeps operations—search, insert, delete—at O(log n) time.

### Red-Black Tree Properties

A binary tree is a Red-Black Tree if it satisfies the following rules:

- Every node is either red or black.
- The root node is always black.
- All leaf (NIL) nodes are black.
- Red nodes cannot have red children (no two reds in a row).
- Every path from a node to its NIL descendants has the same number of black nodes.

## Insertion

New nodes are inserted like in a standard binary search tree: left if smaller, right if larger. Afterward, rebalancing is often needed to maintain the above rules, using **rotations**:

### 1. Left Rotation

Used when a right child causes imbalance. For example:

**Before:**
```bash
       5(B)
     /      \
   2(R)    10(R)
           /    \
        8(B)   12(B)
        /  \
      6(R) 9(R)
```

**After left-rotating at 5:**
```bash
       10(R)
      /     \
   5(B)     12(B)
  /   \
2(R)  8(B)
      /  \
    6(R) 9(R)
```

### 2. Right Rotation

Used when a left child causes imbalance:

**Before:**
```bash
       10(R)
      /     \
   5(B)     12(B)
  /   \
2(R)  8(B)
      /  \
    6(R) 9(R)
```

**After right-rotating at 10:**
```bash
      5(B)
    /     \
  2(R)     10(R)
          /    \
        8(B)   12(B)
      /   \
    6(R) 9(R)
```

As you see we returned to the original tree!
Each rotation is **O(1)** complexity—it only changes a few pointers.

## Removal

If a node has two children, it's swapped with its in-order successor (smallest node in the right subtree), and that successor is removed. If it has one or no children, it can be directly removed.

Rebalancing may be necessary post-removal to preserve Red-Black properties.

## Search

This is where Red-Black Trees shine. To find the next task to run, CFS picks the **leftmost node**—the task with the **smallest `vruntime`**, i.e., the one that has received the least CPU time.

Although CFS relies on the rbtree API from `<linux/rbtree.h>`, it does not explicitly use the node colors during scheduling decisions. However, the red-black properties are still essential to maintain the tree’s balance and performance guarantees.

## Visual Example: CFS Scheduling with Red-Black Tree (sorted by vruntime)

Below is a step-by-step example of how CFS inserts tasks into a Red-Black Tree sorted by vruntime. Each node represents a task. Lower vruntime means the task is more under-served and is scheduled earlier.

The process includes standard BST insertions followed by rebalancing through rotations and recoloring to maintain Red-Black Tree properties.
For simplicity, NIL (null leaf) nodes are omitted.

Let's insert the following values into a Red-Black Tree: 10, 20, 30, 40, 50 (Lower vruntime → more under-served → higher scheduling priority)

Step 1: Insert Task A (vruntime = 10)
Tree is empty. Insert 10 as root and color it black.
```bash
    A(10,B)
```

Step 2: Add Task B (vruntime = 20)
Since 20 > 10 → insert into right subtree of 10.
```bash
    A(10,B)
        \
        B(20,R)
```

Step 3: Add Task C (vruntime = 30)
30 > 10 → right of 10 30 > 20 → right of 20
```bash
    A(10,B)
        \
        B(20,R)
            \
            C(30,R)
```

Violation: Two consecutive red nodes (20 & 30).
Fix: Left rotation at 10, recolor 20 to black and 10 to red.

```bash
      B(20,B)
     /    \
    A(10,R)  C(30,R)
```

Step 4: Add Task D (vruntime = 40)
40 > 20 → right of 20 40 > 30 → right of 30
```bash
      B(20,B)
     /    \
    A(10,R)  C(30,R)
               \
              D(40, R)
```

Violation: There are two consecutive red nodes. Recolor D to black.

```bash
       B(20,B)
        /    \
    A(10,R)  C(30,R)
                 \
               D(40, B)
```

Step 5: Add Task E (vruntime = 50)
50 > 20 → right 50 > 30 → right 50 > 40 → right
```bash
      B(20,B)
        /    \
    A(10,R)  C(30,R)
                 \
               D(40, B)
                 \
               E(50, R)
```
Violation: Consecutive reds (D, E), both are right children.
Fix: Left rotation at C's right child (node D) and recolor.

```bash
      B(20,B)
     /    \
    A(10,R)  D(40,R)
              /   \
          C(30,B) E(50, B)
```
Step 6: Select and Remove the Leftmost Task
The scheduler always picks the task with the smallest vruntime, which is the leftmost node. When Task A completes or yields it is removed from the tree:

```bash
      B(20,B)
        \
      D(40,R)
      /    \
  C(30,B)   E(50,B)
```

The Completely Fair Scheduler exemplifies the elegant trade-offs Linux makes under the hood, combining algorithmic rigor with real-world practicality. By using Red-Black Trees to balance fairness and performance, CFS remains efficient even as the number of tasks increases.

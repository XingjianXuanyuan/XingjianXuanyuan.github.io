---
layout: post
title: "Tree Data Structures in the Linux Kernel"
category: "Computing Systems"
---

<h2>Radix Tree</h2>

Linux's radix tree implementation lives in the file [<pre><code>lib/radix-tree.c</code></pre>](https://elixir.bootlin.com/linux/v5.4.42/source/lib/radix-tree.c). To use it,

~~~ C
#include <linux/radix-tree.h>
~~~

<h2>Red-Black Tree</h2>

**Definition 1.** A **read-black tree**{: style="color: red"} is a binary search tree which has the following *red-black properties*:
1. Every node is either red or black.
2. Every leaf (<pre><code>NULL</code></pre>) is black.
3. If a node is red, then both of its children are black.
4. Every simple path from a node to a descendant leaf contains the same number of black nodes.

Each node in the tree contains a value and up to two children; the node's value will be greater than that of all children in the "left" child branch, and less than that of all children in the "right" branch. (Thus, it is possible to serialize a red-black tree by performing a depth-first. left-to-right traversal.) Implementations of the red-black tree algorithms will usually include the **sentinel**{: style="color: red"} nodes as a convenient means of flagging that you have reached a leaf node. They are <pre><code>NULL</code></pre> black nodes of **Property 2**. Formally, a red-black tree with $n$ internal nodes has height at least $\log_{2}(n+1)$ but at most $2\log_{2}(n+1)$.

Linux's <pre><code>rbtree</code></pre> implementation lives in the file [<pre><code>lib/rbtree.c</code></pre>](https://elixir.bootlin.com/linux/v5.4.42/source/lib/rbtree.c). To use it,

~~~ C
#include <linux/rbtree.h>
~~~
---
layout: post
title: "Tree Data Structures in the Linux Kernel"
category: "Computing Systems"
---

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Radix Tree

Linux's radix tree implementation lives in the file [<code>lib/radix-tree.c</code>](https://elixir.bootlin.com/linux/v5.4.42/source/lib/radix-tree.c). To use it,

~~~ C
#include <linux/radix-tree.h>
~~~

### Integer ID Management

idr, ida, Andrew Morton

### XArray

Matthew Wilcox[^2]

## Red-Black Tree

**Definition 1.** A **read-black tree**{: style="color: red"} is a binary search tree which has the following *red-black properties*:
1. Every node is either red or black.
2. Every leaf (<code>NULL</code>) is black.
3. If a node is red, then both of its children are black.
4. Every simple path from a node to a descendant leaf contains the same number of black nodes.

Each node in the tree contains a value and up to two children; the node's value will be greater than that of all children in the "left" child branch, and less than that of all children in the "right" branch. (Thus, it is possible to serialize a red-black tree by performing a depth-first. left-to-right traversal.) Implementations of the red-black tree algorithms will usually include the **sentinel**{: style="color: red"} nodes as a convenient means of flagging that you have reached a leaf node. They are <code>NULL</code> black nodes of **Property 2**. Formally, a red-black tree with $n$ internal nodes has height at least $\log_{2}(n+1)$ but at most $2\log_{2}(n+1)$.

Linux's <code>rbtree</code> implementation lives in the file [<code>lib/rbtree.c</code>](https://elixir.bootlin.com/linux/v5.4.42/source/lib/rbtree.c). To use it,

~~~ C
#include <linux/rbtree.h>
~~~

[^1]: *Trees I: Radix Trees* by Jonathan Corbet, March 13, 2006, [lwn.net/Articles/175432/](https://lwn.net/Articles/175432/).

[^2]: *XArray* by Matthew Wilcox, [kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt](https://www.kernel.org/doc/html/latest/_sources/core-api/xarray.rst.txt).

[^3]: *Examining Linux 2.6 Page-Cache Performance* by Sonny Rao, Dominique Heger, and Steven Pratt, [landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf](https://landley.net/kdocs/ols/2005/ols2005v2-pages-87-98.pdf)
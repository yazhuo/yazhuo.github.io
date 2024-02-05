---
layout: post
title: SIEVE is simpler than LRU
---

{{ page.title }}
================

Caching is a method of storing temporary data for quick access to keep the online world running smoothly. But with limited space comes a critical decision: what to keep and discard. This is where **eviction algorithms** come into play. Our team recently designed a new cache eviction algorithm called **SIEVE**: it is very effective and simple with just one queue.

- [Website](https://sievecache.com) 
- [Paper](https://yazhuozhang.com/assets/pdf/nsdi24-sieve.pdf)

***Updates***: *The original writing style attracted considerable attention. We have updated the blog post for greater clarity and straightforwardness.* We also include an easy-to-reproduce result at [https://observablehq.com/@1a1a11a/sieve-miss-ratio-plots](https://observablehq.com/@1a1a11a/sieve-miss-ratio-plots). 

## The Importance of Simplicity
It is important to keep cache eviction algorithms simple. Complex algorithms can bring headaches from time to time. For example, they can be tricky to debug and analyze when the miss ratio is high. Moreover, complexity means CPU cycles are needed to make the decisions. 

On the flip side, simpler eviction methods usually do an excellent job of providing good predictability and throughput. 
For example, the Apache traffic server uses FIFO, [MemC3](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/fan) uses CLOCK, and [Segcache](https://www.usenix.org/conference/nsdi21/presentation/yang-juncheng) uses FIFO-Merge eviction algorithms. 
It's crucial to note that while these approaches excel in throughput, some compromise on cache efficiency. 

## Meet SIEVE
SIEVE is a cache eviction algorithm that decides what to keep in the cache and what to discard. It achieves both simplicity and efficiency.

### Design

**Data structure.** 
SIEVE requires only one queue and one pointer called "hand". 
**The queue maintains the insertion order between objects**.
Each object in the queue uses one bit to track the visited/non-visited status.
The hand points to the next eviction candidate in the cache and moves from the tail to the head. 

**SIEVE operations.**
A cache hit changes the visited bit of the accessed object to 1.
SIEVE does not need to perform any operation for a popular object whose visited bit is already set. 
During a cache miss, SIEVE examines the object pointed by the hand. If it has been visited, the visited bit is reset, and the hand moves to the next position (the retained object stays in the original position of the queue). 
It continues this process until it encounters an object with the visited bit being 0 and evicts it. After the eviction, the hand points to the next position (the previous object in the queue).
While an evicted object is in the middle of the queue most of the time, a new object is always inserted into the head of the queue.

![](/blog_images/sieve/sieve_diagram_animation.gif)


**SIEVE vs. CLOCK** At first glance, SIEVE is similar to CLOCK/Second Chance/FIFO-Reinsertion - *Note that they are different implementations of the same eviction algorithm*.
Each algorithm maintains a queue in which each object is associated with a visited bit to track its access status. 
Visited objects are retained (also called "survived") during an eviction. Notably, new objects are inserted at the head of the queue in both SIEVE and FIFO-Reinsertion. 
However, the hand in SIEVE moves from the tail to the head over time, whereas the hand in FIFO-Reinsertion stays at the tail. 
*The key difference is where a retained object is kept.* SIEVE keeps it in the old position, while FIFO-Reinsertion inserts it at the head, together with newly inserted objects.

![](/blog_images/sieve/sieve-diagram.png)

See the [sieve cache implementation code](#sieve-cache-code) at the end of this blog post for a detailed example. 

### Performance
We evaluate SIEVE to answer the following questions:

- Does SIEVE have higher efficiency than state-of-the-art cache eviction algorithms?
- Is SIEVE simpler than other algorithms?
- Can SIEVE improve a cache's throughput and scalability?

In this blog post, we only show part of our evaluation results. If you're interested in the full evaluation, please refer to [our paper](https://yazhuozhang.com/assets/publication/nsdi24-sieve.pdf).

#### Efficiency
Our evaluation, involving over 1559 traces from diverse datasets containing 247,017 million requests to 14,852 million objects, shows that SIEVE outperforms all state-of-the-art eviction algorithms on more than 45% of the traces.

The following figure shows the miss ratio reduction (from FIFO) of different algorithms across traces. The whiskers on the boxplots are defined using p10 and p90, allowing us to disregard extreme data and concentrate on the typical cases.
SIEVE demonstrates the most significant reductions across
nearly all percentiles. 
For example, SIEVE reduces FIFO's miss ratio by more than 42% on 10% of the traces (top whisker), with a mean of 21% on one of the largest CDN datasets. 
As a comparison, all other algorithms have smaller reductions on this dataset.
Compared to advanced algorithms, e.g., ARC, SIEVE reduces
ARC miss ratio by up to 63.2% with a mean of 1.5%.

![](/blog_images/sieve/efficiency-large.png)

#### Simplicity
SIEVE is very simple. We delved into the most popular cache libraries and systems across five diverse programming languages: C++, Go, JavaScript, Python, and Rust. 

Despite the varied ways LRU is implemented across these libraries - some opt for doubly-linked lists, others for arrays - integrating SIEVE was easy. As illustrated in the Table, the required code changes to replace LRU with SIEVE were minimal. In all cases, it took no more than 21 lines of code modifications (tests not included).

| Cache library | Language   | Lines   | Hour of Work   |
| :---------: | :---------: |:---------: | :---------: |
| [groupcache](https://github.com/cacheMon/groupcache) | Golang  | 21  | <1  |
| [mnemonist](https://github.com/cacheMon/mnemonist) | Javascript |12  |  1  |
| [lru-rs](https://github.com/cacheMon/lru-rs) | Rust | 16  |  1  |
| [lru-dict](https://github.com/cacheMon/lru-dict)| Python + C | 21  | <1  |

<!-- <table>
  <tr>
    <th style="border: 1px solid black;">Cache library</th>
    <th style="border: 1px solid black;">Language</th>
    <th style="border: 1px solid black;">Lines</th>
    <th style="border: 1px solid black;">Hour of Work</th>
  </tr>
  <tr>
    <td style="border: 1px solid black;"><a href="https://github.com/cacheMon/groupcache">groupcache</a></td>
    <td style="border: 1px solid black;">Golang</td>
    <td style="border: 1px solid black;">21</td>
    <td style="border: 1px solid black;">&lt;1</td>
  </tr>
  <tr>
    <td style="border: 1px solid black;"><a href="https://github.com/cacheMon/mnemonist">mnemonist</a></td>
    <td style="border: 1px solid black;">Javascript</td>
    <td style="border: 1px solid black;">12</td>
    <td style="border: 1px solid black;">1</td>
  </tr>
  <tr>
    <td style="border: 1px solid black;"><a href="https://github.com/cacheMon/lru-rs">lru-rs</a></td>
    <td style="border: 1px solid black;">Rust</td>
    <td style="border: 1px solid black;">16</td>
    <td style="border: 1px solid black;">1</td>
  </tr>
  <tr>
    <td style="border: 1px solid black;"><a href="https://github.com/cacheMon/lru-dict">lru-dict</a></td>
    <td style="border: 1px solid black;">Python + C</td>
    <td style="border: 1px solid black;">21</td>
    <td style="border: 1px solid black;">&lt;1</td>
  </tr>
</table> -->
<br>

#### Throughput
Besides efficiency, throughput is the other important metric for caching systems. Although we have implemented SIEVE in five different libraries, we focus on Cachelib's results.

Compared to these LRU-based algorithms, SIEVE does not require "promotion" at each cache hit. Therefore, it is faster and more scalable.
At a single thread, SIEVE is 16% (17%) faster than the optimized LRU (TwoQ) and on the tested traces.
At 16 threads, SIEVE shows more than 2× higher throughput than the optimized LRU and TwoQ.

### SIEVE is beyond an eviction algorithm

Beyond being a cache eviction algorithm, SIEVE can serve as a cache primitive for designing more advanced eviction policies.
As a cache primitive, SIEVE can facilitate the design of more advanced eviction algorithms.

We've plugged SIEVE into [LeCaR](https://www.usenix.org/conference/hotstorage18/presentation/vietri), [TwoQ](https://www.vldb.org/conf/1994/P439.PDF), [ARC](https://www.usenix.org/conference/fast-03/arc-self-tuning-low-overhead-replacement-cache), and [S3-FIFO](https://dl.acm.org/doi/10.1145/3600006.3613147), swapping out their LRU or FIFO queue for a SIEVE one.

Compared to SIEVE, LeCaR has much lower efficiency; however, when replacing the LRU in LeCaR with SIEVE, it significantly reduces LeCaR's miss ratio by 4.5% on average.
TwoQ and ARC achieve efficiency close to SIEVE; however, when replacing the LRU with SIEVE, the efficiency of both algorithms gets boosted.
These results highlight the potential of SIEVE as a powerful cache primitive for designing advanced cache eviction algorithms.

![](/blog_images/sieve/sieve_queue_all_large.svg)


### SIEVE is not scan-resistant
Besides web cache workloads, we evaluated SIEVE on
some old block cache workloads. However, SIEVE
sometimes shows a miss ratio higher than LRU. The primary
reason for this discrepancy is that SIEVE is not scan-resistant.
In block cache workloads, which frequently feature scans,
popular objects often intermingle with objects from scans.
Consequently, both objects are rapidly evicted after insertion.
Note that we made a mistake and used "scan-resistant" to mean something else. For the traditional sense of scans (larger than cache size), SIEVE is scan-resistant.  


Since SIEVE does not use a ghost cache — a shadow cache that keeps track of recently evicted items to
make smarter future eviction decisions — it cannot recognize
the popular objects when they are requested again. This problem is less severe on the large cache size, but when the cache
size is small, we observe that having a ghost is critical to be
scan-resistant.

[Marc's latest blog post](https://brooker.co.za/blog/2023/12/15/sieve.html) explored making sieve scan-resistant by adding a small counter for each item. 
It shows some wins and losses on different workloads. 


## We'd Love to Hear from you
As we wrap up this blog post, we would like to give a big shoutout to the people and organizations that open-sourced and shared the traces. SIEVE presents an intriguing opportunity to explore and enhance the efficiency of web caching.
**If you have questions or thoughts or have given SIEVE a try, we're eager to hear from you! Don't hesitate to get in touch :-)**

## Appendix

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.visited = False
        self.prev = None
        self.next = None

class SieveCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # To store cache items as {value: node}
        self.head = None
        self.tail = None
        self.hand = None
        self.size = 0

    def _add_to_head(self, node):
        node.next = self.head
        node.prev = None
        if self.head:
            self.head.prev = node
        self.head = node
        if self.tail is None:
            self.tail = node

    def _remove_node(self, node):
        if node.prev:
            node.prev.next = node.next
        else:
            self.head = node.next
        if node.next:
            node.next.prev = node.prev
        else:
            self.tail = node.prev

    def _evict(self):
        obj = self.hand if self.hand else self.tail
        while obj and obj.visited:
            obj.visited = False
            obj = obj.prev if obj.prev else self.tail
        self.hand = obj.prev if obj.prev else None
        del self.cache[obj.value]
        self._remove_node(obj)
        self.size -= 1

    def access(self, x):
        if x in self.cache:  # Cache Hit
            node = self.cache[x]
            node.visited = True
        else:  # Cache Miss
            if self.size == self.capacity:  # Cache Full
                self._evict()  # Eviction
            new_node = Node(x)
            self._add_to_head(new_node)
            self.cache[x] = new_node
            self.size += 1
            new_node.visited = False  # Insertion

    def show_cache(self):
        current = self.head
        while current:
            print(f'{current.value} (Visited: {current.visited})', end=' -> ' if current.next else '\n')
            current = current.next

# Example usage
cache = SieveCache(3)
cache.access('A')
cache.access('B')
cache.access('C')
cache.access('D')
cache.show_cache()
```
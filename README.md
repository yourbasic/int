# Your basic int

### A tribute to an underrated data type

> you're the shit, you're down as hell,  
> and everything about you is great  
> BUT people don't think too much of you  
> because they don't know you.
> 
> you're so under rated, it sucks.

**Underrated** *defined by [Urban Dictionary][urban].*

# Table of contents

[Introduction](#introduction)

* [Resources](#resources)

[Generic graph data](#generic-graph-data)

* [Breadth-first search](#breadth-first-search)

[Effective searching and sorting](#effective-searching-and-sorting)

* [Needles in huge haystacks](#needles-in-huge-haystacks)

* [Bit count](#bit-count)

* [Fast integer sorting](#fast-integer-sorting)

[Simple sets](#simple-sets)

* [Sieve of Eratosthenes](#sieve-of-eratosthenes)

[Efficient filtering](#efficient-filtering)

* [A blacklist of shady websites](#a-blacklist-of-shady-websites)

* [Bloom filter implementation](#bloom-filter-implementation)


# Introduction

![DOUBLE-STRUCK CAPITAL Z](res/z.png)

**The Zahlen symbol** *used to denote the set of all integers.*

Every kid knows what an [integer number][integer] is,
and every programmer is familiar with the [int data type][int].  
Still we frequently forget how powerful an integer can be.

- **Generic**  
  An `int` or `int[]` is a bit pattern that can represent any digital data.  
  Furthermore, an `int` can point into any type of array.  
  That's as generic as it gets.

- **Effective**  
  With an `int` you have all of basic mathematics at your finger tips,  
  and boolean algebra, implemented with bit-level parallelism, to boot.

- **Simple**  
  Not really, but we've used arithmetic since childhood so it feels that way.  
  Familiarity breeds both simplicity and contempt.

- **Efficient**  
  An `int` fits inside a register sitting on the main datapath of the CPU,  
  and an `int[]` is the main focus of hardware memory optimization.  
  It doesn't get much faster or more efficient than that.

### Resources

The text comes with three [Go][golang] example libraries:

- [github.com/yourbasic/bit][bit]
  contains a bit array and some bit-twiddling functions,
- [github.com/yourbasic/bloom][bloom]
  is a Bloom filter, a probabilistic set data structure, and
- [github.com/yourbasic/graph][graph]
  is a library of basic graph algorithms.


# Generic graph data

![Graph](res/graph.png)

**Cubical graph with vertex colors**, *image from [Wikipedia][wikicube].*

Since graphs are used to model countless types of relations and processes
in varied kinds of systems and settings, there is no telling what kind of data
a generic graph library will encounter; both vertices and edges can have
labels attached to them. Should we use parametric polymorphism or perhaps a pointer
to the top of a type hierarchy?

It's easy to forget that an integer may be the preferred choice.
Here is a solution from the [graph][graph] package:

    All algorithms operate on directed graphs with a fixed number of vertices, 
    labeled from 0 to n-1, and edges with integer cost.

Since vertices are represented by integers, it's easy to add any kind of
vertex data on the side.


### Breadth-first search

For example, this generic implementation of breadth-first search uses
an array of booleans to keep track of which vertices have been visited.

    // BFS traverses g in breadth-first order starting at v.
    // When the algorithm follows an edge (v, w) and finds a previously
    // unvisited vertex w, it calls do(v, w, c) with c equal to
    // the cost of the edge (v, w).
    func BFS(g Iterator, v int, do func(v, w int, c int64)) {
        visited := make([]bool, g.Order())
        visited[v] = true
        for queue := []int{v}; len(queue) > 0; {
            v := queue[0]
            queue = queue[1:]
            g.Visit(v, func(w int, c int64) (skip bool) {
                if visited[w] {
                    return
                }
                do(v, w, c)
                visited[w] = true
                queue = append(queue, w)
                return
            })
        }
    }

*Source code from [bfs.go][graphbfs].*


# Effective searching and sorting

Some of the most effective search and sort algorithms are implemented
by bit manipulation done with bitwise integer operators.
These operators operate, often in parallel, on the single bits of an integer.
Even though they don't form a Turing-complete set of operations,
they can still be surprisingly effective.

The standard set of bitwise operators, found in almost every CPU,
includes the bitwise `not`, `and`, `or` and `xor` instructions; plus
a collection of `shift` and `rotate` instructions. A bit count instruction,
often known as `popcnt`, is also quite common.


### Needles in huge haystacks


The Hamming distance between two integers, the number of positions
at which the corresponding bits are different, is an effective way
to estimate similarity; it can be computed using just one `xor`
and one `popcnt` instruction.

![Hamming distance](res/hamming.png)

**The Hamming distance** *between adjacent vertices
in a cubical graph is one, image from [Wikipedia][wikihamming].*

Rumor has it that organizations who are sifting through huge amounts of data
prefer to buy CPUs that come with a bit count instruction.
As a case in point, the new Zen microarchitecture from AMD supports
no less than four `popcnt` instructions per clock cycle.


### Bit count

If the CPU doesn't have a native bit count operation, `popcnt` can still
be implemented quite efficiently using the more common bitwise operators.
Here is a fun code sample from the [bit][bit] package:

    // Count returns the number of nonzero bits in w.
    func Count(w uint64) int {
        // “Software Optimization Guide for AMD64 Processors”, Section 8.6.
        const maxw = 1<<64 - 1
        const bpw = 64
    
        // Compute the count for each 2-bit group.
        // Example using 16-bit word w = 00,01,10,11,00,01,10,11
        // w - (w>>1) & 01,01,01,01,01,01,01,01 = 00,01,01,10,00,01,01,10
        w -= (w >> 1) & (maxw / 3)

        // Add the count of adjacent 2-bit groups and store in 4-bit groups:
        // w & 0011,0011,0011,0011 + w>>2 & 0011,0011,0011,0011 = 0001,0011,0001,0011
        w = w&(maxw/15*3) + (w>>2)&(maxw/15*3)
    
        // Add the count of adjacent 4-bit groups and store in 8-bit groups:
        // (w + w>>4) & 00001111,00001111 = 00000100,00000100
        w += w >> 4
        w &= maxw / 255 * 15
    
        // Add all 8-bit counts with a multiplication and a shift:
        // (w * 00000001,00000001) >> 8 = 00001000
        w *= maxw / 255
        w >>= (bpw/8 - 1) * 8
        return int(w)
    }

*Source code from [funcs.go][bitfunc].*


### Fast integer sorting

Radix sort uses bit manipulation to good effect and bitwise operators are
crucial in the implementation of the fastest known integer sorting algorithm.
This algorithm sorts *n* integers in O(*n* log log *n*) worst-case time
on a unit-cost RAM machine, the standard computational model in theoretical
computer science. [The fastest sorting algorithm?][sort] has all the details.


# Simple sets

A **bit set**, or bit array, must be the simplest data structure in town.
It consists of an array of integers, where the bit at position *k*
is one whenever *k* belongs to the set.

Even though it's simple, a bit set can be quite powerful. Because it uses
bit-level parallelism, limits memory access, and plays nicely with
the data cache, it tends to be very efficient and often outperforms other
set data structures.

Memory consumption isn't too shabby either. If you have a gigabyte
of RAM to spare, you can story a set of integer elements in the range
0 to 8,589,934,591.

### Sieve of Eratosthenes

![Eratosthenes](res/eratosthenes.png)

**Eratosthenes of Cyrene**, *276-194 BC, image from [Wikipedia][eratosthenes].*

This code snippet implements the sieve of Eratosthenes.
It uses a bit set implementation from the [bit][bit] package
to generate the set of all primes less than *n* in O(*n* log log *n*) time.
Try it with *n* equal to a few hundred millions and be pleasantly surprised.

    sieve := bit.New().AddRange(2, n)
    sqrtN := int(math.Sqrt(n))
    for p := 2; p <= sqrtN; p = sieve.Next(p) {
        for k := p * p; k < n; k += p {
            sieve.Delete(k)
        }
    }

*Example from [godoc.org/github.com/yourbasic/bit][bitdoc].*


# Efficient filtering

Hash functions are yet another triumph for the integer data type.
They are worth a tribute of their own, but in this section we will
just take them for granted. If we combine a bit array with
a set of hash functions we get a *Bloom filter*, a probabilistic
data structure used to test set membership.

A membership test returns either ”likely member” or ”definitely not a member”.
Only false positives can occur: an element that has been added to the filter
will always be identified as ”likely member”.

Bloom filters are both fast and space-efficient. Akamai uses them
to avoid caching one-hit wonders, files that are seen only once.
Google uses them in Chrome to check for potentially harmful URLs.


### A blacklist of shady websites

This piece of code shows a typical Bloom filter use case.

    // Create a Bloom filter with room for 10000 elements
    // at a false-positives rate less than 0.5 percent.
    blacklist := bloom.New(10000, 200)
    
    // Add an element to the filter.
    url := "https://rascal.com"
    blacklist.Add(url)
    
    // Test for membership.
    if blacklist.Test(url) {
        fmt.Println(url, "seems to be shady.")
    } else {
        fmt.Println(url, "has not yet been added to our blacklist.")
    }

*Example from [godoc.org/github.com/yourbasic/bit][bloomdoc].*


### Bloom filter implementation

The implementation of a Bloom filter is straightforward.
An empty Bloom filter is a bit array of *m* bits, all set to 0.
There are also *k* different hash functions, each of which maps
a set element to one of the *m* bit positions.

- To add an element, feed it to the hash functions to get *k* bit positions,  
  and set the bits at these positions to 1.

- To test if an element is in the set, feed it to
  the hash functions to get *k* bit positions.  
  If any of the bits at these positions is 0,
  the element is definitely not in the set.  
  If all are 1, then the element
  is probably in the set.

This example from [Wikipedia][wikibloom] depicts a Bloom filter
that consists of 18 bits and uses 3 hash functions.

![Bloom filter](res/bloom.png)

The colored arrows point to the bits that the elements
of the set {*x*, *y*, *z*} are mapped to. The element *w* is not in the set,
because it hashes to a bit position containing 0.


#### Stefan Nilsson — [korthaj][korthaj]

*This work is licensed under a [Creative Commons Attribution 3.0 Unported License][CCBY3].*

[bit]: https://github.com/yourbasic/bit
[bitdoc]: https://godoc.org/github.com/yourbasic/bit
[bitfunc]: https://github.com/yourbasic/bit/blob/master/funcs.go
[bloom]: https://github.com/yourbasic/bloom
[bloomdoc]: https://godoc.org/github.com/yourbasic/bloom
[CCBY3]: https://creativecommons.org/licenses/by/3.0/deed.en
[eratosthenes]: https://en.wikipedia.org/wiki/File:Eratosthene.01.png
[golang]: https://golang.org
[graph]: https://github.com/yourbasic/graph
[graphbfs]: https://github.com/yourbasic/graph/blob/master/bfs.go
[int]: https://en.wikipedia.org/wiki/Integer_(computer_science)
[integer]: https://en.wikipedia.org/wiki/Integer
[korthaj]: https://github.com/korthaj
[sort]: https://www.nada.kth.se/~snilsson/fast-sorting/
[urban]: http://www.urbandictionary.com/define.php?term=underrated
[wikibloom]: https://en.wikipedia.org/wiki/File:Bloom_filter.svg
[wikicube]: https://commons.wikimedia.org/wiki/File:Cube_diagram;_octal_numbers.svg
[wikihamming]: https://en.wikipedia.org/wiki/File:Hamming_distance_3_bit_binary.svg
[wikiint]: (https://commons.wikimedia.org/wiki/File:Integers-line.svg)


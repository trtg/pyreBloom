__Py__thon + __Re__dis + __Bloom__ Filter = pyreBloom
=====================================================

One of Salvatore's suggestions for Redis' GETBIT and SETBIT commands is to
implement bloom filters. There was an existing python project that we used
for inspiration.

Notice
======
__Important__ -- The most recent version uses different seed values from all
previous releases. Previous releases were using `srand` and `rand`, though they
are not guaranteed to yield the same values on different systems. For example,
two clients compiled on different platforms with different C implementations
may not necessarily agree on what's in the filters. This latest version fixes
this, but will also be incompatible with filters constructed with any previous
versions.

Installation
============

You will need `hiredis` installed, as well as Cython (for the time being --
we'd like to remove this dependency soon), and a C compiler (probably GCC).
With those things installed, it's pretty simple:

	sudo python setup.py install

Usage
=====

There are serial and batch forms for both `add` and `contains`. The batch 
modes are about 4-5 times faster than their serial equivalents, so use them
when you can. When you instantiate a pyreBloom, you should give it a redis
key name, a capacity, and an error rate:

	import pyreBloom
	p = pyreBloom.pyreBloom('myBloomFilter', 100000, 0.01)
	# You can find out how many bits this will theoretically consume
	p.bytes
	# And how many hashes are needed to satisfy the false positive rate
	p.hashes

From that point, you can add elements quite easily:

    tests = ['hello', 'how', 'are', 'you', 'today']
    p.extend(tests)

The batch mode of `contains` differs from the serial version in that it actually
returns which elements are in the bloom filter:

	p.contains('hello')
	# True
	p.contains(['hello', 'whats', 'new', 'with', 'you'])
	# ['hello', 'you']
	'hello' in p
	# True

The Story
=========

We needed to keep track of sets of urls that we had seen when crawling web
pages, and had previously been keeping track of them in redis sets. Redis 
sets are, after all, extremely fast. As you can see in the benchmarks, set
insertions can handle about 500k 20-character insertions per second. _That_
is performant.

However, these sets of urls got to be prohibitively large. But, since we 
didn't really need to know _which_ urls we had seen but merely whether or
not we had seen a given url, we started inserting hashes of urls into redis
sets. Unfortunately, even these got to be prohibitively large. We tried a
lot of things, including limiting the number of discovered urls, but we 
also thought about using bloom filters.

There was an existing library to use redis strings as bloom filters, but it
wasn't inserting elements fast enough for our liking. By implementing our
hash functions in pure C we were able to double our performance. Using the 
C bindings for redis (hiredis), we were able to squeeze another 5x performance
boost, for a total of about 10x over the original implementation.

Rough Bench
===========

Here are numbers from the benchmark script run on a 2011-ish MacBook Pro
and Redis 2.4.0, inserting 10k 20-character psuedo-random words:

	Generating 20000 random test words
	Generated random test words in 0.365890s
	Filter using 4 hash functions and 95850 bits
	Batch insert : 0.209492s (47734.526951 words / second)
	Serial insert: 0.770047s (12986.217154 words / second)
	Batch test   : 0.170484s (58656.590137 words / second)
	Serial test  : 0.728285s (13730.886920 words / second)
	False positive rate: 0.012300 (0.100000 expected)
	Redis set add  : 0.023647s (422885.373502 words / second)
	Redis pipe chk : 0.244068s (40972.163611 words / second)
	Redis pipe sadd: 0.241150s (41467.941791 words / second)
	Redis pipe chk : 0.240877s (41514.979051 words / second)

While set insertions are __much__ faster than our bloom filter insertions
(this is mostly do to the fact that there's not a 'SETMBIT' command), the
pipelined versions of 'sadd' and checking for membership in the set are
actually a little slower than the bloom filter implementation. Win some, 
lose some.


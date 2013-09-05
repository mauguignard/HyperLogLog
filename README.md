The HyperLogLog algorithm [1] is a space efficient method to estimate the
cardinality of extraordinarily large data sets. This module provides an
implementation, written in C using a Murmur3 hash, for python 2.7.x or python 3.x. 

v0.72

## Setup ##

You will need the python development package. On Ubuntu/Mint
you can install this package using:

    sudo apt-get install python-dev

Now install using setup.py:

    sudo python setup.py install

## Quick start ##

    from HLL import HyperLogLog
    
    hll = HyperLogLog(5) # use 2^5 registers
    hll.add('some data')
    estimate = hll.cardinality()
  
## Documentation ##

##### HyperLogLog(<i>k [,seed]) #####

Create a new HyperLogLog using 2^<i>k</i> registers, <i>k</i> must be in the 
range [2, 16]. Set <i>seed</i> to determine the seed value for the Murmur3 
hash. The default value is 314.

* * *

##### add(<i>data</i>)

Adds <i>data</i> to the estimator where <i>data</i> is a string, buffer, or 
memoryview.

##### merge(<i>HyperLogLog</i>)

Merges another HyperLogLog object with the current object. Merging compares the 
registers of each object, setting the register of the current object to the 
maximum value. Only the registers of the current object are affected, the 
registers of the merging object are unaffected.

##### murmur3_hash(<i>data [,seed]</i>)

Gets a signed integer from a Murmur3 hash of <i>data</i> where <i>data</i> is a 
string, buffer, or memoryview. Set <i>seed</i> to determine the seed
value for the Murmur3 hash. The default seed is HyperLogLog's default seed.

##### registers()

Gets a bytearray of the registers.

##### seed()

Gets the seed value used in the Murmur3 hash.

##### set_register(<i>index, value</i>)

Sets the register at <i>index</i> to <i>value</i>. Indexing is zero-based.

##### size()

Gets the number of registers.

## Theory <a name='theory'></a>

This section is intended to provide a description of the HyperLogLog algorithm
and some intuition as to why it works. 

Suppose we have some multi-set (a set containing duplicate elements) occupying 
several TB or PB of disk space and we wish to know the cardinality of this set. 
Using a naive approach, we choose a hash function that randomly and independently
distributes the bits and then hash each element of the multi-set. Duplicate 
elements hash to the same value so the cardinality can then be estimated by
counting the number of hashes. If the cardinality is 100 billion and each hash 
occupies 64 bits then the space required to store the hashes is 64 bits * 10^11 
= 800 GB. This space requirement is prohibitively large for many practical 
applications.

A more sophisticated strategy might utilize <i>linear counting</i>. Similar to 
the  naive approach, linear counting hashes each element in the multi-set. However 
instead of storing the entire hash, each hash instead determines the index of a 
bit in an array of zero bits. This bit is set to 1 and the hash discarded. The 
cardinality can then be estimated by counting the number of non-zero bits in the 
array. The space requirement is size of the bit array. Using the previous example, 
an array with 100 billion elements requires 10^11 bits = 12.5 GB to accurately 
estimate the cardinality. While this is a more reasonable space requirement, 
it is still unsuitable for  extraordinarily large cardinalities e.g. if the 
cardinality is 1 trillion then the space required to store the bits is 1 
trillion bits = 125 GB.

HLL relies on making observations in the underlying bit-patterns of the 
hashed elements. As an explanatory example consider an 8-bit case. 
Suppose h(x) is a hash function that randomly distributes the bits of x 
independelty and with equal probability. Then a hashed element might have the 
following distribution of bits:

|  0  | 0  | 0  | 0  | 1  | 1  | 0  | 1  |
| --- |:--:|:--:|:--:|:--:|:--:|:--:| --:|

Note the position of the first 1 bit, from left to right. This is known as the
rank. In the example above, the first 1 bit is in the fifth position, indexed 
from the left, so the rank is 5. Out of all the numbers that can be formed on 
8 bits, what is the probability of a number having rank 5? If we interpret the 
sequence of bits as a sequence of coin flips where 0 is a tails, 1 is a heads 
and k is the rank (the number of independent flips required to observe a heads) 
then there are k-1 tails with probability p and 1 heads with probability 1-p
so the rank is geometrically distributed according to:

    P(X=k) = p^(k-1) * (1 - p)

Each bit has equal probability of being a 1 or zero due to the properties
of our hash function so p=1/2:

    P(X=k) = (1/2)^(k-1) * (1/2)
    
    P(X=k) = 1/2^k
	
Then for the case of rank 5: 

    P(X=5) = 1/2^5

Note P(X=5) is equivalent to:

    P(X=5) = (# of rank 5 numbers) / (# of possible numbers on 8 bits)
    
It follows that:

    1/2^5 = (# of rank 5 numbers) / (2^8)

    (# rank 5 numbers) = 2^8 / 2^5  = 2^3 = 8

So we might expect 8 elements to have rank 5 and indeed this is the case. All of
the rank 5 numbers are given below:

    00001000
    00001001
    00001010
    00001011
    00001100
    00001101
    00001110
    00001111

More generally, if there are n distinct elements and r is the number of elements 
with rank k, then we would expect about n/2^k of the hashed elements to have 
rank k. Moreover,

    P(X=k) ~ r / n	
	
    n ~  r / P(X=k)
	
    n ~  r * 2^k
	
    log_2(n) ~ k + log2(r)            
	
In other words, if M contains the hashed elements of a multi-set of unknown 
cardinality, n is the true cardinality of M, and R is the maximum rank amongst the 
elements of M, then R provides a rough estimation of log_2(n) with some additive bias. 

Using only a single observable introduces inaccuracy into the results. For 
example, suppose all the elements of M have the same hash. This implies the elements are 
not distinct e.g. the cardinality is 1. However the rank of the hash may be large so 
when the estimate for log_2(n) is computed the estimate will be large and inaccurate.

As an improvement HLL uses multiple observables. HLL divides M into m buckets and takes the maximum rank of each 
bucket. Then for each bucket, we have an estimate of log_2(n/m). These results 
are averaged using a harmonic mean and then multiplied by a bias-reducing constant [1]. 

The HLL algorithm is given by the following pseudocode:

```
Let h: D --> [0, 1] = {0, 1}^32; // hash data from domain D to 32-bit words
Let p(s) be the position of the leftmost 1-bit of s; // e.g. p(001...) = 3, p(0^k) = k + 1
Define a_16 = .673, a_32 = .697, a_64 = .709; a_m = .7213/(1 + 1.079/m) for m >= 128;

Algorithm HYPERLOGLOG(input M: a multiset of items from domain D)
    assume m = 2^b with b in [4 .. 16];
    initialize a collection of m registers, M[1], ..., M[m], to 0;
	
	for v in M do
	    set x := h(v);
		set j := 1 + <x_1 x_2 ... x_b}_2; // the binary address determined by the first b bits of x
		set w := <x_b+1 x_b+2 ... >; // the remaining bits of x
		set M[j] := max(M[j], p[w]);
		
	Z := 0;
	for j := 1 to m do
	     Z := Z + 2^-M[j];
	
	Z := 1/Z;
	E := a_m * m^2 * Z; // raw HLL estimate
	
	if E <= 5/2 * m then
		let V be the number of registers equal to 0;
		if V != 0 then set E* := m*log(m/V) else set E* := E // small range correction
			
	if E < 1/30 * 2^32 then
		set E* := E // intermediate range - no correction
	
	if E > 1/30 * 2^32 then
		set E* := -2^32 * log(1 - E/2^32) // large range correction
	
	return cardinality estimate E* with typical relative error +/- 1.04/m^(1/2)
```

## License

This software is released under the [MIT License](https://gist.github.com/ascv/5123769).

## References

[1] http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf

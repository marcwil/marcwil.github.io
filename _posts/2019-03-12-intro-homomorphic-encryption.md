---
layout: post
comments: true
title:  "FHE1: Introduction to practical homomorphic encryption"
date:   2019-03-15 14:27:04 +0100
categories: jekyll update
---

# Background

It is not hard to be fascinated when first hearing about homomorphic
encryption.

Homomorphic encryption schemes are asymmetric cryptosystems that support the
evaluation of functions on encrypted data.
For example with RSA, ciphertexts can be easily be multiplied. The result is a
ciphertext containing the (encrypted) result of the multiplication:

    Enc(a) * Enc(b) = Enc(a*b)

This is already cool, but actually there is something even cooler: _homomorphic
evaluation of arbitrary (computable) functions_!

The first cryptosystem supporting this has been proposed by Craig Gentry in 2009
in his seminal thesis.
That cryptosystem operates on bits and can evaluate arbitrary computable
functions in polynomial time.
This was the first _fully homomorphic cryptosystem_.

This of course enables a myriad of possible applications from private cloud
computing to protecting the computations on untrusted devices.  For example a
team at Microsoft developed a POC of a neural network (called
[CryptoNets](https://www.microsoft.com/en-us/research/publication/cryptonets-applying-neural-networks-to-encrypted-data-with-high-throughput-and-accuracy/))
that processes encrypted data. For more exciting results like private database queries or protected genome analysis check out the
[publication
list](https://www.microsoft.com/en-us/research/project/homomorphic-encryption/#!publications)
of the Homomorphic Encryption group at Microsoft.

But let's clarify the terminology a bit more:
* Fully homomorphic encryption (FHE) allows to efficiently perform arbitrary
  computations on encrypted data without needing a secret key
* Leveled Fully Homomorphic Schemes are encryption schemes that depending on
  some parameters can evaluate arithmetic circuits of depth `L` for some `L ∈ ℕ`.
  The runtime of the encryption and decryption algorithms does not depend on `L`.
  The evaluation algorithm may have polynomial runtime in `L`.
* Somewhat Homomorphic Schemes can evaluate circuits of constant size and are
  more interesting from a theoretical standpoint, because they are often used in
  order to construct (Leveled) Fully homomorphic encryption schemes.

Since 2009 many researchers including Gentry himself have been working on
various improvements, improving performance to a somewhat practical level.
Here is a (probably incomplete) list of important research papers:
* [Fully Homomorphic Encryption without
  Bootstrapping](https://eprint.iacr.org/2011/277) (commonly referred to as BGV)

  This research paper proposed the first leveled fully homomorphic cryptosystem
  without bootstrapping and also included a batching mechanism that allows to
  pack multiple integers into a single ciphertext.

  This is one of the cryptosystems implemented in
  [HElib](https://github.com/shaih/HElib).
* [Somewhat Practical Fully Homomorphic
  Encryption](https://eprint.iacr.org/2012/144) (commonly referred to as BFV)

  This paper introduced many practical optimizations and is (with further
  optimizations) implemented in [SEAL](https://www.microsoft.com/en-us/research/project/microsoft-seal/)
* [Homomorphic Encryption for Arithmetic of Approximate
  Numbers](https://eprint.iacr.org/2016/421) (commonly referred to as CKKS)

  A fairly recent paper that can encode arrays of approximate fractional (read: something like floats) numbers in a single ciphertext and is very efficient.

  This is scheme is implemented in both [HElib](https://github.com/shaih/HElib)
  and [SEAL](https://www.microsoft.com/en-us/research/project/microsoft-seal/).

# Practical usage

In the following I will show you how to actually use homomorphic encryption.
For that purpose I give some examples using the
[Pyfhel](https://github.com/ibarrond/Pyfhel) Python package. This library uses
(currently in version 2.0) the [SEAL library](http://sealcrypto.org/) by
Microsoft as a backend.

For basic usage we can do something like this:

{% highlight python %}
import Pyfhel
he = Pyfhel.Pyfhel()
he.contextGen(p=10000)
he.keyGen()
# now everything is initialized
half_the_truth = he.encryptInt(21)
two = he.encryptInt(2)
the_truth = he.multiply(half_the_truth, two)
print(he.decryptInt(the_truth))
#=> prints 42
{% endhighlight %}

Interestingly, we can also do this:

{% highlight python %}
pi = he.encryptFrac(3.141592)
two_frac = he.encryptFrac(2.0)
result = he.multiply(pi, two_frac)
print(he.decryptFrac(result))
#=> prints 6.283183999825269
{% endhighlight %}

Let me point out why this is so interesting:
* Apparently we can encrypt integers and floating point numbers and we have
  different functions to do that
* The number 2 had to be encrypted twice, once with `encryptInt` once with
  `encryptFrac`
* The result is not completely correct: 3.141592 x 2 = 6.283184; but instead of
  being completely wrong it is just slightly inaccurate (and contains many
  additional digits)

Why is this?

## Plaintext space

To answer this question let me explain a little bit more about how the SEAL
library works.  To that end let's discuss how plaintexts look like and what
parameters control the cryptosystem in which ways.

The plaintexts that are being manipulated by the SEAL library are actually
quite abstract. The plaintext space is a polynomial ring over a prime integer
ring. That means plaintexts are polynomials with integers modulo a prime number
as coefficients and the polynomials are themselves taken modulo a big
irreducible (something like prime) polynomial.

This already touches upon some of the parameters we can set:
* **Plaintext modulus (p):** This is the prime number for the coefficients.
* **Polynomial modulus (m):** This is the degree of the irreducible polynomial x^m+1.

Additionally there is the **Coefficient modulus** that is harder to select and hidden by the Pyfhel library. Instead of it, in Pyfhel there is a security parameter that can be set to 128, 192 and 256 bit.

## Noise budget & Encodings

To understand what affect the different parameters have, we have to introduce
the concept of noise budget and look at the different data encodings.

The following code snippet helps to visualize the noise budget of ciphertexts:
{% highlight python %}
ctxt1 = he.encryptInt(2)
print(he.decryptInt(ctxt1))
#=> prints 2
print(he.noiseLevel(ctxt1))
#=> prints 30
ctxt2 = he.add(ctxt1, two)
print(he.decryptInt(ctxt1))
#=> prints 4
print(he.noiseLevel(ctxt1))
#=> prints 29
ctxt3 = he.multiply(ctxt2, two)
print(he.decryptInt(ctxt2))
#=> prints 8
print(he.noiseLevel(ctxt2))
#=> prints 6
ctxt4 = he.multiply(ctxt3, two)
print(he.decryptInt(ctxt3))
#=> prints -7906467554648178793
print(he.noiseLevel(ctxt3))
#=> prints 0
{% endhighlight %}

Each ciphertext has an associated noise budget and with each operation the
noise budget shrinks.  The noise impact of additions is a lot smaller than the
one of multiplications.  Also depleting the noise budget is not a good idea: it
makes the ciphertexts indecipherable.

Note: the `he.noiseLevel` function can be used to check the noise budget, but
it needs access to the private key.

All parameters affect the noise budget of a freshly encrypted ciphertext:
* Higher plaintext modulus: slightly less noise budget and better performance
* Higher polynomial modulus: significantly more noise budget and worse
  performance
* Higher security parameter: less noise budget and better performance

This means that there is a trade-off between the number of homomorphic
multiplications (and less strongly additions) that can be performed and the
performance.

Now let's talk about the data encodings. Despite the fact that under the hood,
SEAL operates on polynomials, it is possible to encrypt integers and fractional
numbers.  Additionally there is a 'batch encoding' that enables the encryption
of vectors of integers.

Here is an overview:

* **Integer encoding:** here the coefficients of the plaintext are used to
  store the summands of a b-adic representation of an integer. This means that
  a higher base b or a higher polynomial modulus allows bigger numbers to be
  stored. For more details have a look at the [seal manual for version
  2.3.0](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/12/sealmanual.pdf)
* **Fractional encoding:** this encoding is also based on the (rounded) b-adic
  representation of a number. Given a summands `x*b^1 + y*b^(-1) + z*b^(-2)`
  the lowest degree coefficient will be set to `x`, the highest degree
  coefficient to `y` and the second highest degree coefficient to `z`. This
  means that a higher polynomial modulus allows for larger numbers / more
  precision. Again, for more details check out the [seal
  manual](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/12/sealmanual.pdf).
* **Batch encoding:** this encoding uses fancy math to encode multiple integers
  in a single ciphertext. With a polynomial modulus `m`, the ciphertext can be
  viewed as a container for a 2 by `m`/2 matrix. The elements of this matrix
  are integers modulo the plaintext modulus. The additions and multiplications
  are translated to element wise additions and multiplications of the elements
  in the matrix. Additionally, there are rotation operations that rotate the
  rows or columns of the ciphertext. More details: [seal
  manual](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/12/sealmanual.pdf).

We already saw a code snippet for the integer and fractional encoding, so now
lets see how the batch encoding is used:

{% highlight python %}
he2 = Pyfhel.Pyfhel()
# to allow batch encoding, we need a polynomial modulus p
# so that the polynomial modulus divides p-1
he2.contextGen(p=40961, m=4096, flagBatching=True)
he2.keyGen()
# we need to generate rotation keys to enable rotation
# operations
he2.rotateKeyGen(20)

array1 = he2.encryptBatch([1,2,3,4])
array2 = he2.encryptBatch([2,2,2,2])

# the third argument (True) is used to store the result in
# a new ciphertext (otherwise it would be stored in array1)
array_mult = he2.multiply(array1, array2, True)
print(he2.decryptBatch(array_mult)[:10])
#=> prints [2, 4, 6, 8, 0, 0, 0, 0, 0, 0]

rotated_row = he2.rotate(array1, 1, True)
print(he2.decryptBatch(rotated_row)[:10])
#=> prints [2, 3, 4, 0, 0, 0, 0, 0, 0, 0]
print(he2.decryptBatch(rotated_row)[2040:2050])
#=> prints [0, 0, 0, 0, 0, 0, 0, 1, 0, 0]
{% endhighlight %}

## Relinearization

There is one important thing that is missing: relinearization.
For this it is important to know that ciphertexts have a size that is 2 for a
fresh ciphertext and grows with multiplications.
The problem is that multiplying ciphertexts of bigger size depletes the noise budget even faster.

To help this problem, there is a relinearization operation that shrinks ciphertexts back to size 2.

{% highlight python %}
print(array_mult.size())
#=> prints 3

mult2 = he2.multiply(array_mult, array_mult, True)
print(mult2.size())
#=> prints 5
print(he2.noiseLevel(mult2))
#=> prints 20

he2.relinKeyGen(10,3)
he2.relinearize(array_mult)
print(array_mult.size())
#=> prints 2

mult2_new = he2.multiply(array_mult, array_mult, True)
print(mult2_new.size())
#=> prints 3
print(he2.noiseLevel(mult2_new))
#=> prints 26
{% endhighlight %}

We see that the multiplying the relinearized `array_mult` results in a ciphertext with slightly higher remaining noise budget.

This already concludes this introductory post on practically applying homomorphic encryption.
We talked about what HE is, had a quick look at some of the important research
papers and discussed how to actually use it with a nice and simple Python package that wraps the SEAL library by Microsoft.

In upcoming posts we will have a closer look at how the batch encoding can be used for some optimized computations, e.g. for neural networks.
